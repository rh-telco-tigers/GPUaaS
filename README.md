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
| 1 | FIFO queue scheduling | [`use-case1/`](./use-case1/) | Jobs wait when quota is full; auto-admit when a slot frees |
| 2 | Priority & preemption | [`use-case2/`](./use-case2/) | High-priority work preempts lower-priority; preempted work returns |
| 3 | Cohort borrowing & reclaim | [`use-case3/`](./use-case3/) | Training borrows idle inference GPUs; inference reclaim takes them back |

Open a use-case folder and follow its README. Run **one use case at a time** so they do not compete for the same 3 GPUs.
