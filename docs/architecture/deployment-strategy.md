# Deployment Strategy

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

Planned deployment approach for staging and production. Initial strategy: **blue/green**. Canary is future work.

---

## Implemented

No runtime behavior is implemented. No deployment manager, no runtime targets, no slot management.

---

## Planned

Deployment manager implements blue/green cutover with recorded `deployment_id` per attempt.

---

## Future Work

- Canary deployment with metric-gated promotion
- Multi-region active-active
- Argo Rollouts / Flagger integration

---

## Concepts

| Term | Definition |
|------|------------|
| **Production model version** | `model_version_id` currently serving the active (blue) slot |
| **Candidate model version** | Version in `STAGING` or green slot awaiting cutover |
| **Staging environment** | Non-customer or synthetic-traffic environment for smoke validation |
| **Production cutover** | Atomic switch of active slot blue ↔ green after health pass |
| **Rollback target** | Last `deployment_id` with successful production status in history |
| **Health validation** | Post-deploy checks: readiness, error rate, latency, schema compatibility |
| **Deployment event history** | Append-only deploy records linked to `deployment_id` events |

---

## Blue/Green Strategy (Planned)

```
                    ┌─────────────┐
   Traffic ────────►│  BLUE slot  │  ← current production version
                    └─────────────┘
                    ┌─────────────┐
   (idle or smoke)  │ GREEN slot  │  ← candidate after staging
                    └─────────────┘

Cutover: route traffic BLUE → GREEN; GREEN becomes production; old BLUE retained for rollback
```

| Phase | Lifecycle state | Action |
|-------|-----------------|--------|
| Staging deploy | `STAGING` | Deploy candidate to green (or isolated staging target) |
| Smoke / validation | `STAGING` | Run health checks; no customer traffic required in demo |
| Cutover start | `DEPLOYING` | Begin atomic route switch |
| Cutover complete | `PRODUCTION` | Green active; update deployment history |
| Monitor | `MONITORING` | Arm drift and SLO monitors on new production version |

---

## Why Blue/Green Initially

| Factor | Rationale |
|--------|-----------|
| Rollback speed | Revert route to previous slot without redeploy artifact |
| Audit clarity | Exactly two production-capable slots; `deployment_id` maps to slot |
| Portfolio scope | Demonstrable without multi-day canary analysis |
| Failure isolation | Failed cutover leaves blue serving; maps to `DEPLOYMENT_FAILED` |

---

## Why Canary Is Future Work

Canary requires automated metric comparison during partial traffic, SLO error budgets, and longer observability windows. Adds complexity before base lifecycle proof. Documented as non-goal for Session 1–3.

---

## Why Direct Overwrite Is Forbidden

| Risk | Overwrite problem | Blue/green mitigation |
|------|-------------------|----------------------|
| Partial deploy | Broken model serves traffic | Old slot remains until validation |
| No rollback target | Unknown previous artifact | Prior slot version pinned in history |
| State desync | Control plane says prod but runtime mixed | Slot ownership explicit |
| Audit gap | No `deployment_id` boundary | Each cutover is one deployment record |

Lifecycle forbids `REGISTERED` → `PRODUCTION` and `ROLLED_BACK` → `DEPLOYING` without new approval per [transition-rules.md](transition-rules.md).

---

## Staging Deployment

1. Trigger: `PROMOTION_APPROVED` → deploy to green/staging
2. Evidence: `deployment_id`, image/artifact ref, config ref
3. Event: `StagingStarted`
4. Failure: remain or return `DEPLOYMENT_FAILED`; production slot unchanged

---

## Production Deployment

1. Trigger: `STAGING` → `DEPLOYING` after staging health pass
2. Cutover: route switch with timeout
3. Post-deploy validation: mandatory before `PRODUCTION`
4. Event: `DeploymentSucceeded`
5. Failure: `DEPLOYMENT_FAILED`; automatic revert to blue when configured

---

## Rollback Deployment

1. Trigger: `ROLLBACK_PENDING` → `ROLLBACK_IN_PROGRESS`
2. Target: `to_version_id` from deployment history (last known-good)
3. Action: route traffic to slot holding rollback target (or redeploy artifact to blue)
4. Event: `RollbackCompleted` → `ROLLED_BACK` → `MONITORING`
5. Failure: `RollbackFailed` → `FAILED` with paging

Rollback is route/artifact restoration, not a new promotion of an unapproved version.

---

## Post-Deployment Validation

| Check | Blocking? |
|-------|-----------|
| Readiness probe success | Yes |
| Error rate < threshold (5m window) | Yes |
| p99 latency < policy max | Yes (production) |
| Schema/version compatibility | Yes |

Failures during `DEPLOYING` → `DEPLOYMENT_FAILED` without marking `PRODUCTION`.

---

## Failure Handling

| Scenario | State | Recovery |
|----------|-------|----------|
| Staging deploy fail | `DEPLOYMENT_FAILED` or stay `PROMOTION_APPROVED` | Fix artifact; redeploy staging |
| Cutover timeout | `DEPLOYMENT_FAILED` | Auto-revert blue; retry `DEPLOYING` |
| Health fail post-cutover | `DEPLOYMENT_FAILED` or trigger `ROLLBACK_PENDING` | Policy-defined |
| Rollback fail | `FAILED` | Manual remediation |

---

## Related Documents

- [../workflows/deployment-workflow.md](../workflows/deployment-workflow.md)
- [../workflows/rollback-workflow.md](../workflows/rollback-workflow.md)
- [transition-rules.md](transition-rules.md)
