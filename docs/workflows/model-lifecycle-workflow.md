# Model Lifecycle Workflow

**Status:** Architecture Phase  
**Implementation Status:** Not Started

All workflows in this document are **planned**. No workflow is executable in the repository today.

---

## Implemented

Nothing.

---

## Planned

End-to-end lifecycle orchestration as defined below.

---

## Required Lifecycle

```
Train → Evaluate → Register → Promote → Deploy → Monitor
  → Detect Drift → Retrain → Validate → Promote → Rollback
```

---

## Happy Path (Planned)

| Step | Action | State transition |
|------|--------|------------------|
| 1 | Operator or CI prepares dataset and config | → `DATA_PREPARED` |
| 2 | Training orchestrator starts job | `DATA_PREPARED` → `TRAINING` |
| 3 | Training completes | `TRAINING` → `EVALUATING` |
| 4 | Evaluation gate passes | `EVALUATING` → `REGISTERED` |
| 5 | Operator requests promotion | `REGISTERED` → `PROMOTION_PENDING` |
| 6 | Approver approves | `PROMOTION_PENDING` → `PROMOTION_APPROVED` |
| 7 | Deployment manager deploys to staging | `PROMOTION_APPROVED` → `STAGING` → `DEPLOYING` |
| 8 | Staging validation (manual or smoke) | remain `STAGING` or advance |
| 9 | Production deploy succeeds | `DEPLOYING` → `PRODUCTION` → `MONITORING` |
| 10 | Monitoring stable | remain `MONITORING` |

**Outcome:** Model version serving production with full audit chain.

---

## Drift Path (Planned)

| Step | Action | State transition |
|------|--------|------------------|
| 1 | Monitoring detects drift above threshold | `MONITORING` → `DRIFT_DETECTED` |
| 2 | Control plane or operator initiates retrain | `DRIFT_DETECTED` → `RETRAINING` |
| 3 | New training run (linked to parent version) | `RETRAINING` → `VALIDATING` (after train+eval) |
| 4 | Challenger passes validation vs baseline | `VALIDATING` → `PROMOTION_PENDING` |
| 5 | Approver promotes challenger | → `PROMOTION_APPROVED` → deploy path |
| 6 | New version reaches `MONITORING` | production updated |

**Outcome:** Drift response produces new version with governed promotion.

---

## Rejected Promotion Path (Planned)

| Step | Action | State transition |
|------|--------|------------------|
| 1 | Version in `REGISTERED` or post-`VALIDATING` | → `PROMOTION_PENDING` |
| 2 | Approver rejects with comment | `PROMOTION_PENDING` → `PROMOTION_REJECTED` |
| 3 | Deployment blocked | no path to `STAGING` / `DEPLOYING` |
| 4 | Remediation: new eval or new training | `PROMOTION_REJECTED` → `PROMOTION_PENDING` (new request) or retrain |

**Outcome:** Failed governance visible in audit; production unchanged.

---

## Failed Deployment Path (Planned)

| Step | Action | State transition |
|------|--------|------------------|
| 1 | Approved version enters deploy | `PROMOTION_APPROVED` → `STAGING` → `DEPLOYING` |
| 2 | Deployment manager reports failure | `DEPLOYING` → `FAILED` |
| 3 | Audit records failure reason | — |
| 4 | Operator investigates; optional rollback if partial deploy | `FAILED` → retry `DEPLOYING` or `ROLLBACK` |

**Outcome:** Production not updated or restored; no silent partial state.

---

## Rollback Path (Planned)

| Step | Action | State transition |
|------|--------|------------------|
| 1 | Trigger: operator action or policy (SLO/drift severity) | `PRODUCTION` or `MONITORING` → `ROLLBACK` |
| 2 | Deployment manager applies previous `version_id` from deployment history | — |
| 3 | Rollback completes | → `PRODUCTION` / `MONITORING` (prior version) |
| 4 | Audit records `from_version`, `to_version`, `reason` | — |

**Outcome:** Known-good version restored; failed version remains in registry for analysis.

---

## Cross-Cutting Rules (Planned)

- Every path emits audit and transition events
- No path skips evaluation before `REGISTERED`
- No path reaches `PRODUCTION` without `PROMOTION_APPROVED`
- `FAILED` requires `failure_reason` and documented recovery

---

## Future Work

- Automated canary with metric-based promote/rollback
- Parallel shadow deployment without traffic shift
- Batch model families (multi-version coordinated promotion)
