# Use Case 1: FIFO Queue Scheduling

Teams submit more GPU work than the cluster can run at once. Without a queue, excess jobs fail and people retry by hand. With Kueue, jobs that exceed quota **wait** and are **admitted automatically** the moment a slot opens.

## Setup (3 T4s, one queue)

| Piece | Value |
|-------|-------|
| ClusterQueue | `t4-queue` — **2** GPUs |
| LocalQueue | `team-fifo-queue` → `t4-queue` |
| Jobs | `fifo-fill-1..3` — 1 GPU each |

```bash
oc apply -f use-case2/00-namespace.yaml
oc apply -f use-case2/01-resourceflavor-t4.yaml
oc apply -f use-case2/02-clusterqueue-3gpu-fifo.yaml
oc apply -f use-case2/03-localqueue.yaml
```

Confirm:

```bash
oc get clusterqueue t4-queue -o wide
oc get localqueue -n gpuaas-demo
oc get workloads -n gpuaas-demo
```

**Expect:** queue `Active`, GPU quota **2**, empty (`ADMITTED` / `PENDING` = 0).

---

## Demo

### Phase 1 — Fill the queue, then overflow

Four jobs, one GPU each. Only three fit.

```bash
oc apply -f use-case2/04-jobs-fill-and-overflow.yaml
```

```bash
oc get workloads -n gpuaas-demo -w
oc get jobs -n gpuaas-demo
oc get clusterqueue t4-queue -o wide
```

**Expect:**
- `fifo-fill-1`, `fifo-fill-2` → `Admitted` / Running
- `fifo-waiter-3` → pending / `Inadmissible` (no free GPU quota)
- Queue at **2 / 2**

The third job did not fail — it is waiting.

### Phase 2 — Free a slot; waiter admits

```bash
oc delete job fifo-fill-1 -n gpuaas-demo
```

```bash
oc get workloads -n gpuaas-demo -w
oc get clusterqueue t4-queue -o wide
```

**Expect:** `fifo-waiter-3` moves `Inadmissible` → `QuotaReserved` → `Admitted` within seconds.

No retry script. Kueue saw the free slot and admitted the next job.

### Phase 3 — Final state

```bash
oc get jobs -n gpuaas-demo
oc get workloads -n gpuaas-demo
oc get clusterqueue t4-queue -o wide
```

**Expect:** Two workloads running (`fifo-fill-2`, `fifo-fill-3`), zero pending. The queue self-healed.

---

## Watch (optional)

```bash
oc get workloads -n gpuaas-demo -w
watch -n 2 "oc get clusterqueue t4-queue -o wide"
```

## Cleanup

```bash
oc delete -f use-case2/04-jobs-fill-and-overflow.yaml --ignore-not-found
oc delete -f use-case2/03-localqueue.yaml --ignore-not-found
oc delete -f use-case2/02-clusterqueue-3gpu-fifo.yaml --ignore-not-found
oc delete -f use-case2/01-resourceflavor-t4.yaml --ignore-not-found
oc delete -f use-case2/00-namespace.yaml --ignore-not-found
```

## Notes

- Jobs use `spec.suspend: true` and `kueue.x-k8s.io/queue-name: team-fifo-queue` on Job metadata **and** pod template labels.
- Quota math: 2 GPUs × 1 GPU per job → at most 2 admitted; the 3rd waits until capacity frees.
- Kueue needs `BatchJob` enabled — see root README.
- Clean up other use-case queues first if they still hold these 2 T4s.
