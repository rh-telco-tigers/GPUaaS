# Use Case 1: Two Models, One GPU (Time Slicing)

A single T4 is time-sliced into 2 virtual GPUs. We deploy Granite 4.2 and Ministral side by side on the same card, each getting its own slice. A hardware profile with a node selector pins both models to the right node.

## Setup

| Piece | Value |
|-------|-------|
| Time slicing | `replicas: 2` on the target T4 |
| Hardware profile | `use-case1-hw-profile` - 1 GPU, node selector pointing at the sliced node |
| Granite 4.2 | 55 % GPU memory |
| Ministral | 35 % GPU memory |

Make sure time slicing is already configured - see root [README](../README.md#gpu-time-slicing).

Verify the node shows 2 GPUs:

```bash
oc get node <node-name> -o json \
  | jq '{name: .metadata.name, gpus: .status.capacity["nvidia.com/gpu"]}'
```

---

### Step 1 - Create hardware profile

In the OpenShift AI dashboard, go to **Settings -> Hardware profiles -> Create hardware profile**.

- **Name:** `use-case1-hw-profile`
- **GPU:** `1`
- **Node selector:** `kubernetes.io/hostname` = `<node-name>`

This makes sure both models land on the node where time slicing is active.

### Step 2 - Create Projects

We will create two projects "user-a" and "user-b" which we will use to deploy Ministral models to in the next two steps.

```
$ oc create -f use-case1\00_namespace.yaml
namespace/user-a created
namespace/user-b created
```

### Step 2 - Deploy Ministral

1. Log into OpenShift AI
2. Select AI hub -> Models
3. Search for `ministral`
4. Select `Ministral-3-3B-Instruct-2512`
5. Select Deploy
6. Leave `Model Details` at default and click Next
7. Select project "user-a" and set the **Hardware profile** to `use-case1-hw-profile`
8. Click Next
9. Select "Add custom runtime arguments" and add these serving runtime args:
```
--max-model-len=600
--gpu-memory-utilization=0.35
--enforce-eager
```
10. Click **Next**
11. Click **Deploy model**


### Step 3 - Deploy Second Copy of Ministral

Follow the steps above, but deploy the model into namespace `user-b`

### Step 4 - Check it

```bash
oc get pods -n user-a
oc get pods -n user-b
```

Both pods should be Running on the same node. Confirm GPU usage:

```bash
oc describe node <node-name> | grep -A5 "nvidia.com/gpu"
```

You should see 2/2 virtual GPUs allocated.

---

## Cleanup

Delete both model deployments from the dashboard, then remove `use-case1-hw-profile` from **Settings -> Hardware profiles** and delete the namespaces. 

## Notes

- `gpu-memory-utilization` of 0.35 + 0.35 = 0.70 - leaves some headroom on the physical card.
- `enforce-eager` stops CUDA graph allocation, which helps when two models share a card.
- `max-model-len` and `max-num-seqs` keep KV-cache size in check so both models fit.
