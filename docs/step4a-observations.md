# Step 4a – Back to Deployment with PruneLast=true

## Setup
- Same conditions as step 3: 4 replicas, readinessProbe initialDelaySeconds: 15
- Argo CD auto-sync with prune: true, selfHeal: true, **PruneLast=true**
- In-cluster probe: curl every 200ms (1s timeout) against svc/rollouts-demo

## Results
- Failed requests: **0 of 3,026**
- Outage window: **none**

## What happened
- Argo CD created the Deployment first; Rollout pods kept serving
- Overlap: 8 pods (4 Rollout + 4 Deployment), all selected by the Service via app: rollouts-demo
- Only after the Deployment became healthy (~15s warm-up) did Argo CD prune the Rollout

## Comparison with step 3
| | Step 3 (naive) | Step 4a (PruneLast) |
|---|---|---|
| Outage window | ~12s | none |
| Failed requests | 24 / 2,097 | 0 / 3,026 |

## Caveats
- Needs 2x capacity during the overlap (quota, node headroom, PDBs)
- Fixes Argo CD sync ordering only, not migration sequencing in general
- Going Deployment → Rollout, first revision still skips canary steps:
  never combine the migration with an image change
- No staged handover and no kept-alive fallback: once pruned, the old workload is gone