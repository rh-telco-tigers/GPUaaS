# Use Case 1: Two Models, One GPU (Time Slicing)

A single T4 is time-sliced into 2 virtual GPUs. We deploy Granite 4.2 and Ministral side by side on the same card, each getting its own slice. A hardware profile with a node selector pins both models to the right node.

## Setup

| Piece | Value |
|-------|-------|
| Time slicing | `replicas: 2` on the target T4 |
| Hardware profile | `use-case1-hw-profile` — 1 GPU, node selector pointing at the sliced node |
| Granite 4.2 | 55 % GPU memory |
| Ministral | 35 % GPU memory |

Make sure time slicing is already configured — see root [README](../README.md#gpu-time-slicing).

Verify the node shows 2 GPUs:

```bash
oc get node <node-name> -o json \
  | jq '{name: .metadata.name, gpus: .status.capacity["nvidia.com/gpu"]}'
```

---

## Demo

### Step 1 — Create hardware profile

In the OpenShift AI dashboard, go to **Settings → Hardware profiles → Create hardware profile**.

- **Name:** `use-case1-hw-profile`
- **GPU:** `1`
- **Node selector:** `kubernetes.io/hostname` = `<node-name>`

This makes sure both models land on the node where time slicing is active.

### Step 2 — Deploy Granite 4.2

From the model catalog, deploy **ministral-3B**.Pick `use-case1-hw-profile` as the hardware profile and add these serving runtime args:

```
--max-model-len=600
--gpu-memory-utilization=0.35
--enforce-eager
```


### Step 3 — Deploy Ministral

Deploy **granite-4.2** the same way — same hardware profile, different args:

```
--max-model-len=2048
--max-num-seqs=4
--gpu-memory-utilization=0.55
--enforce-eager
--enable-auto-tool-choice
--tool-call-parser=qwen3_coder
--reasoning-parser=nemotron_v3
```

### Step 4 — Check it

```bash
oc get pods -o wide | grep -E "granite|ministral"
```

Both pods should be Running on the same node. Confirm GPU usage:

```bash
oc describe node <node-name> | grep -A5 "nvidia.com/gpu"
```

You should see 2/2 virtual GPUs allocated.

---

## Cleanup

Delete both model deployments from the dashboard, then remove `use-case1-hw-profile` from **Settings → Hardware profiles**.

## Notes

- `gpu-memory-utilization` of 0.55 + 0.35 = 0.90 — leaves some headroom on the physical card.
- `enforce-eager` stops CUDA graph allocation, which helps when two models share a card.
- `max-model-len` and `max-num-seqs` keep KV-cache size in check so both models fit.
