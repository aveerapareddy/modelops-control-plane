# Local Runbook

**Status:** Architecture Phase  
**Implementation Status:** Not Started

Architecture-phase runbook only. **No commands in this document start a running system**—the platform is not implemented.

---

## Implemented

Nothing.

---

## Planned

Executable local demo per phases in [project-end-state.md](../overview/project-end-state.md).

---

## Current Repository Status

| Item | Status |
|------|--------|
| Architecture documentation | Present (Session 0) |
| Services | Directory boundaries only |
| Database | Not provisioned |
| Docker compose | Not defined |
| API endpoints | Do not exist |
| Lifecycle state machine | Not implemented |

---

## Local Setup Expectations (Future)

When implementation begins, local development is expected to require:

- Python 3.11+ (version pinned in project config when added)
- Docker and docker-compose for PostgreSQL, Prometheus, Grafana
- Optional: MLflow tracking server for experiment URI linking
- Environment file from `.env.example` (not yet created)

This runbook will be updated with verified commands before any step claims to be runnable.

---

## Future Local Demo Intent (Planned)

Goal: demonstrate full lifecycle on a developer machine without Kubernetes.

1. Start infrastructure: PostgreSQL, Prometheus, Grafana via `infra/docker/`
2. Start control plane and dependent services
3. Execute training via orchestrator (minimal sklearn or equivalent script)
4. Walk through promotion approval in operator console
5. Deploy to local HTTP endpoint or container
6. Inject or compute drift signal with documented methodology
7. Complete retrain → validate → promote cycle
8. Execute rollback to prior version
9. Verify audit trail via API or SQL query

Success criteria: [measurable completion criteria](../overview/project-end-state.md#measurable-completion-criteria).

---

## Future Validation Flow (Planned)

| Check | Method |
|-------|--------|
| Invalid transition rejected | API integration test |
| Audit completeness | Query by `version_id` |
| Metrics present | Prometheus scrape targets up |
| No fake drift | Drift event includes `metric_name`, `metric_value`, `threshold` |

---

## Current Limitations

- Cannot train, register, deploy, or monitor models from this repository
- No health endpoints to curl
- No database migrations to apply
- No operator console URL
- Diagrams under `docs/diagrams/` may be added later; none required for Session 0

---

## Operator Actions (Future Reference)

When implemented, privileged actions will require:

- Authenticated session (or `DEMO_MODE` with console warning)
- Confirmation for rollback
- Approver role for promotion

---

## Future Work

- `make demo` or equivalent single entrypoint once services exist
- Troubleshooting section: stuck states, DB connection failures, scrape failures
- CI-local parity: same compose file used in GitHub Actions integration job
