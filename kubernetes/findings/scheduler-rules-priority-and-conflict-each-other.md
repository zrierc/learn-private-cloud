# Kube Scheduler Priorities and Mechanism

> [!NOTE]
> Related docs: [Scheduler](../8-scheduler.md)

I just realize between Node Affinity, Pod Affinity, Pod Anti-Affinity, and Toleration;
we can define all of them in one **Pod Spec**. Then I was thinking:

```txt
What if we create all the rules (Node Affinity, Pod Affinity, etc) that are inversely
related to one another? Which one is got more priority?

# ID
Gimana kalau buat banyak rules (buat Node Affinity, Pod Affinity, etc) tapi rules
nya saling bertolak belakang? Mana yang dapat prioritas?
```

---

## Answer

### 1. Priority Order in kube-scheduler

The scheduler processes rules in this order:

```
Phase 1: FILTERING (hard gates, sequential)
  Must ALL pass or Pod stays unscheduled (Pending)

  Step 1: Taints/Tolerations
    nodes where pod has no matching toleration → eliminated immediately

  Step 2: nodeAffinity (required)
    nodes not matching requiredDuring... → eliminated

  Step 3: podAffinity (required)
    nodes not satisfying requiredDuring... co-location → eliminated

  Step 4: podAntiAffinity (required)
    nodes violating requiredDuring... separation → eliminated

  Result: list of feasible nodes

Phase 2: SCORING (soft preferences, all run and scores added up)
  Runs on remaining feasible nodes after filtering

  nodeAffinity (preferred)       → adds score
  podAffinity (preferred)        → adds score
  podAntiAffinity (preferred)    → adds score
  resource balancing             → adds score
  topology spread                → adds score

  Result: highest scoring node wins
```

So there's no single "taint beats affinity" rule — they operate in different phases:

```
Taints/Tolerations    →  always filtering phase (hard)
required rules        →  filtering phase (hard)
preferred rules       →  scoring phase (soft)
```

---

### 2. Can You Declare All of Them Together?

**Yes, you can.** And the scheduler processes all of them:

```yaml
spec:
  tolerations: # evaluated first in filter
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule

  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: # filter phase
        nodeSelectorTerms:
          - matchExpressions:
              - key: disktype
                operator: In
                values: [ssd]
      preferredDuringSchedulingIgnoredDuringExecution: # score phase
        - weight: 50
          preference:
            matchExpressions:
              - key: region
                operator: In
                values: [us-east]

    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: # filter phase
        - labelSelector:
            matchLabels:
              app: cache
          topologyKey: kubernetes.io/hostname

    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution: # filter phase
        - labelSelector:
            matchLabels:
              app: backend
          topologyKey: kubernetes.io/hostname
```

All of these get processed together in the same scheduling cycle.

---

### What Happens With Conflicting Rules

There are a few conflict scenarios:

#### Conflict in filter phase — Pod goes Pending forever

```
Rule A (nodeAffinity required): only nodes with disktype=ssd
Rule B (podAffinity required):  must be on same node as cache pod
                                cache pod runs on node-3 (disktype=hdd)

Result:
  nodeAffinity eliminates node-3 (not ssd)
  podAffinity requires node-3 (where cache lives)
  NO node satisfies both
  → Pod stays Pending indefinitely
  → kubectl describe pod will show FailedScheduling event
  → you have to fix the rules
```

Kubernetes does not pick a winner — it just can't schedule the Pod.

#### Conflict between filter and score — filter always wins

```
nodeAffinity required: only ssd nodes              ← filter, eliminates hdd nodes
nodeAffinity preferred: prefer us-west region      ← score, preferences among ssd nodes

No conflict here — score only runs on nodes that survived filtering.
Score never overrides filter.
```

#### Conflict within scoring phase — weights decide

```
podAffinity preferred (weight 80):    prefer node-1 (where cache is)
nodeAffinity preferred (weight 20):   prefer node-2 (us-east region)

node-1 score: 80  (satisfies podAffinity preference)
node-2 score: 20  (satisfies nodeAffinity preference)

→ node-1 wins because higher weight
```

Weight is your control knob when soft rules conflict:

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 80 # 1-100, higher = more important
    preference: ...
  - weight: 20
    preference: ...
```

---

### Full Decision Flow

```
Pod created (Pending)
│
├─ Filter: Does pod tolerate node's taints?
│    NO  → node eliminated
│    YES → continue
│
├─ Filter: Does node satisfy nodeAffinity required?
│    NO  → node eliminated
│    YES → continue
│
├─ Filter: Does node satisfy podAffinity required?
│    NO  → node eliminated
│    YES → continue
│
├─ Filter: Does node satisfy podAntiAffinity required?
│    NO  → node eliminated
│    YES → node is FEASIBLE
│
├─ Any feasible nodes left?
│    NO  → Pod stays Pending, FailedScheduling event
│           kubectl describe pod to see why
│    YES → enter scoring phase
│
└─ Score feasible nodes:
     nodeAffinity preferred      → +weight if satisfied
     podAffinity preferred       → +weight if satisfied
     podAntiAffinity preferred   → +weight if satisfied
     resource balancing          → +score if node is less utilized
     topology spread             → +score if helps balance
     highest total score wins    → Pod assigned to that node
```

---

### Practical Rule

```
If rules conflict in filter phase    →  Pod Pending, you MUST fix rules
If rules conflict in score phase     →  higher weight wins, pod still schedules
Filter phase conflict                →  no winner, hard stop
Score phase conflict                 →  weights decide, always resolves
```

When you hit a Pod stuck in Pending always run:

```bash
kubectl describe pod <pod-name>
# look at Events section at the bottom
# it will tell you exactly which rule eliminated which nodes
# e.g. "0/3 nodes available: 1 node had taint, 2 didn't match node affinity"
```

That message format tells you exactly how many nodes were eliminated by each
rule — very useful for debugging conflicting constraints.
