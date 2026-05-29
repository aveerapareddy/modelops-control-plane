# Project Constitution

**Status:** Architecture Phase  
**Implementation Status:** Not Started

Normative engineering rules for ModelOps Control Plane. Deviations require explicit documentation in [tradeoffs.md](../architecture/tradeoffs.md).

---

## Implemented

Nothing. This document defines rules only.

---

## Planned

All principles below govern design and implementation once coding begins.

---

## What the System Is

A **control plane** for ML model lifecycle governance. It:

- Owns authoritative lifecycle state per model version
- Validates state transitions before side effects execute
- Records audit evidence for promotion, deployment, drift, retraining, and rollback
- Coordinates specialized services (training, registry, deployment, monitoring)
- Exposes operator workflows with explicit approval gates

The control plane does not train models, serve inference, or compute drift statistics. It governs **when** those activities occur and **what state** results.

---

## What the System Is Not

| Not | Reason |
|-----|--------|
| CRUD metadata store | Lifecycle semantics require a state machine, not arbitrary updates |
| Dashboard-first product | UI supports operations; metrics and state drive decisions |
| MLflow clone | MLflow is an integration boundary for experiment tracking, not the platform core |
| Training framework | Training runs in external workers; orchestrator submits and tracks |
| Inference server | Deployment manager targets external runtimes |
| Alert generator with fake thresholds | Drift signals must originate from real monitoring pipelines or documented stubs with clear limitations |

---

## Architecture Principles

1. **Explicit lifecycle state.** Every model version has exactly one lifecycle state at a time. State is not inferred from logs or tags.

2. **Transition validation before side effects.** No deployment, promotion, or rollback executes without a valid transition recorded by the control plane.

3. **Audit by default.** Promotion decisions, deployments, drift events, and rollbacks produce append-only audit records with actor, timestamp, and rationale.

4. **Bounded service responsibilities.** Each service owns one domain. Cross-cutting concerns live in `packages/common-schemas` and `packages/observability`.

5. **Integration over reinvention.** Experiment tracking (MLflow), orchestration (Airflow/Prefect), and runtime (Kubernetes) are boundaries, not reimplementations.

6. **Local demo is a subset, not a lie.** Local mode may stub external systems; stubs must be labeled and must not emit fake production metrics.

7. **Documentation precedes code.** API contracts and state transitions are documented before implementation. Code that contradicts docs is a defect.

---

## Lifecycle Ownership

| Domain | Owner |
|--------|-------|
| Lifecycle state machine | `control-plane` |
| Training run submission and status | `training-orchestrator` |
| Model version and artifact metadata | `registry-service` |
| Staging/production deployment | `deployment-manager` |
| Drift and health signals | `monitoring-service` |
| Operator-initiated actions | `operator-console` → `control-plane` API |

The control plane is the **only** component authorized to mutate lifecycle state. Other services emit events and request transitions; they do not self-assign states.

---

## Observability Requirements

All services (when implemented) must emit:

- Structured logs with `correlation_id`, `model_id`, `version_id`, `transition` where applicable
- Metrics for request latency, error rate, and domain-specific counters (e.g., `lifecycle_transitions_total`)
- Distributed traces across control plane → downstream service calls
- State transition events as first-class observability data (not only log lines)

See [observability-and-reliability.md](../architecture/observability-and-reliability.md).

---

## Reliability Expectations

| Concern | Requirement |
|---------|-------------|
| State durability | Lifecycle state survives process restarts; stored in transactional DB |
| Idempotent transitions | Retried requests with same idempotency key do not double-apply side effects |
| Failure visibility | `FAILED` is a valid terminal-adjacent state with reason and recovery path documented |
| Rollback safety | Rollback does not delete audit history; prior production version is restorable |
| Degraded mode | Read-only lifecycle queries work when downstream deployer is unavailable; mutating transitions fail closed |

---

## Documentation Requirements

- Every service directory contains a README with purpose, boundaries, and status
- Architecture docs distinguish **Implemented**, **Planned**, and **Future Work**
- Workflow docs describe happy path and failure paths
- Runbooks describe operational procedures; architecture-phase runbooks do not include fake commands
- Examples are conceptual contracts, not claims of working APIs

---

## Implementation Enforcement Rules

1. **No implementation without documented transition.** New lifecycle states or edges require updates to [runtime-model.md](../architecture/runtime-model.md) and [model-lifecycle-workflow.md](../workflows/model-lifecycle-workflow.md) first.

2. **No silent state mutation.** Direct DB updates to lifecycle state outside the control plane are forbidden in production paths.

3. **No unauthenticated promotion in production.** Approval gates require authenticated operator identity (planned; local demo may relax with explicit env flag).

4. **No fake metrics in CI or demo.** Tests use fixtures; dashboards consume real exporters or documented no-op exporters.

5. **Schema changes are versioned.** `packages/common-schemas` owns event and API payload definitions; breaking changes increment version.

6. **PRs must not claim Implemented without tests.** Definition of done for a feature includes integration test or documented manual validation procedure in runbook.

---

## Future Work

- Multi-tenant isolation and per-tenant lifecycle policies
- Policy-as-code for promotion (OPA/Rego or equivalent)
- Federated registry across regions
- Automated canary analysis integration
