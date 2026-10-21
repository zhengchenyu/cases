# Kueue Partial Preemption Cases

When a high-priority job needs quota, Kueue does not fully evict a low-priority elastic job. Instead it scales one PodSet down toward its `minCount` and reclaims only the replicas it needs. The four cases below verify this behavior.

## Setup

Kueue controller-manager feature gates:

```
--feature-gates=PartialPreemption=true,ElasticJobsViaWorkloadSlices=true,PartialAdmission=true,ElasticJobsViaWorkloadSlicesWithPartialReplicaScaleUp=true
```

Manager config (`kubectl edit cm -n kueue-system kueue-manager-config`, add to `controller_manager_config.yaml`):

```yaml
partialPreemption:
  enableOnDemandPreemption: true
  timeout: 30s
```

Restart the manager after changing the config:

```bash
kubectl rollout restart deployment kueue-controller-manager -n kueue-system \
  && kubectl rollout status deployment kueue-controller-manager -n kueue-system
```

All cases use this config (on-demand mode).

---

## Case 1: single victim partial preemption

Victim has 5 replicas / minCount 1. Preemptor needs 4 CPU.

```bash
# 1. apply the cluster (cq-preemptor with 8 CPU)
kubectl apply -f res/community-preemptor-01-cluster.yaml

# 2. start the victim (uses 5 CPU)
kubectl apply -f res/community-preemptor-02-victim.yaml
kubectl wait job/preemptor-victim --for=jsonpath='{.status.active}'=5 --timeout=300s

# 3. submit the high-priority preemptor (needs 4 CPU)
kubectl apply -f res/community-preemptor-03-preemptor.yaml

# 4. the preemptor is admitted, which means partial preemption worked
kubectl wait job/preemptor-high --for=jsonpath='{.status.active}'=4 --timeout=300s

# 5. check how the victim was scaled down
kubectl get job preemptor-victim -n default \
  -o jsonpath='parallelism={.spec.parallelism} suspend={.spec.suspend}{"\n"}'

# 6. clean up
kubectl delete -f res/community-preemptor-03-preemptor.yaml
kubectl delete -f res/community-preemptor-02-victim.yaml
```

**Expected:** the preemptor is admitted; the victim is not suspended, and its parallelism goes from 5 to 4 (on-demand mode reclaims only the 1 replica it needs).

---

## Case 2: multiple victims, partial preemption with fallback to full eviction

Two victims with different priorities, each 6 replicas / minCount 2, together using 12 CPU.

```bash
# 1. apply the cluster (12 CPU cohort, with low/mid/high priority classes)
kubectl apply -f res/community-preemptor-06-multi-cluster.yaml

# 2. start the low-priority victim (uses 6 CPU)
kubectl apply -f res/community-preemptor-07-victim-low.yaml
kubectl wait job/multi-victim-low --for=jsonpath='{.status.active}'=6 --timeout=300s

# 3. start the higher-priority victim (both victims now use 12 CPU)
kubectl apply -f res/community-preemptor-08-victim-demand.yaml
kubectl wait job/multi-victim-demand --for=jsonpath='{.status.active}'=6 --timeout=300s

# 4. round 1: preemptor needs 5 CPU
kubectl apply -f res/community-preemptor-09-reclaimer-01.yaml
kubectl wait job/multi-reclaimer-01 --for=jsonpath='{.status.active}'=5 --timeout=300s
kubectl get job multi-victim-low multi-victim-demand -n default \
  -o jsonpath='{range .items[*]}{.metadata.name}{" parallelism="}{.spec.parallelism}{" suspend="}{.spec.suspend}{"\n"}{end}'

# 5. round 2: preemptor needs another 3 CPU
kubectl apply -f res/community-preemptor-10-reclaimer-02.yaml
kubectl wait job/multi-reclaimer-02 --for=jsonpath='{.status.active}'=3 --timeout=300s
kubectl get job multi-victim-low multi-victim-demand -n default \
  -o jsonpath='{range .items[*]}{.metadata.name}{" parallelism="}{.spec.parallelism}{" suspend="}{.spec.suspend}{"\n"}{end}'

# 6. clean up
kubectl delete -f res/community-preemptor-10-reclaimer-02.yaml
kubectl delete -f res/community-preemptor-09-reclaimer-01.yaml
kubectl delete -f res/community-preemptor-08-victim-demand.yaml
kubectl delete -f res/community-preemptor-07-victim-low.yaml
```

**Expected:**

| Step | Low-priority victim | Higher-priority victim | Preemptor |
|---|---|---|---|
| After round 1 | parallelism 2 (scaled to minCount, not evicted) | parallelism 5 (reclaims only 1) | reclaimer-01 admitted |
| After round 2 | suspend=true (already at minCount, falls back to full eviction) | parallelism 4 (reclaims only 1) | reclaimer-02 admitted |

