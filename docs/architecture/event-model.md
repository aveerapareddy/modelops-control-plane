# Event Model

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

Conceptual lifecycle event contracts. Not implemented schemas. Future `packages/common-schemas` will derive from this document.

---

## Implemented

No event bus, no event store, no serializers. No runtime behavior is implemented.

---

## Planned

- Append-only event log (DB table or event store)
- Transition API emits event after state commit
- Prometheus counters increment from event types
- Optional webhook consumers per event type

---

## Future Work

- CloudEvents envelope
- Kafka/NATS topic per `event_type`
- Event replay for disaster recovery

---

## Common Event Fields (All Types)

| Field | Type (conceptual) | Required | Description |
|-------|-------------------|----------|-------------|
| `event_id` | UUID | Yes | Unique event identifier |
| `event_type` | string | Yes | Stable enum matching names below |
| `model_id` | string | Yes | Logical model family |
| `model_version_id` | string | Yes | Version scope |
| `run_id` | string | No | Training run when applicable |
| `deployment_id` | string | No | Deploy/rollback scope |
| `previous_state` | string | No | Lifecycle state before transition |
| `next_state` | string | No | Lifecycle state after transition |
| `actor` | string | Yes | `user:*`, `service:*`, `policy:*` |
| `reason` | string | No | Human or policy justification |
| `evidence_uri` | URI | No | Link to eval report, drift report, approval record |
| `correlation_id` | string | Yes | Traces single workflow across services |
| `created_at` | ISO8601 | Yes | Event timestamp (UTC) |

Additional type-specific fields live in `payload` (conceptual JSON object).

---

## Event Catalog

### DataPrepared

| Attribute | Value |
|-----------|-------|
| **Producer** | Control plane |
| **Consumers** | Training orchestrator, audit |
| **Required fields** | `config_hash`, data manifest `evidence_uri` |
| **Retention** | Life of model + 7y audit (policy TBD) |
| **Audit importance** | High — reproducibility anchor |

### TrainingStarted

| Producer | Training orchestrator (via control plane transition) |
| Consumers | Observability, audit |
| Required | `run_id`, `config_hash` |
| Retention | 2y operational |
| Audit | High |

### TrainingCompleted

| Producer | Training orchestrator |
| Consumers | Control plane, registry (pre-register), audit |
| Required | `run_id`, artifact staging URI |
| Retention | 2y |
| Audit | High |

### TrainingFailed

| Producer | Training orchestrator |
| Consumers | Control plane, alerting, audit |
| Required | `run_id`, `reason`, error code |
| Retention | 2y |
| Audit | High |

### EvaluationStarted

| Producer | Control plane |
| Consumers | Audit |
| Required | `run_id` |
| Retention | 2y |
| Audit | Medium |

### EvaluationCompleted

| Producer | Control plane |
| Consumers | Registry, promotion policy, audit |
| Required | `evaluation_id`, metrics `evidence_uri`, `passed` |
| Retention | Life of version |
| Audit | Critical — gate evidence |

### EvaluationFailed

| Producer | Control plane |
| Consumers | Operator console, audit |
| Required | `evaluation_id`, metrics, `passed=false` |
| Retention | Life of version |
| Audit | Critical |

### ModelRegistered

| Producer | Registry service |
| Consumers | Control plane, audit |
| Required | `artifact_uri`, checksum |
| Retention | Life of version |
| Audit | Critical |

### PromotionRequested

| Producer | Control plane |
| Consumers | Operator console, policy engine, audit |
| Required | `promotion_request_id` |
| Retention | Life of version |
| Audit | High |

### PromotionApproved

| Producer | Control plane |
| Consumers | Deployment manager, audit |
| Required | `decision_id`, approver, policy result URI |
| Retention | Life of version |
| Audit | Critical |

### PromotionRejected

| Producer | Control plane |
| Consumers | Operator console, audit |
| Required | `decision_id`, `comment` |
| Retention | Life of version |
| Audit | Critical |

### StagingStarted

| Producer | Deployment manager |
| Consumers | Control plane, audit |
| Required | `deployment_id`, staging target |
| Retention | 2y |
| Audit | High |

