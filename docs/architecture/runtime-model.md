# Runtime Model

**Status:** Architecture Phase  
**Implementation Status:** Not Started

Planned lifecycle state machine semantics for model versions.

**Session 1 update:** Canonical state machine, transitions, and events are defined in [lifecycle-state-machine.md](lifecycle-state-machine.md), [transition-rules.md](transition-rules.md), and [event-model.md](event-model.md). This document remains Session 0 context; prefer Session 1 docs for new work.

---

## Implemented

Nothing. States and transitions below are **planned only**.

---

## Planned

Full state machine with transition validation, persistence, and event emission.

---

## State Definitions

| State | Meaning |
|-------|---------|
| `DATA_PREPARED` | Training dataset and config validated; ready to start training |
| `TRAINING` | Training job active |
| `EVALUATING` | Training complete; evaluation metrics computed or in progress |
| `REGISTERED` | Version recorded in registry with artifact reference |
| `PROMOTION_PENDING` | Promotion requested; awaiting approval |
| `PROMOTION_APPROVED` | Approved for deployment track |
| `PROMOTION_REJECTED` | Promotion denied; version blocked from deploy until new request |
| `STAGING` | Deployed or ready for deploy to non-production target |
| `DEPLOYING` | Deployment in progress |
| `PRODUCTION` | Serving production traffic (or designated prod slot) |
| `MONITORING` | Production deployment under active monitoring |
| `DRIFT_DETECTED` | Monitoring reported drift above threshold |
| `RETRAINING` | New training run in progress (challenger lineage) |
| `VALIDATING` | Challenger evaluated against production baseline |
| `ROLLBACK` | Rollback in progress or completed rollback state (see note) |
| `FAILED` | Terminal failure with `failure_reason`; recovery via explicit transitions |

**Note on `ROLLBACK`:** May be modeled as a transient state during rollback execution, with stable post-rollback state returning to `PRODUCTION` (previous version) or `MONITORING`. Final representation TBD at implementation; behavior must preserve audit trail.

---

## Transition Rules (Planned)

Invalid transitions **must eventually be rejected** by the control plane API with a stable error code (e.g., `INVALID_TRANSITION`). No downstream service may bypass validation.

### Primary happy-path edges

```
DATA_PREPARED      → TRAINING
TRAINING           → EVALUATING | FAILED
EVALUATING         → REGISTERED | FAILED
REGISTERED         → PROMOTION_PENDING
PROMOTION_PENDING  → PROMOTION_APPROVED | PROMOTION_REJECTED
PROMOTION_APPROVED → STAGING
STAGING            → DEPLOYING
DEPLOYING          → PRODUCTION | FAILED
PRODUCTION         → MONITORING
MONITORING         → DRIFT_DETECTED | ROLLBACK (policy/operator)
DRIFT_DETECTED     → RETRAINING
RETRAINING         → VALIDATING | FAILED
VALIDATING         → PROMOTION_PENDING | FAILED
```

### Rejection and recovery edges

```
PROMOTION_REJECTED → PROMOTION_PENDING   (new request after remediation)
PROMOTION_REJECTED → (no direct path to PRODUCTION)
FAILED             → TRAINING | DATA_PREPARED  (explicit retry; audit required)
ROLLBACK           → PRODUCTION | MONITORING   (prior version active)
```

### Forbidden examples (non-exhaustive)

- `REGISTERED` → `PRODUCTION` (skips promotion and deploy)
- `PROMOTION_REJECTED` → `DEPLOYING`
- `TRAINING` → `REGISTERED` (skips evaluation)
- `DRIFT_DETECTED` → `PRODUCTION` (skips retrain/validate)

---

## Transition Ownership

| Transition trigger | Actor |
|--------------------|-------|
| `DATA_PREPARED` → `TRAINING` | Orchestrator or operator |
| `TRAINING` → `EVALUATING` | Orchestrator callback |
| `EVALUATING` → `REGISTERED` | Control plane after eval gate pass |
| `REGISTERED` → `PROMOTION_PENDING` | Operator or automated policy |
| `PROMOTION_PENDING` → `PROMOTION_*` | Approver role |
| `PROMOTION_APPROVED` → `STAGING` | Deployment manager request |
| `DEPLOYING` → `PRODUCTION` | Deployment manager callback |
| `MONITORING` → `DRIFT_DETECTED` | Monitoring service |
| `DRIFT_DETECTED` → `RETRAINING` | Control plane policy or operator |
| `*` → `ROLLBACK` | Operator or automated rollback policy |
| `*` → `FAILED` | Any service with documented failure contract |

---

## Idempotency and Concurrency (Planned)

- Transition requests carry `idempotency_key`; duplicate keys return same result without duplicate side effects
- Optimistic locking on `version_id` + `state_version` prevents lost updates
- Concurrent promotion and deploy requests: second request fails with conflict

---

## Events (Planned)

Each successful transition emits `LifecycleTransitionEvent` (schema in `packages/common-schemas`, not yet defined) containing:

- `model_id`, `version_id`
- `from_state`, `to_state`
- `actor`, `timestamp`, `correlation_id`
- `metadata` (optional: eval scores, approver comment, deploy target)

---

## Implementation Status

| Capability | Status |
|------------|--------|
| State enum | Not started |
| Transition matrix | Not started |
| Persistence | Not started |
| API | Not started |
| Rejection of invalid transitions | Not started |

---

## Future Work

- Sub-states for canary deploy (`DEPLOYING` → `CANARY` → `PRODUCTION`)
- Parallel challenger in `SHADOW` without promotion
- Global "model family" state separate from per-version state