---

## Case 3: multiple victims with a per-round reclaim cap, gradual preemption

Two victims with the same priority, each 6 replicas / minCount 2, both annotated with `partial-preemption-max-reclaimed-count: "2"` (each reclaims at most 2 per round). The preemptor needs 5 CPU, which cannot be freed in one round, so it converges over several rounds.

```bash
# 1. reuse the Case 2 cluster (re-apply if already cleaned up)
kubectl apply -f res/community-preemptor-06-multi-cluster.yaml

# 2. admit the two victims in order (a fixed order keeps the result stable)
kubectl apply -f res/community-preemptor-11-victim-gradual-01.yaml
kubectl wait job/gradual-victim-01 --for=jsonpath='{.status.active}'=6 --timeout=300s
kubectl apply -f res/community-preemptor-12-victim-gradual-02.yaml
kubectl wait job/gradual-victim-02 --for=jsonpath='{.status.active}'=6 --timeout=300s

# 3. submit the high-priority preemptor (needs 5 CPU)
kubectl apply -f res/community-preemptor-13-reclaimer-gradual.yaml

# 4. the preemptor is admitted; check the final victim counts
kubectl wait --for=condition=Available job/gradual-reclaimer -n default --timeout=5m
kubectl get job gradual-victim-01 gradual-victim-02 -n default \
  -o jsonpath='{range .items[*]}{.metadata.name}{" parallelism="}{.spec.parallelism}{" suspend="}{.spec.suspend}{"\n"}{end}'

# 5. clean up
kubectl delete -f res/community-preemptor-13-reclaimer-gradual.yaml
kubectl delete -f res/community-preemptor-12-victim-gradual-02.yaml
kubectl delete -f res/community-preemptor-11-victim-gradual-01.yaml
```

**Expected:** the preemptor is admitted; one victim ends at parallelism 4 and the other at 3 (final state {3,4}: each reclaims at most 2 per round, and the last round reclaims only 1, so there is no over-reclaim); neither victim is suspended.

---

## Case 4: partial target timeout, fallback to full eviction

Create a victim whose `reclaimTargetCount` can never converge, so it times out and falls back to full eviction. The victim is a plain `Workload` (with the partial-preemption annotation and a minCount, but no controller to respond to `reclaimTargetCount`), so it always reaches the timeout.

```bash
# 1. apply the single-victim cluster
kubectl apply -f res/community-preemptor-01-cluster.yaml

# 2. create the plain Workload victim (uses 5 CPU) and wait until admitted
kubectl apply -f res/community-preemptor-04-victim-timeout.yaml
kubectl wait workload/preemptor-victim-raw -n default --for=condition=Admitted --timeout=2m

# 3. submit the high-priority preemptor (needs 4 CPU)
kubectl apply -f res/community-preemptor-03-preemptor.yaml

# 4. first scheduling: the victim gets a reclaimTargetCount, but nothing responds, so count stays 5
kubectl get workload preemptor-victim-raw -n default \
  -o jsonpath='reclaimTargetCount={.status.admission.podSetAssignments[0].reclaimTargetCount} count={.spec.podSets[0].count}{"\n"}'

# 5. wait past the timeout (30s)
sleep 40

# 6. the next scheduling round falls back to full eviction: the scheduler sets Evicted/Preempted on the victim
kubectl get workload preemptor-victim-raw -n default \
  -o jsonpath='evicted={.status.conditions[?(@.type=="Evicted")].status} preempted={.status.conditions[?(@.type=="Preempted")].status}{"\n"}'

# 7. verify the full chain "partial preempt first, then fall back to full eviction" via events
kubectl get events -n default \
  --field-selector involvedObject.name=preemptor-victim-raw \
  --sort-by=.lastTimestamp \
  -o custom-columns=REASON:.reason,MSG:.message

# 8. clean up
kubectl delete -f res/community-preemptor-03-preemptor.yaml
kubectl delete -f res/community-preemptor-04-victim-timeout.yaml
```

**Expected:** the events show `PartiallyPreempted` (scale-down attempted first) followed by `Preempted` (the partial target timed out, so it falls back to full eviction), and the victim conditions are `Evicted=True` and `Preempted=True`. This proves the timeout-then-full-eviction fallback works.

> Note: the victim here is a plain Workload with no parent Job to release its quota, so after it is evicted the quota is not actually freed and the preemptor is not admitted. This case verifies the fallback logic through the event chain, not through preemptor admission. To see the preemptor actually admitted, use a real elastic slice Job as the victim — but then the victim scales down immediately and never stays long enough to hit the timeout.
