# Lifecycle State Machine

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

Canonical definition of lifecycle states for a single `model_version_id`. Session 0 [runtime-model.md](runtime-model.md) introduced preliminary states; this document is authoritative for Session 1 and supersedes ambiguous Session 0 rollback semantics.

---

## Implemented

No runtime behavior is implemented. No state is persisted. No transition API exists.

---

## Planned

Control plane enforces this state machine per model version with optimistic locking on `state_version`.

---

## Future Work

- `SHADOW` sub-state for pre-promotion inference comparison
- `CANARY` deploy sub-state (see [deployment-strategy.md](deployment-strategy.md))
- Model-family-level aggregate state separate from version state

---

## Design Questions This Document Answers

| Question | Answer location |
|----------|-----------------|
| What state is a model in? | State definitions below |
| Who moved it there? | Owner per state; transitions in [transition-rules.md](transition-rules.md) |
| Why was it moved? | `reason` + Required Evidence on transitions |
| What event was emitted? | [event-model.md](event-model.md) |
| What transitions are legal? | [transition-rules.md](transition-rules.md) |
| What triggers retraining? | `DRIFT_DETECTED` → `RETRAINING` (policy or operator) |
| What triggers rollback? | Operator or automated policy from `PRODUCTION` / `MONITORING` |
| What approval gates exist? | `PROMOTION_PENDING` → `PROMOTION_APPROVED` |
| What audit evidence exists? | Transition evidence + event retention per event model |
| What happens when something fails? | Failure states and recovery paths below |

---

## State Taxonomy

| Category | States |
|----------|--------|
| **Pre-training** | `DATA_PREPARED` |
| **Training / eval** | `TRAINING`, `TRAINING_FAILED`, `EVALUATING`, `EVALUATION_FAILED` |
| **Registry / promotion** | `REGISTERED`, `PROMOTION_PENDING`, `PROMOTION_APPROVED`, `PROMOTION_REJECTED` |
| **Deploy** | `STAGING`, `DEPLOYING`, `DEPLOYMENT_FAILED`, `PRODUCTION` |
| **Operate** | `MONITORING`, `DRIFT_DETECTED` |
| **Challenger loop** | `RETRAINING`, `VALIDATING`, `VALIDATION_FAILED` |
| **Rollback** | `ROLLBACK_PENDING`, `ROLLBACK_IN_PROGRESS`, `ROLLED_BACK` |
| **Global failure** | `FAILED` |

### Terminal and quasi-terminal states

| State | Terminal? | Notes |
|-------|-----------|-------|
| `PROMOTION_REJECTED` | Quasi-terminal | Stable until new promotion request; no deploy |
| `ROLLED_BACK` | Quasi-terminal | Stable post-rollback; transitions to `MONITORING` on prior version |
| `FAILED` | Recoverable terminal | Requires explicit remediation transition; not a silent dead end |

No state is fully immutable except by operator policy override (audited).

### Recoverable failure states

`TRAINING_FAILED`, `EVALUATION_FAILED`, `DEPLOYMENT_FAILED`, `VALIDATION_FAILED` — scoped failures with defined retry transitions. Evidence and `reason` required.

### Non-recoverable failure (without remediation)

`FAILED` — used when invariant violated, rollback failed irrecoverably, or system cannot determine safe next state. Recovery only via explicit audited transitions to `DATA_PREPARED`, `TRAINING`, or `ROLLBACK_PENDING` per [transition-rules.md](transition-rules.md).

---

## Forbidden Shortcuts (Normative)

### Why `REGISTERED` → `PRODUCTION` is forbidden

Registration records artifact and evaluation evidence only. Production requires:

1. Promotion policy satisfaction and approval (`PROMOTION_APPROVED`)
2. Staging validation and blue/green deploy per [deployment-strategy.md](deployment-strategy.md)
3. Deployment history and health validation

Skipping promotion or deploy removes audit reconstructability and bypasses governance.

### Why `PRODUCTION` is reached only through deployment workflow

