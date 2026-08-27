# Use Case 3: Cohort Borrowing & Reclaim

Inference keeps reserved GPUs for serving. When those GPUs are idle, training can **borrow** them. When serving needs them back, inference **reclaims**.

## Setup (3 T4s, one cohort)

| Team | Queue | Quota | Rule |
|------|-------|-------|------|
| Inference | `t4-inferencing-queue` | 2 GPUs | lends at most 1 |
| Training | `t4-training-queue` | 1 GPU | can borrow 1 |

```bash
oc apply -f use-case4/00-namespace.yaml
oc apply -f use-case4/01-resourceflavor-t4.yaml
oc apply -f use-case4/02-cohort-clusterqueues.yaml
oc apply -f use-case4/03-localqueues.yaml
```

Confirm both ClusterQueues are `Active` and empty:

```bash
oc get clusterqueue t4-inferencing-queue t4-training-queue -o wide
oc get localqueue -n gpuaas-cohort-demo
```

In the **RHOAI console**, create a HardwareProfile on project `gpuaas-cohort-demo` tied to LocalQueue **`inference-queue`** (1 GPU / `t4-gpu`).

---

## Demo

### Phase 1 — Inference on its own quota

Deploy **one small catalog model** with that HardwareProfile.

```bash
oc get workloads -n gpuaas-cohort-demo -w
oc get clusterqueue t4-inferencing-queue t4-training-queue -o wide
```

**Expect:** model `Admitted`, inference **1/2**, training empty. One GPU is idle and lendable.

### Phase 2 — Training on its own quota

```bash
oc apply -f use-case4/04-phase2-training-own.yaml
oc get workloads -n gpuaas-cohort-demo
oc get clusterqueue t4-inferencing-queue t4-training-queue -o wide
```

**Expect:** `training-job-1` `Admitted`. Inference **1/2**, training **1/1**. Still no borrow.

### Phase 3 — Training borrows

```bash
oc apply -f use-case4/05-phase3-training-borrow.yaml
oc get workloads -n gpuaas-cohort-demo -w
oc get clusterqueue t4-inferencing-queue t4-training-queue -o wide
```

**Expect:** `training-job-2` `Admitted` via borrow. Training **2** (1 own + 1 borrowed). All **3** GPUs busy.

### Phase 4 — Inference reclaims

Deploy a **second small catalog model** with the same HardwareProfile.

```bash
oc get workloads -n gpuaas-cohort-demo -w
oc get events -n gpuaas-cohort-demo --field-selector=reason=Preempted --sort-by='.lastTimestamp'
oc get clusterqueue t4-inferencing-queue t4-training-queue -o wide
```

**Expect:** second model `Admitted`; `training-job-2` evicted; `training-job-1` stays. Inference **2/2**, training **1/1**.

---

## Watch (optional)

```bash
oc get workloads -n gpuaas-cohort-demo -w
watch -n 2 "oc get clusterqueue t4-inferencing-queue t4-training-queue -o wide"
```

## Cleanup

Remove both catalog models (and the HardwareProfile if you don’t need it), then:

```bash
oc delete -f use-case4/05-phase3-training-borrow.yaml --ignore-not-found
oc delete -f use-case4/04-phase2-training-own.yaml --ignore-not-found
oc delete -f use-case4/03-localqueues.yaml --ignore-not-found
oc delete -f use-case4/02-cohort-clusterqueues.yaml --ignore-not-found
oc delete -f use-case4/01-resourceflavor-t4.yaml --ignore-not-found
oc delete -f use-case4/00-namespace.yaml --ignore-not-found
```

## Notes

- Use **small** catalog models (1 GPU each) so Phase 1 / 4 stay fast.
- Borrow needs the same flavor on both queues (`t4-gpu`).
- Phase 4 needs `reclaimWithinCohort: Any` on the inference ClusterQueue.
- Clean up use-case1/2 queues first if they still hold these 3 T4s.
- Kueue needs `BatchJob` plus serving frameworks (`Deployment` / `StatefulSet`) — see root README.
