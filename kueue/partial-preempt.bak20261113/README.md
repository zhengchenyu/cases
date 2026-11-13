# Kueue Partial Preemption Cases

When a high-priority job needs quota, Kueue reclaims it from lower-priority elastic jobs in stages: first it scales all low-tier victims down toward their `minCount`, then it fully evicts the low tier (newest first), and only then it takes the above-`minCount` replicas of the higher-tier elastic victim. The cases below verify this behavior.

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

Three elastic victims fill the 12 CPU cohort: low-01 (count=4, minCount=2) and low-02 (count=4, minCount=2, created after low-01) form the low tier (both in cq-multi-batch, nominal 3), mid-01 (count=4, minCount=2) is the higher tier (also in cq-multi-batch). The three preemptors all run in cq-multi-online (nominal 9). Then three preemptors arrive one by one and leave again: preemptor-01 (2 CPU), preemptor-02 (3 CPU), preemptor-03 (4 CPU).

Reclaim logic under test: when a preemptor needs quota, Kueue first takes the low-tier victims down to their `minCount`, then fully evicts the low tier (newest first), and only then takes the above-`minCount` replicas of the higher-tier victim. mid-01's above-`minCount` part is never touched while low-tier capacity is left. Replicas shed beyond what the preemptor needs are filled back afterwards (see step 2).

The checks below run after the preemptor is admitted.

```bash
# 1. apply the cluster (12 CPU cohort: cq-multi-batch nominal 3 + cq-multi-online nominal 9, cohort "multi")
kubectl apply -f res/partial-preemption-multi-cluster.yaml

# 2. start the victims (12 CPU total, all in cq-multi-batch). low-02 must be created after low-01:
#    the full-eviction order depends on it.
kubectl apply -f res/partial-preemption-low-01.yaml
kubectl wait job/low-01 --for=jsonpath='{.status.active}'=4 --timeout=300s
kubectl apply -f res/partial-preemption-low-02.yaml
kubectl wait job/low-02 --for=jsonpath='{.status.active}'=4 --timeout=300s
kubectl apply -f res/partial-preemption-mid-01.yaml
kubectl wait job/mid-01 --for=jsonpath='{.status.active}'=4 --timeout=300s

## check (admit is workload-level, suspend is job-level; joined by job owner).
## An elastic job can own multiple workload slices (an admitted slice plus an
## unadmitted scale-up slice), so the awk keeps the largest admitted count per
## owner and skips slices that carry no admission.
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 mid-01 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=4 suspend=false
## low-02 admit=4 suspend=false
## mid-01 admit=4 suspend=false

# 3. step 1: preemptor-01 needs 2 CPU
kubectl apply -f res/partial-preemption-preemptor-01.yaml
kubectl wait job/preemptor-01 --for=jsonpath='{.status.active}'=2 --timeout=300s
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 mid-01 preemptor-01 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=3 suspend=false
## low-02 admit=3 suspend=false
## mid-01 admit=4 suspend=false
## preemptor-01 admit=2 suspend=false
## the 2 reclaimed replicas come from the low tier only, spread evenly (low-01 and low-02
## each give 1: 4→3). mid-01 is untouched.
## this step is a pure shrink: each low job has 1 SchedulingGated pod — the shed replica is
## recreated by the Job controller but kept gated (parallelism stays 4).

# 4. step 2: preemptor-02 needs 3 CPU
kubectl apply -f res/partial-preemption-preemptor-02.yaml
kubectl wait job/preemptor-02 --for=jsonpath='{.status.active}'=3 --timeout=300s
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 mid-01 preemptor-01 preemptor-02 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=3 suspend=false
## low-02 admit=null suspend=true
## mid-01 admit=4 suspend=false
## preemptor-01 admit=2 suspend=false
## preemptor-02 admit=3 suspend=false
## low-02 (created later) is squeezed to minCount first (3→2), then low-01 (3→2); still
## 1 CPU short → low-02 is fully evicted (2→0). That over-releases 1 CPU, which is filled
## back to low-01 (2→3). mid-01 is untouched.

# 5. step 3: preemptor-03 needs 4 CPU
kubectl apply -f res/partial-preemption-preemptor-03.yaml
kubectl wait job/preemptor-03 --for=jsonpath='{.status.active}'=4 --timeout=300s
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 mid-01 preemptor-01 preemptor-02 preemptor-03 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=null suspend=true
## low-02 admit=null suspend=true
## mid-01 admit=3 suspend=false
## preemptor-01 admit=2 suspend=false
## preemptor-02 admit=3 suspend=false
## preemptor-03 admit=4 suspend=false
## low-01 still holds 1 replica above minCount: it is taken first (3→2), then low-01 is
## fully evicted (2→0). Still 1 CPU short — only now the higher tier is touched: mid-01
## gives 1 (4→3).
## mid-01 gets 1 SchedulingGated pod for its shed replica (3 running + 1 gated).

# 6. step 4: delete preemptor-03, it releases 4 CPU
kubectl delete -f res/partial-preemption-preemptor-03.yaml
kubectl wait job/mid-01 --for=jsonpath='{.status.active}'=4 --timeout=300s
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 mid-01 preemptor-01 preemptor-02 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=null suspend=true
## low-02 admit=null suspend=true
## mid-01 admit=4 suspend=false
## preemptor-01 admit=2 suspend=false
## preemptor-02 admit=3 suspend=false
## mid-01 has the higher priority and takes 1 CPU first (3→4), ungating its SchedulingGated
## pod. The remaining 3 CPU cannot admit either low job: partial admission is not enabled,
## so both low workloads stay suspended (each needs its full count of 4).

# 7. step 5: delete preemptor-02 and preemptor-01, all three victims restore to 4
kubectl delete -f res/partial-preemption-preemptor-02.yaml
kubectl delete -f res/partial-preemption-preemptor-01.yaml
kubectl wait job/low-01 job/low-02 --for=jsonpath='{.status.active}'=4 --timeout=300s
awk -F'\t' 'NR==FNR{if($2!=""&&$2+0>a[$1]+0)a[$1]=$2; next} {print $1" admit="(($1 in a)?a[$1]:"null")" suspend="$2}' \
  <(kubectl get workload -n default -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].name}{"\t"}{.status.admission.podSetAssignments[0].count}{"\n"}{end}') \
  <(kubectl get job low-01 low-02 mid-01 -n default -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.suspend}{"\n"}{end}')
## expected:
## low-01 admit=4 suspend=false
## low-02 admit=4 suspend=false
## mid-01 admit=4 suspend=false

# 8. clean up
kubectl delete -f res/partial-preemption-mid-01.yaml
kubectl delete -f res/partial-preemption-low-02.yaml
kubectl delete -f res/partial-preemption-low-01.yaml
kubectl delete -f res/partial-preemption-multi-cluster.yaml
```

---