`PRODUCTION` means an active production slot (blue or green) serves traffic with a recorded `deployment_id`. Only `DEPLOYING` → `PRODUCTION` after deployment manager reports success and health validation passes. Direct assignment would decouple runtime truth from control plane state.

---

## State Definitions

### DATA_PREPARED

| Field | Value |
|-------|-------|
| **Purpose** | Training inputs and config validated; version eligible to start training |
| **Entry** | New version created; data manifest and `config_hash` verified |
| **Exit** | Training job submitted |
| **Owner** | Training orchestrator (submit); control plane (state) |
| **Next** | `TRAINING` |

### TRAINING

| Field | Value |
|-------|-------|
| **Purpose** | Training job executing |
| **Entry** | Orchestrator acknowledged job start |
| **Exit** | Run completes or fails |
| **Owner** | Training orchestrator |
| **Next** | `EVALUATING`, `TRAINING_FAILED` |

### TRAINING_FAILED

| Field | Value |
|-------|-------|
| **Purpose** | Training run failed; artifact not produced |
| **Entry** | Orchestrator reported failure |
| **Exit** | Retry or abandon |
| **Owner** | Training orchestrator (report); operator (retry decision) |
| **Next** | `TRAINING`, `FAILED` |

### EVALUATING

| Field | Value |
|-------|-------|
| **Purpose** | Metrics computed; promotion gates evaluated pre-registration |
| **Entry** | Training succeeded |
| **Exit** | Eval gate pass or fail |
| **Owner** | Control plane (gate); training orchestrator (metric upload) |
| **Next** | `REGISTERED`, `EVALUATION_FAILED` |

### EVALUATION_FAILED

| Field | Value |
|-------|-------|
| **Purpose** | Evaluation gate failed; version not registry-eligible |
| **Entry** | Gate failure recorded with metrics |
| **Exit** | Retrain or mark failed |
| **Owner** | Control plane |
| **Next** | `TRAINING`, `FAILED` |

### REGISTERED

| Field | Value |
|-------|-------|
| **Purpose** | Version in registry with immutable `artifact_uri` and eval record |
| **Entry** | Eval gate passed; registry write confirmed |
| **Exit** | Promotion requested |
| **Owner** | Registry service (metadata); control plane (state) |
| **Next** | `PROMOTION_PENDING` |

### PROMOTION_PENDING

| Field | Value |
|-------|-------|
| **Purpose** | Awaiting policy checks and/or human approval |
| **Entry** | Promotion request submitted |
| **Exit** | Approved, rejected, or failed |
| **Owner** | Control plane; approver role |
| **Next** | `PROMOTION_APPROVED`, `PROMOTION_REJECTED`, `FAILED` |

### PROMOTION_APPROVED

| Field | Value |
|-------|-------|
| **Purpose** | Authorized to enter deploy track |
| **Entry** | Policy + approval satisfied |
| **Exit** | Staging deployment initiated |
| **Owner** | Control plane |
| **Next** | `STAGING` |

### PROMOTION_REJECTED

| Field | Value |
|-------|-------|
| **Purpose** | Promotion denied; deploy blocked |
| **Entry** | Approver or policy rejection |
| **Exit** | New promotion request after remediation |
| **Owner** | Approver / control plane |
| **Next** | `PROMOTION_PENDING` |

### STAGING

| Field | Value |
|-------|-------|
| **Purpose** | Candidate deployed to non-production (green slot or staging target) |
| **Entry** | Staging deploy succeeded |
| **Exit** | Production cutover or failure |
| **Owner** | Deployment manager |
| **Next** | `DEPLOYING`, `DEPLOYMENT_FAILED` |

### DEPLOYING

| Field | Value |
|-------|-------|
| **Purpose** | Production cutover in progress (blue/green switch) |
| **Entry** | Cutover initiated from staging |
| **Exit** | Success, failure, or rollback interrupt |
| **Owner** | Deployment manager |
| **Next** | `PRODUCTION`, `DEPLOYMENT_FAILED`, `FAILED` |

### DEPLOYMENT_FAILED

