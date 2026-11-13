# Kueue Partial Preemption Cases

When a high-priority job needs quota, Kueue reclaims it from the elastic victims in stages: it first shrinks the victims' above-`minCount` replicas (lower priority and more recently admitted first), then fully evicts a victim that has reached its `minCount` when that is not enough, and finally fills back the replicas it over-reclaimed. The cases below verify this behavior.

## Setup

```
## 1 build image
GIT_TAG=$(git describe --tags --always --dirty)
make image-build  PLATFORMS=linux/amd64  GIT_TAG=${GIT_TAG}

# 2 build chart package and install
make helm-chart-package GIT_TAG=${GIT_TAG}
helm -n kueue-system upgrade --install kueue "$(ls -t bin/kueue-0*.tgz | head -n 1)" \
  --create-namespace \
  --set 'controllerManager.featureGates[0].name=PartialPreemption' \
  --set 'controllerManager.featureGates[0].enabled=true' \
  --set 'controllerManager.featureGates[1].name=ElasticJobsViaWorkloadSlices' \
  --set 'controllerManager.featureGates[1].enabled=true' \
  --set 'controllerManager.featureGates[2].name=ElasticJobsViaWorkloadSlicesWithPartialReplicaScaleUp' \
  --set 'controllerManager.featureGates[2].enabled=true'
```

---

## Case 1: multiple victims, partial preemption with fallback to full eviction

Two elastic victims fill the 12 CPU cohort: low-01 (count=6, minCount=2) and low-02 (count=6, minCount=2, created after low-01), both in cq-multi-batch and at the same priority — so the more recently admitted low-02 is squeezed first and fully evicted first. The two preemptors run in cq-multi-online (nominal 9). Then the two preemptors arrive one by one and leave again: preemptor-01 (5 CPU), preemptor-02 (4 CPU).

The Job integration reads the minimum from `kueue.x-k8s.io/podset-partial-preemption-min-count` on the Job metadata. Keep this exact annotation key in both victim manifests; an unrecognized key leaves the Workload without a partial-preemption minimum, so the scheduler will fully evict it.

The checks below run after the preemptor is admitted.

```bash
# 1. apply the cluster (12 CPU cohort: cq-multi-batch nominal 3 + cq-multi-online nominal 9, cohort "multi")
kubectl apply -f res/partial-preemption-multi-cluster.yaml

# 2. start the victims (12 CPU total, all in cq-multi-batch). low-02 must be created
#    after low-01: the squeeze/eviction order depends on it.
kubectl apply -f res/partial-preemption-low-01.yaml
kubectl wait job/low-01 --for=jsonpath='{.status.active}'=6 --timeout=300s
## sleep 3s to generate different admission timestamp
sleep 3s
kubectl apply -f res/partial-preemption-low-02.yaml
kubectl wait job/low-02 --for=jsonpath='{.status.active}'=6 --timeout=300s

## check (admit is workload-level, suspend is job-level; joined by job owner).
## An elastic job can own multiple workload slices (an admitted slice plus an
## unadmitted scale-up slice), so the awk keeps the largest admitted count per
## owner and skips slices that carry no admission.
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=6 suspend=false
## low-02 admit=6 suspend=false

# 3. round 1: preemptor-01 needs 5 CPU
kubectl apply -f res/partial-preemption-preemptor-01.yaml
kubectl wait job/preemptor-01 --for=jsonpath='{.status.active}'=5 --timeout=300s
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 preemptor-01 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=5 suspend=false
## low-02 admit=2 suspend=false
## preemptor-01 admit=5 suspend=false
## preemptor-01 takes 5: low-02 (admitted later) gives 4 first (6→2, its minCount), then
## low-01 is squeezed by 4 as well — but low-01 is filled back, so it nets only 1 (6→5).
## this round is a pure shrink: low-02 has 4 SchedulingGated pods (2 running + 4 gated) and
## low-01 has 1 (5 running + 1 gated) — shed replicas are recreated by the Job controller
## but kept gated (parallelism stays 6).

# 4. round 2: preemptor-02 needs 4 CPU
kubectl apply -f res/partial-preemption-preemptor-02.yaml
kubectl wait job/preemptor-02 --for=jsonpath='{.status.active}'=4 --timeout=300s
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 preemptor-01 preemptor-02 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=3 suspend=false
## low-02 admit=null suspend=true
## preemptor-01 admit=5 suspend=false
## preemptor-02 admit=4 suspend=false
## low-01 is squeezed to minCount first (5→2, taking 3), then low-02 (the newer victim) is
## fully preempted (2→0). The fillback raises low-01 by 1 (2→3).

# 5. round 3: delete preemptor-02, it releases 4 CPU
kubectl delete -f res/partial-preemption-preemptor-02.yaml
kubectl wait job/low-01 --for=jsonpath='{.status.active}'=6 --timeout=300s
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 preemptor-01 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=6 suspend=false
## low-02 admit=null suspend=true
## preemptor-01 admit=5 suspend=false
## low-01 scales up first: it takes 3 (3→6), ungating its SchedulingGated pods. The
## remaining 1 CPU cannot admit low-02: partial admission is not enabled and low-02 needs
## its full count of 6.

# 6. round 4: delete preemptor-01, it releases 5 CPU
kubectl delete -f res/partial-preemption-preemptor-01.yaml
kubectl wait job/low-02 --for=jsonpath='{.status.active}'=6 --timeout=300s
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=6 suspend=false
## low-02 admit=6 suspend=false
## the 5 released CPU (plus the 1 idle) are enough for low-02's full count: it is re-admitted
## at 6.

# 7. clean up
kubectl delete -f res/partial-preemption-low-02.yaml
kubectl delete -f res/partial-preemption-low-01.yaml
kubectl delete -f res/partial-preemption-multi-cluster.yaml
```

---
