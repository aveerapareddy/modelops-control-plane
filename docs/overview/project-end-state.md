# Project End State

**Status:** Architecture Phase  
**Implementation Status:** Not Started

Defines what "done" means for ModelOps Control Plane. This is the completion target for the portfolio project, not Session 0.

---

## Implemented

Nothing.

---

## Planned

Full platform as described below.

---

## Definition of Complete

The project is complete when an operator (or scripted demo) can execute the full lifecycle **end to end** on a local or documented staging environment, with reconstructable audit evidence at each step:

```
Train → Evaluate → Register → Promote → Deploy → Monitor
  → Detect Drift → Retrain → Validate → Promote → Rollback
```

"Complete" does not mean production multi-region deployment. It means the lifecycle is **demonstrable**, **governed**, and **auditable** with real state transitions—not mocked UI flows.

---

## Major Platform Components

| Component | Done When |
|-----------|-----------|
| Control plane | State machine enforces all documented transitions; invalid transitions rejected |
| Training orchestrator | Submits training job, reports completion/failure to control plane |
| Registry service | Registers model version with artifact URI and evaluation summary reference |
| Deployment manager | Deploys approved version to staging/production targets; records deployment history |
| Monitoring service | Ingests drift/health signals; triggers `DRIFT_DETECTED` via control plane |
| Operator console | Surfaces lifecycle state, approval queue, rollback action (minimal, operational) |
| Storage | Persists runs, versions, promotions, deployments, drift, rollback, audit trail |
| Observability | Metrics, logs, traces, and transition events visible in local Prometheus/Grafana or equivalent |

---

## Primary Lifecycle Workflow

1. **Train:** Orchestrator runs training; control plane holds `TRAINING` → `EVALUATING`.
2. **Evaluate:** Metrics recorded; gate pass/fail determines registration eligibility.
3. **Register:** Version enters registry; state `REGISTERED`.
4. **Promote:** Operator or policy requests promotion; approval gate → `PROMOTION_APPROVED` or `PROMOTION_REJECTED`.
5. **Deploy:** Deployment manager applies artifact; `STAGING` → `DEPLOYING` → `PRODUCTION`.
6. **Monitor:** Production inference monitored; state `MONITORING`.
7. **Detect drift:** Monitoring emits drift event; `DRIFT_DETECTED`.
8. **Retrain:** New training run linked to parent version; `RETRAINING` → `VALIDATING`.
9. **Validate:** Challenger evaluated against production baseline.
10. **Promote:** Approved challenger follows promotion path again.
11. **Rollback:** Operator or automated policy reverts to last known-good production version.

See [model-lifecycle-workflow.md](../workflows/model-lifecycle-workflow.md) for paths and failure modes.

---

## Phase Outputs

| Phase | Output |
|-------|--------|
| 0 – Architecture | Docs, repo skeleton, service boundaries (current) |
| 1 – Control plane core | State machine + persistence + transition API |
| 2 – Registry + training hook | Register versions from completed training runs |
| 3 – Promotion + deployment | Approval workflow and staging/prod deploy |
| 4 – Monitoring + drift | Real signal ingestion (not fake alerts) |
| 5 – Retrain + validate + rollback | Closed loop through rollback |
| 6 – Operator console + demo | End-to-end local demo with runbook |

---

## Intentional Non-Goals

- Replacing enterprise feature stores or data platforms
- Building a proprietary training framework
- Multi-cloud control plane federation
- Real-time model serving optimization (batching, GPU scheduling)
- Full SOC2/compliance certification (patterns documented only)
- 99.99% SLA claims for portfolio scope

---

## Local / Demo Limitations

End-state demo runs on **local subset**:

- Single-node or docker-compose deployment
- Stubbed or lightweight training job (e.g., sklearn script), not cluster-scale training
- MLflow optional integration for experiment URI, not required for lifecycle proof
- Drift may use statistical test on synthetic distribution shift **with documented methodology**—not hardcoded `drift=true`
- Production target may be a local HTTP endpoint or process, not multi-cluster K8s

Demo must still exercise **real state transitions** and **audit records**.

---

## Measurable Completion Criteria

| # | Criterion | Verification |
|---|-----------|--------------|
| 1 | Invalid transition returns error | API test: e.g., `REGISTERED` → `PRODUCTION` without promotion rejected |
| 2 | Full happy path completes | Runbook script or integration test completes Train→Deploy→Monitor |
| 3 | Drift path completes | Injected or computed drift triggers retrain workflow to `VALIDATING` |
| 4 | Rejected promotion path | Approval denial leaves version in `PROMOTION_REJECTED` with audit entry |
| 5 | Rollback path | Production serves previous version; audit shows rollback event |
| 6 | Audit reconstructability | Given `version_id`, query returns promotion, deploy, drift, rollback chain |
| 7 | Observability | At least one dashboard panel per: transitions, training duration, deployments, drift |
| 8 | Documentation match | README and architecture docs accurately reflect implemented behavior |

---

## Future Work

- Production hardening: HA control plane, regional failover
- Advanced policy engine for auto-promotion and auto-rollback
- Cost attribution per training run and deployment
- Integration with enterprise IAM (OIDC/SAML)
