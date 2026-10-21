# Step 3 – Naive migration (Deployment → Rollout via kind swap)

## Setup
- 4 replicas, readinessProbe initialDelaySeconds: 15
- Argo CD auto-sync with prune: true, selfHeal: true
- In-cluster probe: curl every 200ms (1s timeout) against svc/rollouts-demo

## Results
- First error: 16:01:57 UTC
- Last error:  16:02:09 UTC
- Outage window: ~12 seconds (zero ready endpoints)
- Failed requests: 24 of 2,097 (all `000` – refused/timeout)
- Note: the count understates impact, since failing requests slow the probe loop;
  at production traffic levels this window means thousands of failures

## What happened
- Argo CD pruned the Deployment → ReplicaSet and pods deleted
- Rollout created in the same sync; first revision, so canary steps skipped
- New pods needed 15s warm-up; old pods' brief Terminating overlap covered ~3s

## Lesson
The migration is itself an unprotected rollout and needs its own zero-downtime plan.