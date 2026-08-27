# GPUaaS Demonstration

Demo repo for **GPU-as-a-Service** on OpenShift AI using **Red Hat Kueue**.

Each use case is self-contained: flavors, queues, and workloads live in that folder. Install Kueue once, then follow the use-case README you want to run.

---

## Prerequisites

- OpenShift 4.22+ with cluster-admin (`oc`)
- GPU nodes available (demos target **3× T4**)
- In the DataScienceCluster, Kueue is **Removed** before installing the operator:

```yaml
spec:
  components:
    kueue:
      managementState: Removed
```

---

## GPU Time Slicing

Two physical T4 is sliced into **2 virtual GPUs**. 

### 1. Create the time-slicing ConfigMap

This creates a ConfigMap in the `nvidia-gpu-operator` namespace that tells the NVIDIA device plugin how to slice each GPU. The config sets `replicas: 2`, meaning each physical T4 will be advertised as 2 virtual GPUs. So in total we will get 5 GPUs.

```bash
oc apply -f time-slicing/time-slicing-config.yaml
```

### 2. Patch the GPU ClusterPolicy

The NVIDIA GPU Operator manages GPU resources through a `ClusterPolicy` CR called `gpu-cluster-policy`. This patch updates its `devicePlugin` section to reference the time-slicing ConfigMap created above, so the device plugin picks up the slicing configuration.

```bash
oc patch clusterpolicy gpu-cluster-policy -n nvidia-gpu-operator --type merge \
  -p '{"spec": {"devicePlugin": {"config": {"name": "time-slicing-config"}}}}'
```

### 3. Label GPU nodes

To make sure the resources are configured correctly, we label a specific node stating that the device-plugin.config should point to the configuration we created in the previous steps. This means also that the configuration can be applied on a per node basis.

```bash
    oc label \
    --overwrite node <node-name-1> \
    nvidia.com/device-plugin.config=Tesla-T4

    oc label \
    --overwrite node <node-name-2> \
    nvidia.com/device-plugin.config=Tesla-T4
```

After a moment, verify the advertised GPU count has doubled (e.g. 1 physical GPUs → 2):

```bash
oc get node -l nvidia.com/gpu.product=Tesla-T4-SHARED -o json \
  | jq '.items[] | {name: .metadata.name, gpus: .status.capacity["nvidia.com/gpu"]}'
```

---

## Install Red Hat Kueue

### 1. Install the operator

Via the OpenShift console (OperatorHub), or:

```bash
helm install kueue-operator ./helm-charts/kueue -n openshift-kueue-operator --create-namespace
```

Then set Kueue to **Unmanaged** in the DataScienceCluster:

```yaml
spec:
  components:
    kueue:
      managementState: Unmanaged
```

Verify:

```bash
oc get csv -n openshift-kueue-operator | grep kueue
oc get pods -n openshift-kueue-operator
```

CSV should be `Succeeded`; operator and controller pods `Running`.

### 2. Enable workload frameworks

Patch the `Kueue` CR so Jobs (and related frameworks) are managed:

```bash
oc patch kueues.kueue.openshift.io cluster --type merge -p '{
  "spec": {
    "config": {
      "integrations": {
        "frameworks": ["BatchJob","Deployment","StatefulSet","PyTorchJob","RayCluster","RayJob","TrainJob"]
      }
    }
  }
}'
```

**Note:** Prefer not enabling `Pod` together with `BatchJob` — Job child pods can stick on scheduling gates.

Namespaces used by demos are labeled `kueue.openshift.io/managed: "true"` in each use-case manifest — no separate global queue config needed.

---

## Use cases

| # | Use case | Guide | What it shows |
|---|----------|-------|---------------|
| 1 | Two models on one GPU | [`use-case1/`](./use-case1/) | Time-slice a T4 into 2 virtual GPUs; deploy Granite 4.2 and Ministral side by side |
| 2 | FIFO queue scheduling | [`use-case2/`](./use-case2/) | Jobs wait when quota is full; auto-admit when a slot frees |
| 3 | Priority & preemption | [`use-case3/`](./use-case3/) | High-priority work preempts lower-priority; preempted work returns |
| 4 | Cohort borrowing & reclaim | [`use-case4/`](./use-case4/) | Training borrows idle inference GPUs; inference reclaim takes them back |

Open a use-case folder and follow its README. Run **one use case at a time** so they do not compete for the same 3 GPUs.
