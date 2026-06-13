# 2P2D Wide-EP-16 MoRI-IO Guide (DORMANT)

> **Status: DORMANT - Not Yet Enabled**
>
> This guide requires the MoRI-IO WRITE-mode feature in the llm-d-router sidecar,
> which is currently **dormant** (disabled by default). Deploying with these
> configurations will fail until the feature is enabled in a future Release Candidate.

## Overview

This guide deploys **2 prefill + 2 decode** pods running DeepSeek-V3 with:
- Global DP=EP=16, TP=1
- AMD MI300X GPUs (8 per pod)
- MoRI-IO RDMA Write for KV cache transfer

## Why is this feature dormant?

The deployment guide has been merged to:
1. Allow early review and feedback
2. Enable incremental CI test development
3. Validate the configuration structure

However, the **sidecar feature flag** (`MoRIIOFeatureEnabled`) is set to `false`.
The feature will be enabled in a future Release Candidate (RC) after:
- [ ] Sidecar MoRI-IO WRITE-mode is validated
- [ ] End-to-end CI tests are added
- [ ] Production deployment validation is complete
- [ ] LeaderWorkerSet (LWS) deployment guides are finalized

## How to enable (FUTURE)

Once the sidecar feature is ready, the constant in
`llm-d-inference-scheduler/pkg/sidecar/proxy/options.go` will be changed:

```go
// Current (dormant):
MoRIIOFeatureEnabled = false

// Future (enabled):
MoRIIOFeatureEnabled = true
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     2P2D Wide-EP-16 Topology                    │
├─────────────────────────────────────────────────────────────────┤
│  PREFILL SIDE                      DECODE SIDE                  │
│  ┌───────────────┐                 ┌───────────────┐            │
│  │ prefill-master│ ──MoRI-IO──▶   │ decode-master │◀── sidecar │
│  │ ranks 0..7    │  RDMA Write     │ ranks 0..7    │            │
│  └───────┬───────┘                 └───────┬───────┘            │
│          │ DP coord                        │ DP coord           │
│  ┌───────▼───────┐                 ┌───────▼───────┐            │
│  │ prefill-child │ ──MoRI-IO──▶   │ decode-child  │            │
│  │ ranks 8..15   │  RDMA Write     │ ranks 8..15   │            │
│  └───────────────┘                 └───────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

## Files

| File | Purpose |
|------|---------|
| `kustomization.yaml` | Inherits from `../moriio`, applies all patches |
| `moriio-config.yaml` | **ConfigMap with all placeholders** (edit this!) |
| `children.yaml` | Child deployments for ranks 8..15 |
| `patch-*-master.yaml` | Widen masters from DP=8 → DP=16 |
| `patch-sidecar-2p2d.yaml` | Sidecar multi-pod fan-out |
| `patch-nccl-env.yaml` | RDMA fabric configuration |
| `patch-strategy-recreate.yaml` | Prevents rolling update wedge |
| `inductor-shim-configmap.yaml` | PyTorch inductor workaround |

## Configuration

All deployment-specific values are centralized in `moriio-config.yaml`.
Edit the `REPLACE_*` placeholders before deploying:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: moriio-2p2d-config
data:
  # Pod IPs
  ZMQ_PROXY_HOST: "REPLACE_ZMQ_PROXY_IP"
  PREFILL_MASTER_HOST: "REPLACE_PREFILL_MASTER_IP"
  PREFILL_CHILD_HOST: "REPLACE_PREFILL_CHILD_IP"
  DECODE_MASTER_HOST: "REPLACE_DECODE_MASTER_IP"
  DECODE_CHILD_HOST: "REPLACE_DECODE_CHILD_IP"

  # RDMA Fabric (cluster-specific)
  FABRIC_SOCKET_IFNAME: "REPLACE_SOCKET_IFNAME"
  FABRIC_IB_GID_INDEX: "REPLACE_IB_GID_INDEX"
  FABRIC_NCCL_IB_HCA: "REPLACE_NCCL_IB_HCA"
  FABRIC_MORI_RDMA_DEVICES: "REPLACE_MORI_RDMA_DEVICES"
  FABRIC_UCX_NET_DEVICES: "REPLACE_UCX_NET_DEVICES"
  FABRIC_RDMA_TC: "REPLACE_RDMA_TC"
  FABRIC_RDMA_SL: "REPLACE_RDMA_SL"
```

## Deploy (when enabled)

```bash
# 1) Set images
cd guides/pd-disaggregation/modelserver/amd/vllm/moriio-2p2d
kustomize edit set image \
  REPLACE_MODEL_SERVER_IMAGE=<your-vllm-moriio-rocm-image> \
  REPLACE_ROUTING_SIDECAR_IMAGE=<your-sidecar-image>

# 2) Edit moriio-config.yaml with your cluster values

# 3) Build and apply
kubectl kustomize . | kubectl -n llm-d apply -f -

# 4) Watch pods come up
kubectl -n llm-d get pods -w
```

## Related

- [Sidecar MoRI-IO README](https://github.com/llm-d/llm-d-inference-scheduler/blob/main/pkg/sidecar/proxy/MORIIO_README.md)
- [1P1D Base Guide](../moriio/README.md)

## Contact

For questions about this feature or its timeline, please contact:
- AMD team
- llm-d maintainers
