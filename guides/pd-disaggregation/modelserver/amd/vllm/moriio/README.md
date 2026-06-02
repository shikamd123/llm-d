# AMD / vLLM / MoRI-IO P/D Disaggregation Overlay

Single-pod **1P1D** P/D disaggregation overlay for DeepSeek-V3 on AMD
Instinct GPUs, using the **MoRI-IO** KV-transfer connector and the
**MoRI-EP** Wide-EP all-to-all backend (DP=8, EP=8, TP=1).

This is the AMD-side companion to the existing NIXL overlay at
`../base/`. Use this overlay when you want RDMA-Write KV transfer
(MoRI-IO) instead of NIXL/UCX, and Wide-EP MoE expert parallelism
inside a single pod instead of cross-pod TP.

## Topology

| Role    | Replicas | TP | DP | EP        | KV connector                    |
| ------- | -------- | -- | -- | --------- | ------------------------------- |
| Prefill | 1        | 1  | 8  | enabled   | `MoRIIOConnector` (kv_producer) |
| Decode  | 1        | 1  | 8  | enabled   | `MoRIIOConnector` (kv_consumer) |

Each pod consumes 8 GPUs on a single node; vLLM runs 8 data-parallel
ranks locally with one OpenAI front-end per rank
(`--api-server-count=8`). The routing sidecar in front of the decode
pod hashes each request's UUID with blake2s and routes both the
prefill and decode leg to the same DP rank, so a single conversation
stays affinitised to one (P, D) rank pair.

A small ZMQ "registration proxy" Deployment + Service
(`moriio-zmq-proxy.yaml`) is included so the overlay is deployable
on any cluster that satisfies the prerequisites below — no extra
out-of-tree manifests required.

## How it differs from `../base/`

The sibling NIXL overlay (`pd-disaggregation/modelserver/amd/vllm/base/`)
runs Llama-3.3-70B-Instruct-FP8-KV with TP=4 decode / TP=1 prefill and
the upstream `NixlConnector` (kv_role=kv_both). This overlay adapts
the same base recipe to:

1. **Model**: `deepseek-ai/DeepSeek-V3` (~700 GiB MoE) instead of
   Llama-3.3-70B-Instruct-FP8-KV (~70 GiB dense).
2. **Connector**: `MoRIIOConnector` with split `kv_producer` /
   `kv_consumer` roles (NIXL uses `kv_both`).
3. **Parallelism**: TP=1, DP=8, EP enabled, with a **role-specific
   MoRI all-to-all backend** &mdash; prefill uses
   `--all2all-backend=mori_high_throughput` (backed by `InterNodeV1`,
   optimised for large-batch throughput-bound prefill), decode uses
   `--all2all-backend=mori_low_latency` (backed by `InterNodeV1LL`,
   optimised for small-batch latency-bound decode). Both roles also
   pass `--data-parallel-size{,-local}=8`, `--api-server-count=8`,
   `--enable-expert-parallel`. The legacy single-name `mori` backend
   was split upstream; do **not** use it here.
4. **Resources**: 8 GPUs/pod (vs. 1 prefill / 4 decode on the NIXL
   overlay), `hostNetwork: true`, RDMA NIC (`rdma/ib: 1`), and host
   mounts for `/dev/kfd`, `/dev/dri`, `/dev/infiniband`,
   `/sys/class/infiniband`, `/sys/class/net` &mdash; MoRI-IO needs
   direct GPU-direct-RDMA NIC access.
