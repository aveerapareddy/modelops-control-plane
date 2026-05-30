# Transition Rules

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

Explicit allowed and rejected transitions. Control plane must reject any transition not listed as allowed.

---

## Implemented

No runtime behavior is implemented. No transition validation code exists.

---

## Planned

Transition API validates `(from_state, to_state, actor, evidence)` against this document.

---

## Future Work

- Machine-readable transition matrix export for contract tests
- Policy engine overrides with `OVERRIDE` audit event type

---

## Transition Record Schema (Conceptual)

Each allowed transition produces:

| Field | Required |
|-------|----------|
| `from_state` | Yes |
| `to_state` | Yes |
| `trigger` | Yes — event or operator action name |
| `actor` | Yes — `user:*`, `service:*`, `policy:*` |
| `evidence` | Per table — URIs, metric IDs, approval IDs |
| `event_emitted` | Yes — see [event-model.md](event-model.md) |
| `failure_path` | If transition fails mid-flight, target failure state |

---

## Allowed Transitions

### Data and training

| From | To | Trigger | Actor | Required Evidence | Event Emitted | Failure Path |
|------|-----|---------|-------|-------------------|---------------|--------------|
| `DATA_PREPARED` | `TRAINING` | `submit_training` | `service:training-orchestrator` | `run_id`, `config_hash`, data manifest URI | `TrainingStarted` | — |
| `TRAINING` | `EVALUATING` | `training_succeeded` | `service:training-orchestrator` | `run_id`, artifact URI (pre-register) | `TrainingCompleted` | — |
| `TRAINING` | `TRAINING_FAILED` | `training_failed` | `service:training-orchestrator` | `run_id`, `reason`, error code | `TrainingFailed` | — |
| `TRAINING_FAILED` | `TRAINING` | `retry_training` | `user:operator` or `policy:retry` | New `run_id`, retry `reason` | `TrainingStarted` | — |
| `TRAINING_FAILED` | `FAILED` | `abandon_version` | `user:operator` | `reason` | `LifecycleFailed` | — |

### Evaluation and registration

| From | To | Trigger | Actor | Required Evidence | Event Emitted | Failure Path |
|------|-----|---------|-------|-------------------|---------------|--------------|
| `EVALUATING` | `REGISTERED` | `eval_gate_passed` | `service:control-plane` | `evaluation_id`, metrics JSON, `passed=true` | `EvaluationCompleted`, `ModelRegistered` | — |
| `EVALUATING` | `EVALUATION_FAILED` | `eval_gate_failed` | `service:control-plane` | `evaluation_id`, metrics, `passed=false` | `EvaluationFailed` | — |
| `EVALUATION_FAILED` | `TRAINING` | `retrain_after_eval_fail` | `user:operator` | `reason`, new `config_hash` optional | `TrainingStarted` | — |
| `EVALUATION_FAILED` | `FAILED` | `abandon_version` | `user:operator` | `reason` | `LifecycleFailed` | — |

### Promotion

| From | To | Trigger | Actor | Required Evidence | Event Emitted | Failure Path |
|------|-----|---------|-------|-------------------|---------------|--------------|
| `REGISTERED` | `PROMOTION_PENDING` | `request_promotion` | `user:operator` or `policy:auto` | `promotion_request_id` | `PromotionRequested` | — |
| `PROMOTION_PENDING` | `PROMOTION_APPROVED` | `approve_promotion` | `user:approver` | `decision_id`, policy check results | `PromotionApproved` | — |
| `PROMOTION_PENDING` | `PROMOTION_REJECTED` | `reject_promotion` | `user:approver` | `decision_id`, `comment` | `PromotionRejected` | — |
| `PROMOTION_PENDING` | `FAILED` | `promotion_workflow_error` | `service:control-plane` | `reason` | `LifecycleFailed` | — |
| `PROMOTION_REJECTED` | `PROMOTION_PENDING` | `request_promotion` | `user:operator` | New request after remediation | `PromotionRequested` | — |
| `PROMOTION_APPROVED` | `STAGING` | `staging_deploy_succeeded` | `service:deployment-manager` | `deployment_id`, staging target ref | `StagingStarted` | `DEPLOYMENT_FAILED` |

### Deployment

| From | To | Trigger | Actor | Required Evidence | Event Emitted | Failure Path |
|------|-----|---------|-------|-------------------|---------------|--------------|
| `STAGING` | `DEPLOYING` | `start_production_cutover` | `service:deployment-manager` | `deployment_id`, health check pass | `DeploymentStarted` | `DEPLOYMENT_FAILED` |
| `DEPLOYING` | `PRODUCTION` | `cutover_succeeded` | `service:deployment-manager` | `deployment_id`, health validation URI | `DeploymentSucceeded` | `DEPLOYMENT_FAILED` |
| `DEPLOYING` | `DEPLOYMENT_FAILED` | `cutover_failed` | `service:deployment-manager` | `deployment_id`, `reason` | `DeploymentFailed` | — |
| `DEPLOYMENT_FAILED` | `DEPLOYING` | `retry_deploy` | `user:operator` or `policy:retry` | `deployment_id`, retry `reason` | `DeploymentStarted` | — |
| `DEPLOYMENT_FAILED` | `STAGING` | `revert_to_staging` | `service:deployment-manager` | Slot state proof | `StagingStarted` | `FAILED` |
| `DEPLOYMENT_FAILED` | `FAILED` | `deploy_exhausted` | `service:control-plane` | `reason` | `LifecycleFailed` | — |
| `PRODUCTION` | `MONITORING` | `monitoring_armed` | `service:monitoring-service` | Monitor config ref | `MonitoringStarted` | — |

