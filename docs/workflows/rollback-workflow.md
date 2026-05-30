# Rollback Workflow

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

---

## Implemented

No runtime behavior is implemented.

---

## Planned

Restore last known-good production version via blue/green slot switch per [deployment-strategy.md](../architecture/deployment-strategy.md).

---

## Future Work

- Automated rollback on SLO policy
- Partial rollback (traffic percentage) with canary

---

## Purpose

Revert production to a prior approved version when deploy, drift response, or SLO policy requires it.

---

## Actors

| Actor | Role |
|-------|------|
| `user:operator` | Request rollback |
| `policy:rollback` | Automated trigger from SLO/drift severity |
| `service:deployment-manager` | Execute slot switch / redeploy |
| `service:control-plane` | Eligibility checks, state transitions |

---

## Inputs

- `model_id`
- Current production `model_version_id` (`from_version_id`)
- Rollback target `to_version_id`
- `reason` (required)

---

## Outputs

- `rollback_id`
- Restored production version
- States: `ROLLED_BACK` → `MONITORING`
- Rollback event in audit store

---

## Required Evidence

| Step | Evidence |
|------|----------|
| Request | `reason`, eligibility check record |
| Eligibility | Target exists in deployment history with `status=succeeded` |
| Execute | `deployment_id` or slot switch proof |
| Complete | Post-rollback health validation URI |

---

## Rollback Eligibility

| Rule | Detail |
|------|--------|
| Target must be prior successful production deploy | From deployment history |
| Target artifact checksum must match registry | Integrity |
| Cannot rollback to never-deployed version | Reject |
| `ROLLED_BACK` → `DEPLOYING` without approval | **Forbidden** — new version needs promotion |

---

## State Transitions

```
PRODUCTION | MONITORING → ROLLBACK_PENDING → ROLLBACK_IN_PROGRESS → ROLLED_BACK → MONITORING
                                              ↓
                                           FAILED
```

---

## Events Emitted

`RollbackRequested` → `RollbackStarted` → `RollbackCompleted` | `RollbackFailed`

---

## Failure Modes

| Failure | State | Impact |
|---------|-------|--------|
| Target artifact missing | Request rejected | Production unchanged |
| Slot switch fail | `FAILED` | Incident; possible split-brain |
| Health fail post-rollback | `RollbackFailed` → `FAILED` | Manual ops |
| Concurrent deploy | Conflict error | No partial rollback |

---

## Recovery Paths

| From | Action |
|------|--------|
| `ROLLBACK_IN_PROGRESS` fail | Manual remediation; may retry `ROLLBACK_PENDING` |
| `FAILED` | `remediate_rollback` with ops approval |

---

## Validation After Rollback

- Readiness probes on restored version
- Error rate below threshold (5m)
- Monitoring re-armed on `to_version_id`
- Drift baselines reset or flagged for review (planned)

---

## Current Implementation Status

Documentation only. No rollback API, no deployment manager, no eligibility service.
