# Use Case 3: Priority & Preemption

A shared GPU queue fills with lower-priority serving work. When a **high-priority** job arrives, Kueue **preempts** it so production runs immediately. The preempted model is not deleted — it waits and **comes back** when capacity frees.

## Setup (2 T4s, one queue)

| Piece | Value |
|-------|-------|
| ClusterQueue | `t4-queue` — **2** GPUs, `withinClusterQueue: LowerPriority` |
| LocalQueue | `inference-queue` → `t4-queue` |
| Priorities | `medium-priority` (500), `high-priority` (1000) |


### Enable Kueue in the RHOAI Dashboard

Before creating queues, make sure Kueue integration is turned on in the dashboard.
Edit the `OdhDashboardConfig` CR and set `disableKueue: false`:

```bash
oc patch OdhDashboardConfig odh-dashboard-config -n redhat-ods-applications \
  --type merge -p '{"spec":{"dashboardConfig":{"disableKueue":false}}}'
```

> **Verify:** Open the RHOAI dashboard — the **Distributed workload metrics** tab should now appear under **Settings**.

### Apply Kueue resources

```bash
oc apply -f use-case3/00-namespace.yaml
oc apply -f use-case3/01-resourceflavor-t4.yaml
oc apply -f use-case3/02-clusterqueue-3gpu-fifo.yaml
oc apply -f use-case3/03-workload-priority.yaml
oc apply -f use-case3/04-localqueue.yaml
```

Confirm:

```bash
oc get workloadpriorityclass
oc get clusterqueue t4-queue -o wide
oc get localqueue -n gpuaas-demo
```

**Expect:** priorities `500` / `1000`, queue `Active`, GPU quota **2**, empty.


### Create a HardwareProfile

In the **RHOAI console**, create a HardwareProfile for project `gpuaas-demo` with these settings:

| Setting | Value |
|---------|-------|
| LocalQueue | `inference-queue` |
| GPU accelerator | **1** (T4) |
| Priority | **medium** |

> Setting medium priority here is intentional — it lets the high-priority preemptor job win during the demo.

---

## Demo

### Phase 1 — Medium-priority serving fills the queue

Deploy a **small catalog model** with that HardwareProfile.

```bash
oc get workloads -n gpuaas-demo -w
oc get pods -n gpuaas-demo
oc get clusterqueue t4-queue -o wide
```

**Expect:** model workload `Admitted`, serving pods Running, queue holding GPU capacity.

Serving is up — but only at medium priority.

### Phase 2 — High priority arrives and preempts

```bash
oc apply -f use-case3/preemptor-job.yaml
```

```bash
oc get workloads -n gpuaas-demo -w
oc get events -n gpuaas-demo --field-selector=reason=Preempted --sort-by='.lastTimestamp'
oc get clusterqueue t4-queue -o wide
```

**Expect:**
- `high-priority-inference-job` → `Admitted` (2 pods × 1 GPU)
- Model workload → `Evicted` / `Inadmissible` (not deleted, not permanently Failed)
- Serving pods stop; preemptor pods Running

**Say:** The model wasn't deleted. Kueue evicted it so high priority could run.

### Phase 3 — Preemptor gone; model returns

```bash
oc delete -f use-case3/preemptor-job.yaml
```

```bash
oc get workloads -n gpuaas-demo -w
oc get pods -n gpuaas-demo
oc get clusterqueue t4-queue -o wide
```

**Expect:** preemptor gone; model re-admits (`QuotaReserved` → `Admitted`); serving pods come back Ready.

No redeploy — capacity freed, and the medium-priority workload returns on its own.

---

## Watch (optional)

```bash
oc get workloads -n gpuaas-demo -w
watch -n 2 "oc get clusterqueue t4-queue -o wide"
watch -n 2 "oc get pods -n gpuaas-demo"
```

## Cleanup

Remove the catalog model (and HardwareProfile if you don’t need it), then:

```bash
oc delete -f use-case3/preemptor-job.yaml --ignore-not-found
oc delete -f use-case3/04-localqueue.yaml --ignore-not-found
oc delete -f use-case3/03-workload-priority.yaml --ignore-not-found
oc delete -f use-case3/02-clusterqueue-3gpu-fifo.yaml --ignore-not-found
oc delete -f use-case3/01-resourceflavor-t4.yaml --ignore-not-found
oc delete -f use-case3/00-namespace.yaml --ignore-not-found
```

## Notes

- Use a **small** catalog model so Phase 1 is fast and fits under the 2-GPU quota until the preemptor claims all three.
- Preemptor uses `kueue.x-k8s.io/queue-name` and `kueue.x-k8s.io/priority-class` on Job metadata **and** pod template labels; `spec.suspend: true`.
- Kueue needs `BatchJob` plus serving frameworks (`Deployment` / `StatefulSet`) — see root README.
- Clean up other use-case queues first if they still hold these 2 T4s.
