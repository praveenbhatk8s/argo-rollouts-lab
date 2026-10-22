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
- At idle, KEDA scaled the Rollout from 4 (Git value) down to minReplicaCount 2
  → proof that KEDA now owns the replica count

## Results
| Time (HPA age) | CPU | Replicas | Event |
|---|---|---|---|
| 5m46s | 221% / 50% | 3 | Load hits, HPA scales out |
| 5m46s | 45% | 7 | First scale-out step |
| 6m47s | 49% | 10 | Reached maxReplicaCount |
| 8m02s | 8% | 10 | Load stopped; stabilization window running |
| 9m02s | 3% | 2 | Scaled straight back to minReplicaCount |

- Peak: 221% CPU utilization → 10 replicas (max)
- Scale-out: 3 → 10 in ~60s (two steps: 3 → 7 → 10)
- Scale-in: 10 → 2 in a single step, ~60s after load stopped (stabilization window)
- Argo CD: Synced throughout

## Observations
- CPU readings were noisy (6%–90%) under steady load: each wave of new pods
  needed 15s to pass readiness, and the HPA treats unready pods conservatively,
  which distorts the average during scale-out
- Scale-in dropped 80% of capacity in one step: the default scale-down policy
  allows 100% per period. Production would add scaleDown.policies
  (e.g. max 50% or N pods per minute) to avoid cold-start pain on the next spike
- All scaled pods were stable (yellow): no release in progress, so the Rollout
  had nothing to split

## Lessons
- One owner of the total: KEDA → HPA → Rollout.spec.replicas
- The Rollout only decides the canary/stable split within that total
- GitOps needs ignoreDifferences on the field KEDA controls, at every level
  (Deployment in 4b, Rollout here). Deleting the field from Git resets it to 1
- Never target the template-holder Deployment with KEDA or HPA after a workloadRef migration