| Field | Value |
|-------|-------|
| **Purpose** | Deploy or health validation failed |
| **Entry** | Deployment manager failure callback |
| **Exit** | Retry deploy or escalate |
| **Owner** | Deployment manager |
| **Next** | `DEPLOYING`, `STAGING`, `FAILED` |

### PRODUCTION

| Field | Value |
|-------|-------|
| **Purpose** | Version holds active production slot after successful cutover |
| **Entry** | `DEPLOYING` success + health validation |
| **Exit** | Handoff to monitoring or rollback request |
| **Owner** | Deployment manager (runtime); control plane (state) |
| **Next** | `MONITORING`, `ROLLBACK_PENDING` |

### MONITORING

| Field | Value |
|-------|-------|
| **Purpose** | Production version under active drift and health observation |
| **Entry** | Stable production confirmed |
| **Exit** | Drift detected or rollback |
| **Owner** | Monitoring service (signals); control plane (state) |
| **Next** | `DRIFT_DETECTED`, `ROLLBACK_PENDING` |

### DRIFT_DETECTED

| Field | Value |
|-------|-------|
| **Purpose** | Drift evidence recorded; retraining required by policy |
| **Entry** | Monitoring threshold breach with evidence |
| **Exit** | Retrain initiated |
| **Owner** | Monitoring service (detect); control plane (transition) |
| **Next** | `RETRAINING` |

### RETRAINING

| Field | Value |
|-------|-------|
| **Purpose** | Challenger training run linked to `parent_version_id` |
| **Entry** | Retrain triggered from drift or operator |
| **Exit** | Train+eval complete for challenger |
| **Owner** | Training orchestrator |
| **Next** | `VALIDATING`, `TRAINING_FAILED`, `FAILED` |

### VALIDATING

| Field | Value |
|-------|-------|
| **Purpose** | Challenger compared to production baseline |
| **Entry** | Challenger eval complete |
| **Exit** | Validation pass or fail |
| **Owner** | Control plane |
| **Next** | `PROMOTION_PENDING`, `VALIDATION_FAILED` |

### VALIDATION_FAILED

| Field | Value |
|-------|-------|
| **Purpose** | Challenger did not beat baseline under policy |
| **Entry** | Validation gate failure |
| **Exit** | Retry retrain or fail |
| **Owner** | Control plane |
| **Next** | `RETRAINING`, `FAILED` |

### ROLLBACK_PENDING

| Field | Value |
|-------|-------|
| **Purpose** | Rollback approved; awaiting execution |
| **Entry** | Operator or policy requests rollback |
| **Exit** | Execution starts |
| **Owner** | Control plane; operator |
| **Next** | `ROLLBACK_IN_PROGRESS` |

### ROLLBACK_IN_PROGRESS

| Field | Value |
|-------|-------|
| **Purpose** | Prior production version being restored |
| **Entry** | Deployment manager started rollback deploy |
| **Exit** | Success or failure |
| **Owner** | Deployment manager |
| **Next** | `ROLLED_BACK`, `FAILED` |

### ROLLED_BACK

| Field | Value |
|-------|-------|
| **Purpose** | Known-good version active; drift incident contained |
| **Entry** | Rollback deploy + validation succeeded |
| **Exit** | Resume monitoring on restored version |
| **Owner** | Deployment manager; control plane |
| **Next** | `MONITORING` |

### FAILED

| Field | Value |
|-------|-------|
| **Purpose** | Unrecoverable without explicit remediation plan |
| **Entry** | Invariant violation, exhausted retries, or rollback failure |
| **Exit** | Audited remediation only |
| **Owner** | Control plane |
| **Next** | `DATA_PREPARED`, `TRAINING`, `ROLLBACK_PENDING` (case-specific; see transition rules) |

---

## Related Documents

- [transition-rules.md](transition-rules.md) — explicit transition table
- [event-model.md](event-model.md) — events per transition
- [deployment-strategy.md](deployment-strategy.md) — blue/green semantics
- [promotion-policy-model.md](promotion-policy-model.md) — gates before `PROMOTION_APPROVED`
- [../diagrams/lifecycle-state-machine.md](../diagrams/lifecycle-state-machine.md) — visual diagram
