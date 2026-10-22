# Step 6 – KEDA scaling a Rollout (no release in progress)

## Setup
- KEDA installed via Argo CD (Helm chart, pinned version)
- ScaledObject → scaleTargetRef: argoproj.io/v1alpha1 Rollout rollouts-demo
  (NOT the Deployment, which is only the workloadRef template holder at 0 replicas)
- Trigger: cpu, Utilization, target 50% of request (50m); min 2, max 10
- HPA behavior: scaleDown stabilizationWindowSeconds: 60
- Argo CD ignoreDifferences on Rollout /spec/replicas (same pattern as Deployment in step 4b)
- Load: 20 parallel curl loops against svc/rollouts-demo

## Wiring verified
- KEDA created HPA keda-hpa-rollouts-demo with REFERENCE Rollout/rollouts-demo
- Works because Rollouts implement the /scale subresource
- Right after the ScaledObject was created, KEDA scaled the Rollout from 4 (Git value)
  down to minReplicaCount 2 → proof that KEDA now owns the replica count

## Timeline (UTC, from HPA and pod events)
| Time | Event | Replicas |
|---|---|---|
| 20:02:08 | Load pod started | 2 |
| 20:02:32 | HPA rescale (CPU 221% / 50%) | 3 |
| 20:02:47 | HPA rescale | 7 |
| 20:03:47 | HPA rescale, reached maxReplicaCount | 10 |
| 20:04:46 | Load pod killed | 10 |
| 20:06:03 | HPA rescale: all metrics below target | 2 |

## Results
- Idle: **2 replicas** (minReplicaCount)
- Peak: **10 replicas** (maxReplicaCount) at **221% CPU utilization**
- Reaction time: **24s** from load start to first scaling decision
- Scale-out: **99s** from load start to max (2 → 3 → 7 → 10)
- Scale-in: **77s** from load stop to min (60s stabilization window + ~17s metric lag), 10 → 2 in a single step
- Argo CD: **Synced / Healthy throughout**. Last sync 19:57:15 (ScaledObject push), before the
  load test → self-heal never fought KEDA's replica changes

## Observations
- CPU readings were noisy (6%–90%) under steady load: each wave of new pods needed 15s
  to pass readiness, and the HPA treats unready pods conservatively, which distorts
  the average during scale-out
- Scale-in dropped 80% of capacity in one step: the default scale-down policy allows
  100% per period. Production would add scaleDown.policies (e.g. max 50% or N pods
  per minute) to avoid cold starts on the next spike
- All scaled pods were stable (yellow): no release in progress, so the Rollout had
  nothing to split

## Lessons
- One owner of the total: KEDA → HPA → Rollout.spec.replicas
- The Rollout only decides the canary/stable split within that total
- GitOps needs ignoreDifferences on the field KEDA controls, at every level
  (Deployment in 4b, Rollout here). Deleting the field from Git resets it to 1
- Never target the template-holder Deployment with KEDA or HPA after a workloadRef migration