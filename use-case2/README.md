# Use Case 2: FIFO Queue Scheduling

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

> **NOTE:** If you are using a GPU other than Tesla T-4, you will need to create a new resourceflavor, clusterqueue and localqueue to match and update your jobs. 

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

Three jobs, one GPU each. Only two fit.

```bash
oc apply -f use-case2/04-jobs-fill-and-overflow.yaml
```

Check the status of the workload:

```bash
oc get workloads -n gpuaas-demo
NAME                    QUEUE             RESERVED IN   ADMITTED   FINISHED   AGE
job-fifo-fill-1-50b0a   team-fifo-queue   t4-queue      True                  5s
job-fifo-fill-2-c3fa2   team-fifo-queue   t4-queue      True                  5s
job-fifo-fill-3-49830   team-fifo-queue                                       5s
```

Check the status of the jobs:
```sh
$ oc get jobs -n gpuaas-demo
NAME          STATUS      COMPLETIONS   DURATION   AGE
fifo-fill-1   Running     0/1           112s       113s
fifo-fill-2   Running     0/1           112s       113s
fifo-fill-3   Suspended   0/1                      113s
```

Check the status of the clusterqueue
```sh
$ oc get clusterqueue t4-queue -o wide
NAME       COHORT   STRATEGY         PENDING WORKLOADS   ADMITTED WORKLOADS
t4-queue            BestEffortFIFO   1                   2
```

Note that the third job did not fail, it is waiting for the next opportunity to run.

### Phase 2 - Free a slot; waiter admits

Wait 5 minutes for `fifo-fill-1` to complete. Once that job finishes, `fifo-fill-3` will automatically take the next spot in the queue and start. You can also speed up this process by running the following command:

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
- Quota math: 2 GPUs x 1 GPU per job ... at most 2 admitted; the 3rd waits until capacity frees.
- Kueue needs `BatchJob` enabled - see root README.
- Clean up other use-case queues first if they still hold these 2 T4s.
