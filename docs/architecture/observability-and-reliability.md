# Observability and Reliability

**Status:** Architecture Phase  
**Implementation Status:** Not Started

Planned observability and reliability requirements for all platform services.

---

## Implemented

Nothing.

---

## Planned

Unified observability via `packages/observability` and infra configs under `infra/prometheus` and `infra/grafana`.

---

## Observability Pillars

### Metrics

All services expose Prometheus-compatible metrics with standard labels:

- `service`, `environment`, `model_id` (where applicable), `version_id` (where applicable)

| Metric (planned name) | Type | Purpose |
|-------------------------|------|---------|
| `lifecycle_transitions_total` | Counter | Transitions by `from`, `to`, `result` |
| `lifecycle_transition_duration_seconds` | Histogram | Control plane transition latency |
| `training_run_duration_seconds` | Histogram | End-to-end training time |
| `evaluation_gate_pass_total` | Counter | Eval pass/fail |
| `deployment_attempts_total` | Counter | By environment and status |
| `deployment_duration_seconds` | Histogram | Deploy latency |
| `drift_events_total` | Counter | By `signal_type` |
| `retraining_triggered_total` | Counter | Drift-driven retrains |
| `rollback_executions_total` | Counter | Manual vs automated |
| `workflow_failures_total` | Counter | By stage and `failure_reason` |
| `http_requests_total` | Counter | Standard RED metrics per service |

### Logs

- Structured JSON logs
- Required fields: `timestamp`, `level`, `service`, `correlation_id`, `message`
- Lifecycle logs add: `model_id`, `version_id`, `from_state`, `to_state`, `actor`
- No secrets or raw artifact payloads in logs

### Traces

- OpenTelemetry propagation from operator console through control plane to downstream services
- Span names reflect domain: `transition.validate`, `registry.register`, `deploy.apply`
- Sample rate configurable; 100% in local demo acceptable

### State transition events

Transitions are not only logged—they are **exportable events** for:

- Prometheus counters (above)
- Optional event stream for external SIEM (future)
- Grafana annotations for incident correlation

---

## Domain-Specific Signals

| Domain | Signals |
|--------|---------|
| Training | Run start/end, duration, resource errors, `TRAINING` → `FAILED` |
| Evaluation | Metric values at eval time, gate pass/fail, comparison to baseline |
| Deployment | Target, revision, success/fail, active production version |
| Drift | Metric name, value, threshold, detection method |
| Retraining | Parent version, new run_id, lineage |
| Rollback | From/to version, duration, post-rollback health |

---

## Reliability Requirements

### Availability targets (portfolio / local)

| Tier | Target | Scope |
|------|--------|-------|
| Control plane | Restart-safe | State in DB; no in-memory-only lifecycle |
| Downstream services | Best effort in demo | Degraded mode documented in runbook |
| Production-grade | Not claimed | Future work for HA deployment |

### Failure handling

| Failure | Behavior |
|---------|----------|
| Invalid transition | 4xx response, no side effects, metric `result=rejected` |
| Downstream deploy timeout | Remain in `DEPLOYING` or move to `FAILED` with reason; no silent success |
| DB unavailable | Fail closed on mutations; read cache not authoritative |
| Duplicate idempotency key | Return original outcome |
| Monitoring outage | Do not auto-trigger drift; alert on monitoring health separately |

### Workflow failures

Stages emit `workflow_failures_total` with labels:

- `stage`: `training`, `evaluation`, `registration`, `promotion`, `deployment`, `monitoring`, `drift`, `retraining`, `validation`, `rollback`
- `failure_reason`: stable enum (e.g., `TIMEOUT`, `GATE_FAILED`, `APPROVAL_DENIED`)

---

## Alerting (Planned)

Local demo uses Grafana alert rules on:

- Stuck `DEPLOYING` > threshold
- `FAILED` rate spike
- No successful scrape from control plane
- Drift events without subsequent `RETRAINING` within policy window (optional)

No fake alert webhooks. Alert rules reference real metric series or documented no-op exporter.

---

## SLOs (Documentation Only — Not Implemented)

| SLI | Notes |
|-----|-------|
| Transition API success rate | Excludes client invalid transitions |
| Time to complete deploy after `PROMOTION_APPROVED` | Measured in demo scripts |

Formal SLO/error budget process is future work.

---

## Future Work

- SLO dashboards and error budget burn alerts
- Log-based metrics for audit anomaly detection
- Synthetic probes against production inference endpoints
