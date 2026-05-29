# System Overview

**Status:** Architecture Phase  
**Implementation Status:** Not Started

High-level design for ModelOps Control Plane.

---

## Implemented

Nothing.

---

## Planned

Architecture described in this document.

---

## Control Plane Responsibility

The control plane is the **authoritative lifecycle coordinator**. It:

- Maintains current lifecycle state per `(model_id, version_id)`
- Validates transition requests against the state machine
- Emits domain events for downstream services to react (deploy, monitor, etc.)
- Persists audit records for operator and automated actions
- Fails closed: invalid transitions return error without partial side effects

It does not store model weights, run training loops, or serve predictions.

---

## Major Services

```
                    +------------------+
                    |  Operator        |
                    |  Console         |
                    +--------+---------+
                             | HTTPS
                             v
                    +------------------+
                    |  Control Plane   |<------------------+
                    |  (state machine) |                   |
                    +--------+---------+                   |
                             |                             |
         +-------------------+-------------------+         |
         |                   |                   |         |
         v                   v                   v         |
+--------+----+    +--------+----+    +--------+----+    |
| Training    |    | Registry    |    | Deployment  |    |
| Orchestrator|    | Service     |    | Manager     |    |
+--------+----+    +--------+----+    +--------+----+    |
         |                   |                   |         |
         v                   v                   v         |
+--------+----+    +--------+----+    +--------+----+    |
| Training    |    | Artifact    |    | Runtime     |    |
| workers     |    | store refs  |    | targets     |    |
+-------------+    +-------------+    +-------------+    |
                                                         |
                    +------------------+                 |
                    | Monitoring       |-----------------+
                    | Service          |  drift / health events
                    +------------------+
```

| Service | Role |
|---------|------|
| **control-plane** | Lifecycle API, transition validation, audit append |
| **training-orchestrator** | Job spec submission, run status callbacks |
| **registry-service** | Version metadata, artifact URI, evaluation linkage |
| **deployment-manager** | Apply/remove deployments on staging and production targets |
| **monitoring-service** | Aggregate inference and data metrics; report drift |
| **operator-console** | Read lifecycle, submit approvals, trigger rollback |

---

## Lifecycle Flow (Planned)

```
DATA_PREPARED → TRAINING → EVALUATING → REGISTERED
    → PROMOTION_PENDING → PROMOTION_APPROVED | PROMOTION_REJECTED
    → STAGING → DEPLOYING → PRODUCTION → MONITORING
    → DRIFT_DETECTED → RETRAINING → VALIDATING → (promote cycle)
    → ROLLBACK (from PRODUCTION or MONITORING under policy)
    → FAILED (from multiple states with recovery paths)
```

Detailed semantics: [runtime-model.md](runtime-model.md).  
Workflow narratives: [model-lifecycle-workflow.md](../workflows/model-lifecycle-workflow.md).

---

## Trust Boundaries

| Boundary | Trust level | Notes |
|----------|-------------|-------|
| Operator → Console | Authenticated human or service account (planned) | Promotion and rollback are privileged |
| Console → Control plane | Internal network, mTLS or JWT (planned) | All mutations go through API |
| Control plane → DB | High trust | Sole writer of lifecycle state |
| Orchestrator → Control plane | Service identity | May only report run status and request allowed transitions |
| Deployment manager → Runtime | Scoped credentials | Deploy only approved artifact URIs from registry |
| Monitoring → Control plane | Service identity | May request `DRIFT_DETECTED`; cannot self-promote |
| External experiment tracker | Read/write via integration | Not source of lifecycle truth |

---

## Integration Boundaries

| System | Integration pattern | Control plane relationship |
|--------|---------------------|----------------------------|
| MLflow (optional) | Experiment/run IDs stored as references | Registry links `mlflow_run_id`; lifecycle state remains internal |
| Airflow / Prefect (optional) | Orchestrator delegates job execution | Training orchestrator boundary; see [tradeoffs.md](tradeoffs.md) |
| Object storage (S3/GCS/local) | Artifact URIs in registry | Immutable artifact references after register |
| Kubernetes / process runtime | Deployment manager applies manifests or process config | Production target external to control plane |
| Prometheus | Metrics scrape from all services | Observability package standardizes labels |

---

## Intentionally Out of Scope

- Model training algorithms and hyperparameter search
- Feature engineering pipelines
- Online feature serving
- A/B test analysis platform (may link evaluation results, not build full experimentation UI)
- Enterprise data catalog
- Billing and chargeback (future work)

---

## Future Work

- Event bus (NATS/Kafka) for async domain events instead of synchronous callbacks only
- Read replicas for lifecycle query API
- GraphQL or unified query layer for operator console (evaluate after REST baseline)
