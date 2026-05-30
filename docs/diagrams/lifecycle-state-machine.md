# Lifecycle State Machine Diagram

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

Source-controlled diagram for Session 1 state machine. Canonical definitions: [lifecycle-state-machine.md](../architecture/lifecycle-state-machine.md), [transition-rules.md](../architecture/transition-rules.md).

---

## Implemented

No runtime behavior is implemented.

---

## Planned

Diagram generated from enforced transition matrix at implementation time (optional).

---

## Happy Path

```mermaid
stateDiagram-v2
    direction LR

    [*] --> DATA_PREPARED
    DATA_PREPARED --> TRAINING
    TRAINING --> EVALUATING
    EVALUATING --> REGISTERED
    REGISTERED --> PROMOTION_PENDING
    PROMOTION_PENDING --> PROMOTION_APPROVED
    PROMOTION_APPROVED --> STAGING
    STAGING --> DEPLOYING
    DEPLOYING --> PRODUCTION
    PRODUCTION --> MONITORING
```

---

## Failure Paths (Training / Eval / Deploy)

```mermaid
stateDiagram-v2
    direction TB

    TRAINING --> TRAINING_FAILED
    TRAINING_FAILED --> TRAINING : retry
    TRAINING_FAILED --> FAILED : abandon

    EVALUATING --> EVALUATION_FAILED
    EVALUATION_FAILED --> TRAINING : retrain
    EVALUATION_FAILED --> FAILED : abandon

    DEPLOYING --> DEPLOYMENT_FAILED
    DEPLOYMENT_FAILED --> DEPLOYING : retry
    DEPLOYMENT_FAILED --> FAILED : exhausted

    PROMOTION_PENDING --> PROMOTION_REJECTED
    PROMOTION_REJECTED --> PROMOTION_PENDING : new request
```

---

## Drift and Retraining Path

```mermaid
stateDiagram-v2
    direction TB

    MONITORING --> DRIFT_DETECTED
    DRIFT_DETECTED --> RETRAINING
    RETRAINING --> VALIDATING
    VALIDATING --> PROMOTION_PENDING
    PROMOTION_PENDING --> PROMOTION_APPROVED
    PROMOTION_APPROVED --> STAGING

    RETRAINING --> TRAINING_FAILED
    VALIDATING --> VALIDATION_FAILED
    VALIDATION_FAILED --> RETRAINING : retry
```

---

## Rollback Path

```mermaid
stateDiagram-v2
    direction TB

    PRODUCTION --> ROLLBACK_PENDING
    MONITORING --> ROLLBACK_PENDING
    ROLLBACK_PENDING --> ROLLBACK_IN_PROGRESS
    ROLLBACK_IN_PROGRESS --> ROLLED_BACK
    ROLLED_BACK --> MONITORING
    ROLLBACK_IN_PROGRESS --> FAILED
```

---

## Combined Overview

```mermaid
flowchart TB
    subgraph train["Train / Register"]
        DP[DATA_PREPARED] --> TR[TRAINING]
        TR --> EV[EVALUATING]
        EV --> REG[REGISTERED]
        TR --> TF[TRAINING_FAILED]
        TF --> TR
        EV --> EF[EVALUATION_FAILED]
        EF --> TR
    end

    subgraph promote["Promote / Deploy"]
        REG --> PP[PROMOTION_PENDING]
        PP --> PA[PROMOTION_APPROVED]
        PP --> PR[PROMOTION_REJECTED]
        PR --> PP
        PA --> ST[STAGING]
        ST --> DEP[DEPLOYING]
        DEP --> PROD[PRODUCTION]
        DEP --> DF[DEPLOYMENT_FAILED]
        DF --> DEP
        PROD --> MON[MONITORING]
    end

    subgraph drift["Drift / Retrain"]
        MON --> DD[DRIFT_DETECTED]
        DD --> RT[RETRAINING]
        RT --> VAL[VALIDATING]
        VAL --> PP
        VAL --> VF[VALIDATION_FAILED]
        VF --> RT
    end

    subgraph rb["Rollback"]
        PROD --> RP[ROLLBACK_PENDING]
        MON --> RP
        RP --> RI[ROLLBACK_IN_PROGRESS]
        RI --> RBK[ROLLED_BACK]
        RBK --> MON
        RI --> FAIL[FAILED]
    end

    train --> promote
    promote --> drift
```

---

## Forbidden Edges (Not Shown)

These must be rejected by the control plane:

- `REGISTERED` → `PRODUCTION`
- `TRAINING` → `PRODUCTION`
- `DRIFT_DETECTED` → `PRODUCTION`
- `PROMOTION_REJECTED` → `DEPLOYING`
- `ROLLED_BACK` → `DEPLOYING` (without new approval)
- `FAILED` → `PRODUCTION`

See [transition-rules.md](../architecture/transition-rules.md).