5. **Routing sidecar**: extra CLI flags (`--moriio-write-mode`,
   `--moriio-dp-size=8`, `--moriio-tp-size=1`,
   `--moriio-parallel-dispatch`, `--moriio-local-pod-ip=$(POD_IP)`)
   plus a `POD_IP` downward-API env var. The sidecar image must be a
   `llm-d-router` build that supports those flags (see "Required
   overrides" below).

Nothing about DP, TP, EP, or the model is hard-coded inside vLLM or
`llm-d-router`; everything visible above is either a CLI flag or an
env var, so this overlay can be cloned and re-tuned (e.g. DP=4 for
half-pod runs, or TP=2/DP=4 hybrids) without touching binaries.

## Required overrides

`kustomization.yaml` declares two image references as **unbound
placeholders**. `kustomize build` will fail until both are set.

| Placeholder                       | Set to                                                                         |
| --------------------------------- | ------------------------------------------------------------------------------ |
| `REPLACE_MODEL_SERVER_IMAGE`      | A vLLM ROCm image with MoRI-IO + MoRI-EP support (see `docker/Dockerfile.rocm` and the companion vLLM PR adding MoRI-IO + MoRI-EP). |
| `REPLACE_ROUTING_SIDECAR_IMAGE`   | A `llm-d-router` build that accepts the `--moriio-*` CLI flags (see the `llm-d-inference-scheduler` PR introducing them).            |

Override both at deploy time:

```bash
kustomize edit set image \
  REPLACE_MODEL_SERVER_IMAGE=<your-registry>/<vllm-rocm-moriio-image>:<tag> \
  REPLACE_ROUTING_SIDECAR_IMAGE=<your-registry>/<llm-d-router-image>:<tag>
```

…or pin them in a per-cluster overlay one directory deeper.

## Quickstart

```bash
# Pin the two images first (see "Required overrides" above), then:
kustomize build guides/pd-disaggregation/modelserver/amd/vllm/moriio \
  | kubectl apply -n llm-d -f -
```

Drive traffic at the decode pod's `Service` on port 8000 (the
routing-proxy initContainer); it forwards to vLLM on 8200 after the
MoRI-IO handshake is complete.

## Prerequisites on the cluster

- AMD Instinct GPU nodes with at least 8 GPUs per node and an RDMA
  NIC exposed via the `k8s-rdma-shared-dev-plugin` (resource name
  `rdma/ib`).
- AMD GPU operator providing `amd.com/gpu`.
- A working `Secret/llm-d-hf-token` (key `HF_TOKEN`) for the gated
  DeepSeek-V3 weights, mounted via the recipe base. Create it with:

  ```bash
  kubectl -n llm-d create secret generic llm-d-hf-token \
    --from-literal=HF_TOKEN="${HF_TOKEN:?set HF_TOKEN first}"
  ```

- Sufficient free space in the pod's writable layer for the
  DeepSeek-V3 weights, **or** an out-of-tree overlay that mounts
  pre-staged weights at `/models/DeepSeek-V3` and switches the
  `vllm serve` first-positional arg from `deepseek-ai/DeepSeek-V3`
  (HF Hub) to that path. `VLLM_ENGINE_READY_TIMEOUT_S` is set to a
  permissive value to cover first-boot HF download.
- A node label / nodeSelector strategy if you need to pin pods to
  specific worker nodes; this overlay does not pin by default. Add a
  small per-cluster overlay if you need that.

See `../../../../prerequisites/` for cluster-bring-up details that
apply to all P/D guides.

## Verifying the deployment

```bash
kubectl -n llm-d get pods
# Expect: pd-disaggregation-rocm-moriio-vllm-prefill-* Running
#         pd-disaggregation-rocm-moriio-vllm-decode-*  Running
#         moriio-zmq-proxy-*                          Running

kubectl -n llm-d logs deploy/moriio-zmq-proxy
# Expect a "register role=kv_producer ..." line per prefill DP rank
# and a "register role=kv_consumer ..." line per decode DP rank.

kubectl -n llm-d logs -l llm-d.ai/role=prefill --tail=50 \
  | grep -E 'all2all-backend|MoRIIOConnector|engine ready'
# Expect: --all2all-backend mori_high_throughput, kv_role=kv_producer

kubectl -n llm-d logs -l llm-d.ai/role=decode --tail=50 \
  | grep -E 'all2all-backend|MoRIIOConnector'
# Expect: --all2all-backend mori_low_latency, kv_role=kv_consumer
```

## Tear down

```bash
kustomize build guides/pd-disaggregation/modelserver/amd/vllm/moriio \
  | kubectl delete -n llm-d -f - --ignore-not-found
```