### DeploymentStarted

| Producer | Deployment manager |
| Consumers | Control plane, observability |
| Required | `deployment_id`, slot (`blue`/`green`) |
| Retention | 2y |
| Audit | High |

### DeploymentSucceeded

| Producer | Deployment manager |
| Consumers | Control plane, monitoring service |
| Required | `deployment_id`, health `evidence_uri` |
| Retention | Life of version |
| Audit | Critical |

### DeploymentFailed

| Producer | Deployment manager |
| Consumers | Control plane, alerting |
| Required | `deployment_id`, `reason` |
| Retention | 2y |
| Audit | Critical |

### MonitoringStarted

| Producer | Monitoring service |
| Consumers | Control plane, observability |
| Required | Monitor config ref |
| Retention | 1y operational |
| Audit | Medium |

### DriftDetected

| Producer | Monitoring service |
| Consumers | Control plane, policy engine, audit |
| Required | `drift_event_id`, metric, value, threshold, methodology ref |
| Retention | Life of model incident |
| Audit | Critical — must not be synthetic |

### RetrainingStarted

| Producer | Control plane |
| Consumers | Training orchestrator, audit |
| Required | `parent_version_id`, `run_id` |
| Retention | 2y |
| Audit | High |

### RetrainingCompleted

| Producer | Training orchestrator |
| Consumers | Control plane |
| Required | `run_id` |
| Retention | 2y |
| Audit | High |

### ValidationStarted

| Producer | Control plane |
| Consumers | Audit |
| Required | `baseline_version_id` |
| Retention | Life of challenger |
| Audit | High |

### ValidationCompleted

| Producer | Control plane |
| Consumers | Promotion workflow, audit |
| Required | Comparison `evidence_uri`, `passed=true` |
| Retention | Life of version |
| Audit | Critical |

### ValidationFailed

| Producer | Control plane |
| Consumers | Operator console, audit |
| Required | Comparison `evidence_uri`, `passed=false` |
| Retention | Life of version |
| Audit | Critical |

### RollbackRequested

| Producer | Control plane |
| Consumers | Deployment manager, audit |
| Required | `rollback_id`, target `version_id`, `reason` |
| Retention | Permanent incident record |
| Audit | Critical |

### RollbackStarted

| Producer | Deployment manager |
| Consumers | Control plane, observability |
| Required | `rollback_id`, `from_version_id`, `to_version_id` |
| Retention | Permanent |
| Audit | Critical |

### RollbackCompleted

| Producer | Deployment manager |
| Consumers | Control plane, monitoring service |
| Required | `rollback_id`, health `evidence_uri` |
| Retention | Permanent |
| Audit | Critical |

### RollbackFailed

| Producer | Deployment manager |
| Consumers | Control plane, paging |
| Required | `rollback_id`, `reason` |
| Retention | Permanent |
| Audit | Critical |

### LifecycleFailed

| Producer | Control plane |
| Consumers | Operator console, paging, audit |
| Required | `reason`, last known `previous_state` |
| Retention | Permanent |
| Audit | Critical |

---

## Event Emission Rules

1. Event is emitted **after** durable state commit (outbox pattern planned).
2. `previous_state` and `next_state` must match [transition-rules.md](transition-rules.md).
3. `actor` on the event must match transition actor.
4. Missing required evidence field → transition rejected; no event emitted.
5. `correlation_id` inherited from parent workflow (training, promotion, rollback).

---

## Consumer Responsibilities

| Consumer | Rule |
|----------|------|
| Training orchestrator | React only to `RetrainingStarted`, `TrainingStarted` triggers |
| Deployment manager | React to `PromotionApproved`, `RollbackStarted` |
| Monitoring service | Emit `DriftDetected`; never mutate lifecycle directly |
| Operator console | Read-only except via control plane API |
| Audit store | Append-only; no delete |

---

## Related Documents

- [transition-rules.md](transition-rules.md)
- [../examples/lifecycle-examples.md](../examples/lifecycle-examples.md)
