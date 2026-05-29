# Tradeoffs

**Status:** Architecture Phase  
**Implementation Status:** Not Started

Early architectural decisions and alternatives rejected or deferred.

---

## Implemented

Nothing.

---

## Planned

Decisions below guide implementation unless superseded by an ADR with explicit rationale.

---

## Build vs Integrate

| Capability | Decision | Rationale |
|------------|----------|-----------|
| Lifecycle state machine | **Build** | Core differentiator; must not delegate to MLflow stages |
| Experiment tracking | **Integrate** (MLflow optional) | Commodity; store references only |
| Workflow orchestration | **Integrate** (Airflow/Prefect boundary) | Training orchestrator submits jobs; does not embed DAG engine |
| Metrics backend | **Integrate** (Prometheus) | Standard; custom TSDB not justified |
| Model serving runtime | **Integrate** (K8s/process) | Deployment manager applies configs; no custom inference server |
| Auth | **Integrate** (OIDC planned) | No custom identity provider |

**Risk:** Over-integration couples release cycles. Mitigation: adapter interfaces per boundary.

---

## MLflow Integration Boundary

| In scope | Out of scope |
|----------|--------------|
| Store `mlflow_run_id`, experiment ID | MLflow as lifecycle state source |
| Link artifacts if MLflow logged them | Require MLflow for registry to function |
| Optional URI resolution for demo | Sync MLflow stage names to lifecycle states |

**Tradeoff:** Teams without MLflow must still register via registry API with direct `artifact_uri`.

---

## Orchestration Engine Boundary

| Training orchestrator owns | Engine owns |
|---------------------------|-------------|
| Job spec, callback to control plane | DAG scheduling, retries, sensors |
| Mapping run_id → version lineage | Cluster resource management |

**Alternative rejected:** Embed Prefect/Airflow inside control plane—violates single responsibility.

**Local demo:** Subprocess or simple queue worker acceptable; must still emit same callbacks as production adapter.

---

## Local Demo vs Production Platform

| Aspect | Local demo | Production target |
|--------|------------|-------------------|
| HA | Single instance | Replicated control plane (future) |
| Auth | `DEMO_MODE` relaxed | OIDC + RBAC |
| Artifacts | `file://` URIs | `s3://` with IAM |
| Drift | Documented statistical test on sample data | Pipeline-connected monitoring |
| Deploy target | Local process or docker | Kubernetes |

**Tradeoff:** Demo code paths must not become the only tested path. Integration tests should target the same transition API as demo.

---

## Real Drift Detection vs Fake Alerts

| Approach | Verdict |
|----------|---------|
| Hardcoded `drift_detected=true` | **Rejected** |
| Random alert generator | **Rejected** |
| Statistical test (KS, PSI) on logged features with thresholds | **Accepted for demo** with methodology in docs |
| Production monitoring integration | **Planned** |

**Tradeoff:** Real drift detection requires data plumbing; demo invests in minimal real computation, not UI theater.

---

## Operator Console: Platform Tool vs Vanity Dashboard

| Build | Do not build |
|-------|--------------|
| Lifecycle state per version | Fancy chart gallery |
| Approval queue | Non-actionable metric tiles |
| Rollback with confirmation | Custom CSS showcase |
| Links to Grafana for deep metrics | Reimplement Prometheus UI |

Console is for **actions and state**, not analytics replacement.

---

## Synchronous vs Event-Driven Coordination

**Initial:** Synchronous HTTP callbacks from orchestrator/deployment manager to control plane.

**Future:** Event bus for decoupling and replay.

**Tradeoff:** Simpler debugging and portfolio delivery speed vs scale. Acceptable for v1.

---

## PostgreSQL as Single Store

**Decision:** One relational DB for lifecycle + audit + registry metadata.

**Alternative rejected:** Split registry to separate DB in v1—increases ops burden without portfolio benefit.

**Future:** Read replicas and TSDB for metrics if load requires.

---

## Future Work

- Formal ADR process in `docs/architecture/adr/`
- Revisit event-driven architecture after Phase 3
- Evaluate Temporal for long-running deploy sagas
