# Drift Retraining Workflow

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

---

## Implemented

No runtime behavior is implemented.

---

## Planned

Closed loop: drift detection must trigger state change and retraining, not logging only.

---

## Future Work

- Auto-promotion after validation under strict policy
- Feature store integration for drift features

---

## Purpose

Respond to production drift by training a challenger, validating against baseline, and re-entering promotion.

---

## Actors

| Actor | Role |
|-------|------|
| `service:monitoring-service` | Detect drift, emit evidence |
| `policy:drift` | Mandate `RETRAINING` transition |
| `user:operator` | Manual retrain trigger override |
| `service:training-orchestrator` | Challenger training |
| `service:control-plane` | Validation gate, promotion entry |

---

## Inputs

- Production `model_version_id` under `MONITORING`
- Drift metrics (PSI, KS, performance decay — methodology documented)
- Policy drift thresholds

---

## Outputs

- `drift_event_id`
- Challenger `run_id`, `model_version_id`
- Validation report vs baseline
- State path to `PROMOTION_PENDING` or failure states

---

## Required Evidence

| Step | Evidence |
|------|----------|
| Drift | `metric_name`, `metric_value`, `threshold`, feature group, methodology URI |
| Retrain | `parent_version_id`, `run_id` |
| Validate | Baseline comparison URI, `passed` flag |

**Insufficient:** Log line only, dashboard annotation without `DriftDetected` event, hardcoded `drift=true`.

---

## State Transitions

```
MONITORING → DRIFT_DETECTED → RETRAINING → VALIDATING → PROMOTION_PENDING
                ↓                              ↓
           (no prod skip)              VALIDATION_FAILED → RETRAINING
```

---

## Events Emitted

`DriftDetected` → `RetrainingStarted` → `RetrainingCompleted` → `ValidationStarted` → `ValidationCompleted` | `ValidationFailed` → `PromotionRequested`

---

## Failure Modes

| Failure | State | Action |
|---------|-------|--------|
| False positive drift | `DRIFT_DETECTED` | Operator suppress + audit (future) |
| Retrain fail | `TRAINING_FAILED` / `FAILED` | Retry or abandon |
| Validation fail | `VALIDATION_FAILED` | Retry retrain |
| Promote without validate | **Rejected** | `DRIFT_DETECTED` → `PRODUCTION` invalid |

---

## Recovery Paths

| From | Action |
|------|--------|
| `VALIDATION_FAILED` | `retry_retrain` |
| Challenger abandon | `FAILED` or remain on production version |
| Production stable, drift resolved | Stay `MONITORING` on current version if retrain cancelled (operator, audited) |

---

## Promotion Loop

Challenger passing validation enters same [promotion-workflow.md](promotion-workflow.md) as initial register path, with `drift_requires_manual_promote` per policy.

---

## Current Implementation Status

Documentation only. No monitoring service, no drift algorithms, no retrain automation.
