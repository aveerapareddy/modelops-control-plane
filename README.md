# ModelOps Control Plane

**Status:** Architecture Phase  
**Implementation:** Not Started  
**Current Milestone:** Architecture Foundation

## Engineering Thesis

ML platforms fail when lifecycle state is implicit, promotion is ungoverned, and production behavior cannot be traced back to training artifacts and evaluation evidence. This project implements an explicit control plane that owns model lifecycle state, enforces valid transitions, records audit evidence, and coordinates training, registration, promotion, deployment, monitoring, drift response, retraining, and rollback.

The system is designed to be defensible in a Staff-level system design interview: every component has a bounded responsibility, every transition has an owner, and every production change has a reconstructable audit trail.

## What This System Is

A lifecycle governance platform for ML models. It coordinates:

- Training orchestration hooks
- Evaluation gating
- Model registration and versioning
- Promotion approval workflows
- Deployment coordination
- Production monitoring signals
- Drift detection and retraining triggers
- Validation and re-promotion
- Rollback execution

The control plane is the source of truth for **where a model version is in its lifecycle** and **what actions are permitted next**.

## What This System Is Not

- A tutorial CRUD application
- A metrics dashboard or BI tool
- An MLflow replacement
- A training framework
- A feature store
- A data labeling platform
- A fake demo with hardcoded alerts

## Planned Architecture

```
Operator / CI
     |
     v
+------------------+     +----------------------+
| Operator Console |     | Control Plane API    |
+------------------+     +----------+-----------+
                                    |
        +---------------------------+---------------------------+
        |                           |                           |
        v                           v                           v
+----------------+    +---------------------+    +------------------+
| Training       |    | Registry Service    |    | Deployment       |
| Orchestrator     |    |                     |    | Manager          |
+----------------+    +---------------------+    +------------------+
        |                           |                           |
        v                           v                           v
+----------------+    +---------------------+    +------------------+
| Monitoring     |    | Storage (runs,        |    | External targets |
| Service        |    | versions, audit)      |    | (K8s, endpoints) |
+----------------+    +---------------------+    +------------------+
```

Major services (planned):

| Service | Responsibility |
|---------|----------------|
| `control-plane` | Lifecycle state machine, transition validation, workflow coordination |
| `training-orchestrator` | Training job submission and run status |
| `registry-service` | Model version metadata and artifact references |
| `deployment-manager` | Staging and production deployment coordination |
| `monitoring-service` | Drift and health signal ingestion |
| `operator-console` | Operator workflows; not a vanity dashboard |

Shared packages (planned): `common-schemas`, `observability`.

## Repository Structure

```
docs/           Architecture, workflows, runbooks, examples
services/       Service boundaries (implementation not started)
packages/       Shared schemas and observability utilities
infra/          Database, container, K8s, and observability config
```

See [docs/overview/project-constitution.md](docs/overview/project-constitution.md) for engineering rules and [docs/architecture/system-overview.md](docs/architecture/system-overview.md) for system design.

## Implementation Status

| Area | Status |
|------|--------|
| Architecture documentation | In progress (Session 0) |
| Control plane API | Not started |
| Lifecycle state machine | Not started |
| Training orchestration | Not started |
| Registry service | Not started |
| Deployment manager | Not started |
| Monitoring / drift | Not started |
| Operator console | Not started |
| Infrastructure (DB, K8s, metrics) | Not started |

Nothing in this repository executes a lifecycle workflow today.

## Non-Goals (Session 0 and Early Phases)

- End-to-end runnable demo
- MLflow integration
- Airflow / Prefect workflows
- Kubernetes manifests with working deployments
- FastAPI service implementations
- Drift detection algorithms
- Synthetic metrics or alerts
- Production-grade auth (documented as planned)

## Next Milestone

**Architecture Foundation (current):** Complete normative docs, service boundaries, lifecycle contracts, and repository skeleton.

**Next:** Define API contracts and state machine implementation plan; scaffold control plane service with transition validation only (no external integrations).
