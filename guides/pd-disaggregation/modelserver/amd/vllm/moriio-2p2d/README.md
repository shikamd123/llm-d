# 2P2D Wide-EP-16 overlay (DeepSeek-V3, MoRI-IO)

Overlay for **2 prefill + 2 decode** pods running DeepSeek-V3 with global
DP=EP=16, TP=1 on AMD MI300X. It is additive to the 1P1D DP=EP=8 base in
`../moriio`: it inherits that base, patches the master Deployments to the
DP=16 / dp_local=8 shape, and adds the prefill/decode child Deployments
for the second 8-rank pod on each side.

## Topology

| Role                       | Global DP ranks | OAI front-end                   |
|----------------------------|-----------------|---------------------------------|
| services + zmq-proxy + EPP | —               | —                               |
| prefill-master             | 0..7            | yes (8 procs)                   |
| prefill-child              | 8..15           | no (`--headless`)               |
| decode-master              | 0..7            | yes (8 procs) + routing-sidecar |
| decode-child               | 8..15           | no (`--headless`)               |

Each model-server pod has 8 GPUs, `hostNetwork: true` (so pod IP = node
mgmt IP), and mounts DSV3 weights + a per-role vLLM cache from node-local
host paths (`REPLACE_MODEL_HOST_PATH` / `REPLACE_CACHE_HOST_PATH`).

## What's new vs. 1P1D

1. **Cross-pod DP**: the master pods set `--data-parallel-size=16
   --data-parallel-size-local=8 --data-parallel-address=$(POD_IP)
   --data-parallel-rpc-port=...`; the child pods add
   `--data-parallel-start-rank=8 --headless`.
2. **Sidecar Wide-EP fan-out flags** on the decode sidecar:
   `--moriio-dp-size=16`, `--moriio-dp-size-local=8`,
   `--moriio-remote-hosts=<prefill master,child>`,
   `--moriio-decode-hosts=<decode master,child>`.
3. The vLLM MoRI-IO connector consumes `remote_hosts` /
   `remote_dp_size_local` from `kv_transfer_params` and resolves
   `pod_idx = global_dp_rank // dp_size_local` per handshake. When those
   keys are absent (the 1P1D path) it falls through to the single-host
   branch unchanged.

## Placeholders to replace before deploy

Replace every `REPLACE_*` token (in the YAML files) with your values:

| Placeholder                       | Value                                            |
|-----------------------------------|--------------------------------------------------|
| `REPLACE_MODEL_SERVER_IMAGE_NAME` / `_TAG` | MoRI-IO WRITE-mode model-server image   |
| `REPLACE_ROUTING_SIDECAR_IMAGE_NAME` / `_TAG` | Wide-EP multi-pod routing sidecar    |
| `REPLACE_SERVICES_NODE`           | hostname of the services / zmq-proxy node        |
| `REPLACE_PREFILL_MASTER_NODE` / `_CHILD_NODE` | prefill node hostnames               |
| `REPLACE_DECODE_MASTER_NODE` / `_CHILD_NODE`  | decode node hostnames                |
| `REPLACE_ZMQ_PROXY_IP`            | services-node IP serving the zmq-proxy (port 10001) |
| `REPLACE_PREFILL_MASTER_IP` / `_CHILD_IP` | prefill pod IPs (= node mgmt IPs)        |
| `REPLACE_DECODE_MASTER_IP` / `_CHILD_IP`  | decode pod IPs (= node mgmt IPs)         |
| `REPLACE_MODEL_HOST_PATH`         | node-local dir holding the DSV3 weights          |
| `REPLACE_CACHE_HOST_PATH`         | node-local base dir for per-role vLLM cache (suffixed `/prefill`, `/decode`, ...) |

The RDMA fabric env in `patch-nccl-env.yaml` (mlx5 device lists, GID
index, RoCE TC/SL) is also fabric-specific; adjust it for your topology.

## Deploy

```bash
# 1) Resolve node hostnames and InternalIPs:
kubectl get nodes -o wide

# 2) Replace the REPLACE_* placeholders in the YAML files (sed/envsubst).

# 3) Build and apply:
kubectl kustomize . | kubectl -n llm-d apply -f -

# 4) Wait for the 4 model-server pods + decode-master sidecar to be Ready:
kubectl -n llm-d get pods -w
```

## Coexistence with the 1P1D overlay

The two overlays share zero Kubernetes objects: different Deployment
names (2P2D adds `*-prefill-child` / `*-decode-child`), different images,
different node pinning, and different DP coordinator ports. The 1P1D base
(`../moriio`) is inherited unchanged.
