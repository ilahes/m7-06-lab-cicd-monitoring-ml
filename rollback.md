# Rollback Runbook — Vision Moderation

**Owner:** ml-platform on-call

---

## When to Roll Back (triggers sustained > 5 min)

- [ ] HTTP 5xx > 1% — `AvailabilityBurnFast` firing
- [ ] p99 latency > 1 s on `/v1/moderate` — `LatencyP99High` firing
- [ ] `ModelVersionMismatch` firing on any pod
- [ ] `canary-verify.sh` exited non-zero during deploy ← **FIX: was missing**
- [ ] p95 > 400 ms sustained 15+ min (SLA breach imminent)

**Roll back first. Investigate after.**

---

## How to Roll Back

**A — GitHub Actions (preferred)**
Actions → *Deploy Vision Moderation Model* → Run workflow → set `ref` to last known-good SHA.

**B — Direct script**
```bash
export IMAGE_TAG=<last-known-good-sha>
export TARGET_ENV=production
./scripts/rollback.sh
```

**C — Kubernetes (emergency only)**
```bash
kubectl rollout undo deployment/vision-moderation -n production
kubectl rollout status deployment/vision-moderation -n production
```

---

## Verify After Rollback (wait 5 min)

- [ ] `AvailabilityBurnFast` resolved
- [ ] `LatencyP99High` resolved
- [ ] `ModelVersionMismatch` resolved
- [ ] Grafana: error rate < 0.1%, p95 < 200 ms
- [ ] `TARGET_ENV=production ./scripts/smoke.sh` exits 0

---

## Who to Notify

| When | Channel | Who |
|---|---|---|
| Rollback initiated | `#ml-incidents` | On-call (auto-paged) |
| Rollback complete | `#ml-incidents` | Team lead + partner AM |
| SLA breach confirmed | Email + Slack DM | Head of Eng + partner |

Post message:
`"Rolling back from <new-sha> to <old-sha>. Trigger: [alert name]. ETA: 5 min."`

---

## Do NOT

- ❌ Roll forward before root cause is identified
- ❌ Disable alerts to reduce noise
- ❌ Restart individual pods as first step
- ❌ Edit production manifests by hand

---

## When to Roll Forward

Only when **all** conditions are met:

1. Root cause documented in incident thread
2. Fix merged; new image passes full CI (lint → test → scan)
3. Fix verified in staging smoke tests
4. Team lead approves in `#ml-incidents`
5. 30-day error budget remaining > 20%
