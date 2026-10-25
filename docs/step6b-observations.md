# Step 6b – Autoscaling during a canary (KEDA + Rollouts)

## Setup
- Rollout with workloadRef → Deployment rollouts-demo (template holder, 0 replicas)
- Canary steps: 25% → pause{} → 50% → pause 30s → 75% → pause 30s → 100%
- No traffic router: weight = share of pods
- KEDA ScaledObject → Rollout rollouts-demo, cpu trigger (Utilization 50%), min 2, max 10,
  scaleDown stabilizationWindowSeconds: 60
- Argo CD ignoreDifferences on Rollout and Deployment /spec/replicas
- Logger: every 10s records total (spec.replicas), Running stable/canary pods
  (by stableRS / currentPodHash), and HPA CPU → docs/step6b-raw.log

## Part A – Canary count vs autoscaled total (release: orange, paused at setWeight 25)
KEDA total frozen with annotation autoscaling.keda.sh/paused-replicas

| Time (UTC) | Total (KEDA) | Canary | Stable | Pods running | HPA |
|---|---|---|---|---|---|
| 18:02:04 | 2 (autoscaled, idle) | 1 | 2 | **3** | 4% / 50% |
| 18:03:41 | 4 (paused-replicas=4) | 1 | 3 | 4 | **absent** |
| 18:05:16 | 8 (paused-replicas=8) | 2 | 6 | 8 | **absent** |

- Weight applies to the autoscaled total: the canary's traffic share stays at 25%,
  but the absolute number of canary pods grows with the total (1 → 1 → 2, and 3 at 10 in Part B)
  → blast radius in pods grows at peak load
- **Round-up surge:** at total=2 the Rollout ran 3 pods (0.5 → 1 canary, 1.5 → 2 stable);
  at total=10, 11 pods (Part B). The split only lands exactly on the total when it divides evenly
- **KEDA pause removes the HPA:** the logger's hpa field went empty the moment paused-replicas
  was set (18:03:41); KEDA held the count itself and recreated the HPA on unpause (age reset to 0s).
  `kubectl get hpa` shows nothing while paused

## Part B – CPU-hungry canary (release: purple + busybox CPU-burn sidecar, no external load)

### Timeline (UTC, 2026-10-25, from HPA/Rollout events and logger)
| Time | Event | Total | Canary / Stable | HPA CPU |
|---|---|---|---|---|
| 18:12:42 | HPA scale-out (canary CPU only, no external load) | 6 | – | above target |
| 18:13:12 | HPA scale-out | 8 | – | above target |
| 18:13:42 | HPA reached maxReplicaCount | 10 | 2 / 6 | 126% / 50% |
| 18:13:53 | Rollout rebalanced to 25% of new total | 10 | **3 / 8** | 126% |
| 18:14:04–18:14:25 | Pinned at max | 10 | 3 / 8 | 110% |
| 18:14:30 | **RolloutAborted** (revision 14) | 10 | → 0 / 8 | collapsing |
| 18:15:27 | HPA scale-in: all metrics below target | 5 | 0 / 5 | low |
| 18:15:42 | HPA scale-in | 2 | 0 / 2 | ~2–5% |

### Results
| | Predicted | Observed |
|---|---|---|
| Scale-out | to max 10 | 2 → 6 → 8 → 10 in **60s** |
| HPA CPU while pinned at 10 | ~100% | **110–126%** (over 2x target at max) |
| Canary pods at 10 replicas | ~3 | **3** (grew 1 → 2 → 3 with the total) |
| Scale-in after abort | after 60s window | **72s** to 2 replicas (10 → 5 → 2) |

- An earlier run (2026-10-22) behaved the same: 127% at 2 replicas → scaled to 10 → pinned at 105–184%
- `kubectl top pods` after abort and revert (docs/step6b-top.txt): 2 stable pods at 2m CPU each,
  i.e. the workload's real idle need; everything above that was caused by the canary

### Finding: pod-based canary surges above the autoscaled total
- At total=10 the Rollout ran **11 pods** (8 stable + 3 canary)
- Without a traffic router, both sides of the split round up (7.5 → 8, 2.5 → 3)
- KEDA's maxReplicaCount is therefore not a hard pod ceiling during a canary

### Side observation: metrics API unavailable
- At 17:49–17:50 the HPA logged "unable to fetch metrics from resource metrics API"
  (metrics-server not ready after cluster start). While metrics are unavailable the
  HPA cannot make decisions and holds the current replica count
- A CPU-based autoscaler depends on metrics-server's availability: worth monitoring

## What happened
- The canary's CPU regression pushed the HPA's pod-averaged utilization over target,
  with zero real traffic
- The HPA scaled the whole Rollout; the Rollout kept the 25% split, so the autoscaler
  created MORE canary pods, multiplying the bad version (1 → 3)
- Stable pods sat nearly idle: capacity wasted on autoscaling a bug
- Nothing stopped it automatically: CPU throttling doesn't restart containers
  (only OOM or liveness failures do). It stayed pinned at max until a manual abort
- Abort removed the canary pods → average CPU collapsed → back to 2 replicas 72s later

## Mitigations
- Automated canary analysis on canary-specific metrics (CPU/latency/errors per
  rollouts-pod-template-hash) → abort without a human
- Traffic router + setCanaryScale → canary pod count fixed, independent of autoscaling
  (also removes the round-up surge above the total)
- Scale on external demand signals (RPS, queue depth via KEDA) rather than pod CPU,
  so a canary regression can't inflate the scaling signal
- KEDA paused-replicas for high-risk releases (remember: HPA is removed while paused)
- maxReplicaCount as a cost ceiling, keeping in mind the canary can surge past it