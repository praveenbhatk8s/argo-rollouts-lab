# Step 4b – Migration via workloadRef (progressive Deployment scale-down)

## Setup
- Same conditions as steps 3 and 4a: 4 replicas, readinessProbe initialDelaySeconds: 15
- Argo CD auto-sync with prune: true, selfHeal: true, PruneLast=true
- Argo CD ignoreDifferences on Deployment /spec/replicas + RespectIgnoreDifferences=true
- Rollout with workloadRef → Deployment rollouts-demo, scaleDown: progressively
- In-cluster probe: curl every 200ms (1s timeout) against svc/rollouts-demo

## Results
- Failed requests: **0 of 2,248**
- Outage window: **none**
- End state:
  - Deployment rollouts-demo: 0/0 (kept as the pod template source)
  - Rollout rollouts-demo: 4/4 available
  - ReplicaSet rollouts-demo-7fb997b8bd (Deployment's): scaled to 0
  - ReplicaSet rollouts-demo-846c79cd8c (Rollout's): 4/4 ready

## What happened
- Rollout read its pod template from the Deployment (no template in the Rollout)
- First revision → no canary steps; Rollout scaled up to 4
- As each Rollout pod became Ready, the controller scaled the Deployment down (4 → 0)
- Service selected both sets of pods via app: rollouts-demo throughout the handover
- Argo CD stayed Synced because Deployment replica drift is ignored

## Comparison
| | Step 3 (naive) | Step 4a (PruneLast) | Step 4b (workloadRef) |
|---|---|---|---|
| Outage window | ~12s | none | none |
| Failed requests | 24 / 2,097 | 0 / 3,026 | 0 / 2,248 |
| Peak pod count | ~4 (gap) | 8 (full overlap) | ~5 (gradual) |
| Handover | Delete + create | All at once, after healthy | One pod at a time |
| Old workload after | Deleted | Deleted | Kept at 0 replicas |
| Rollback | Another outage | Another swap | Scale Deployment up, then remove Rollout |
| Mechanism | – | Argo CD sync ordering | Built into Argo Rollouts |

## Gotchas learned
- Argo CD selfHeal vs Rollouts controller fight over Deployment replicas → fixed with
  ignoreDifferences on /spec/replicas (+ RespectIgnoreDifferences)
- Deleting `replicas` from the Deployment in Git is a trap: apply resets it to 1
- Rollback order matters: scale the Deployment up FIRST, then remove the Rollout,
  otherwise the ignored replica count leaves the Deployment at 0 → full outage
- First Rollout revision still skips canary steps: never combine migration with an image change

## When to use which
- PruneLast: quick fix when you only need GitOps ordering and have 2x headroom
- workloadRef: production migrations, because of the gradual handover, lower peak
  capacity and a kept-alive rollback target