### Monitoring, drift, challenger

| From | To | Trigger | Actor | Required Evidence | Event Emitted | Failure Path |
|------|-----|---------|-------|-------------------|---------------|--------------|
| `MONITORING` | `DRIFT_DETECTED` | `drift_threshold_breached` | `service:monitoring-service` | `drift_event_id`, metric, threshold | `DriftDetected` | — |
| `DRIFT_DETECTED` | `RETRAINING` | `start_retrain` | `policy:drift` or `user:operator` | `parent_version_id`, `run_id` | `RetrainingStarted` | — |
| `RETRAINING` | `VALIDATING` | `challenger_ready` | `service:training-orchestrator` | `run_id`, eval metrics | `RetrainingCompleted`, `ValidationStarted` | `TRAINING_FAILED` |
| `RETRAINING` | `TRAINING_FAILED` | `retrain_failed` | `service:training-orchestrator` | `reason` | `TrainingFailed` | — |
| `VALIDATING` | `PROMOTION_PENDING` | `validation_passed` | `service:control-plane` | Baseline comparison report | `ValidationCompleted`, `PromotionRequested` | — |
| `VALIDATING` | `VALIDATION_FAILED` | `validation_failed` | `service:control-plane` | Baseline comparison report | `ValidationFailed` | — |
| `VALIDATION_FAILED` | `RETRAINING` | `retry_retrain` | `user:operator` | `reason` | `RetrainingStarted` | — |
| `VALIDATION_FAILED` | `FAILED` | `abandon_challenger` | `user:operator` | `reason` | `LifecycleFailed` | — |

### Rollback

| From | To | Trigger | Actor | Required Evidence | Event Emitted | Failure Path |
|------|-----|---------|-------|-------------------|---------------|--------------|
| `PRODUCTION` | `ROLLBACK_PENDING` | `request_rollback` | `user:operator` or `policy:rollback` | `rollback_id`, target `version_id`, `reason` | `RollbackRequested` | — |
| `MONITORING` | `ROLLBACK_PENDING` | `request_rollback` | `user:operator` or `policy:rollback` | Same as above | `RollbackRequested` | — |
| `ROLLBACK_PENDING` | `ROLLBACK_IN_PROGRESS` | `execute_rollback` | `service:deployment-manager` | Prior prod `deployment_id` | `RollbackStarted` | — |
| `ROLLBACK_IN_PROGRESS` | `ROLLED_BACK` | `rollback_succeeded` | `service:deployment-manager` | Post-rollback health URI | `RollbackCompleted` | — |
| `ROLLBACK_IN_PROGRESS` | `FAILED` | `rollback_failed` | `service:deployment-manager` | `reason` | `RollbackFailed` | Manual remediation |
| `ROLLED_BACK` | `MONITORING` | `resume_monitoring` | `service:monitoring-service` | Restored version monitor ref | `MonitoringStarted` | — |

### Global failure recovery

| From | To | Trigger | Actor | Required Evidence | Event Emitted | Failure Path |
|------|-----|---------|-------|-------------------|---------------|--------------|
| `FAILED` | `DATA_PREPARED` | `remediate_reset` | `user:operator` | Remediation plan URI | `DataPrepared` (new version) | — |
| `FAILED` | `TRAINING` | `remediate_retrain` | `user:operator` | `reason`, new `run_id` | `TrainingStarted` | — |
| `FAILED` | `ROLLBACK_PENDING` | `remediate_rollback` | `user:operator` | Eligibility proof | `RollbackRequested` | — |

---

## Invalid Transitions (Must Reject)

| From | To | Rejection reason |
|------|-----|------------------|
| `REGISTERED` | `PRODUCTION` | Skips promotion and deployment; no `deployment_id` |
| `TRAINING` | `PRODUCTION` | No artifact, eval, or approval |
| `PROMOTION_REJECTED` | `DEPLOYING` | Governance denial must be overturned via new approval |
| `DRIFT_DETECTED` | `PRODUCTION` | Skips retrain, validate, promote loop |
| `ROLLED_BACK` | `DEPLOYING` | New deploy requires `PROMOTION_APPROVED` on target version |
| `FAILED` | `PRODUCTION` | No evidence path; must remediate through legal chain |
| `EVALUATING` | `PRODUCTION` | Eval and promotion gates required |
| `STAGING` | `PRODUCTION` | Must pass through `DEPLOYING` for cutover audit |
| `MONITORING` | `PRODUCTION` | Production already active; use deploy workflow for new version |
| `VALIDATING` | `PRODUCTION` | Challenger must promote then deploy |
| `PROMOTION_PENDING` | `STAGING` | Approval required |
| `DATA_PREPARED` | `REGISTERED` | Training and evaluation required |

### Why invalid transitions matter

1. **Audit reconstructability** — Interview and incident questions require a complete chain from train to prod.
2. **Blast radius** — Skipping gates deploys unapproved or unevaluated artifacts.
3. **Runtime truth** — `PRODUCTION` implies deployment manager owns the slot; illegal transitions desync state from infrastructure.
4. **Operator trust** — Rejected transitions return stable errors (`INVALID_TRANSITION`), not partial side effects.

---

## Idempotency

Duplicate `idempotency_key` for the same `(version_id, from_state, to_state)` returns the original transition result without re-emitting side effects.

---

## Related Documents

- [lifecycle-state-machine.md](lifecycle-state-machine.md)
- [event-model.md](event-model.md)
- [../workflows/](../workflows/) — narrative workflows per domain
