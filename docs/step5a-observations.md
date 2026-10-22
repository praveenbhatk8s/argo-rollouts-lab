# Step 5a – First canary release (blue → yellow) + abort drill (red)

## Setup
- Rollout with workloadRef → Deployment rollouts-demo (template holder, 0 replicas)
- Canary steps: 25% → pause{} → 50% → pause 30s → 75% → pause 30s → 100%
- No traffic router: weight = share of pods (4 replicas)
- Probe prints the colour per request; split measured over the last 20s

## Release: blue → yellow
- Change made in apps/rollouts-demo/app.yaml (Deployment image), not in the Rollout
- Deployment got a new ReplicaSet at 0 replicas (records the template, runs nothing)
- Rollout created a canary ReplicaSet and ran the steps

| Stage | Pods (canary/stable) | Measured split |
|---|---|---|
| 25% → pause{} | 1 / 3 | ~32% yellow (30 of 94 requests over 20s) |
| 50% → pause 30s | 2 / 2 | ~50% |
| 75% → pause 30s | 3 / 1 | ~75% |
| Complete | 4 / 0 | 100% yellow, yellow becomes stable |

- Argo CD while paused: **Suspended** (not Healthy) — Rollouts health check visible in GitOps UI
- Promoted manually: `kubectl argo rollouts promote rollouts-demo -n demo`

## Abort drill: yellow → red
- Paused at 25% (1 red pod), then `kubectl argo rollouts abort rollouts-demo -n demo`
- Canary scaled to 0 immediately; 100% of traffic back on yellow
- Rollout and Argo CD showed **Degraded**
- Git still said `red`: abort changed the cluster, not the desired state
- Fix: reverted the image to `yellow` in Git → Rollout Healthy instantly with **no canary steps**
  (yellow matched the existing stable ReplicaSet, so nothing to roll out)

## Lessons
- After a workloadRef migration, developers keep editing the same Deployment manifest;
  releases become progressive without changing their workflow
- Without a traffic router, weight granularity = 1 pod (25% steps at 4 replicas; 5% impossible)
- Abort protects production in seconds, but Git remains the source of truth:
  the real rollback is a Git revert (or roll forward with a fix)
- Rolling back to the current stable version is instant and safe: stable is already running
- Pod-based weighting is approximate: kube-proxy picks endpoints randomly per connection,
  so 1/4 pods measured ~32% over a 94-request sample; exact percentages need a traffic router