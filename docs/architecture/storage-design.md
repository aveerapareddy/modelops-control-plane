# Storage Design

**Status:** Architecture Phase  
**Implementation Status:** Not Started

Planned persistence layer responsibilities. No migrations or schemas exist yet.

---

## Implemented

Nothing.

---

## Planned

PostgreSQL (or equivalent relational store) as primary system of record for lifecycle and audit data. Object storage for artifacts by reference only.

---

## Storage Principles

1. **Lifecycle state lives in the control plane database**, not in MLflow tags or K8s labels alone.
2. **Artifacts are immutable** once registered; new training produces new `version_id`.
3. **Audit is append-only**; corrections are new records, not overwrites.
4. **Foreign keys and constraints** enforce referential integrity between runs, versions, and deployments.

---

## Entity Responsibilities

### Training runs

| Field (conceptual) | Purpose |
|--------------------|---------|
| `run_id` | Unique training execution |
| `model_id` | Logical model family |
| `parent_version_id` | Lineage for retraining (nullable) |
| `status` | Run-level status (orchestrator domain) |
| `config_hash` | Reproducibility fingerprint |
| `started_at`, `completed_at` | Timing |
| `mlflow_run_id` | Optional external reference |

Owned primarily by training orchestrator; control plane links run to lifecycle transitions.

### Model versions

| Field (conceptual) | Purpose |
|--------------------|---------|
| `version_id` | Unique version identifier |
| `model_id` | Parent model |
| `run_id` | Originating training run |
| `lifecycle_state` | Current state (control plane authoritative) |
| `state_version` | Optimistic lock counter |
| `artifact_uri` | Pointer to serialized model |
| `created_at` | Registration time |

Owned by registry service for metadata; **lifecycle_state** written only by control plane.

### Model artifacts

- Binary payloads in object storage (S3, GCS, local filesystem in demo)
- Database stores `artifact_uri`, `checksum`, `format`, `size_bytes`
- Integrity verified at register and deploy time

### Evaluation results

| Field (conceptual) | Purpose |
|--------------------|---------|
| `evaluation_id` | Unique eval record |
| `version_id` | Evaluated version |
| `metrics` | JSON blob: accuracy, latency, calibration, etc. |
| `baseline_version_id` | For challenger validation |
| `passed` | Gate result |
| `evaluated_at` | Timestamp |

Gates `EVALUATING` → `REGISTERED` and `VALIDATING` → `PROMOTION_PENDING`.

### Promotion decisions

| Field (conceptual) | Purpose |
|--------------------|---------|
| `decision_id` | Unique decision |
| `version_id` | Target version |
| `decision` | `APPROVED` / `REJECTED` |
| `approver` | Identity |
| `comment` | Rationale |
| `decided_at` | Timestamp |

Maps to `PROMOTION_APPROVED` / `PROMOTION_REJECTED`.

### Deployment history

| Field (conceptual) | Purpose |
|--------------------|---------|
| `deployment_id` | Unique deploy attempt |
| `version_id` | Deployed version |
| `environment` | `staging` / `production` |
| `target_ref` | K8s deployment name, service ID, etc. |
| `status` | `in_progress` / `succeeded` / `failed` |
| `deployed_at` | Timestamp |

Supports rollback target selection (last known-good).

### Drift events

| Field (conceptual) | Purpose |
|--------------------|---------|
| `drift_event_id` | Unique event |
| `model_id`, `production_version_id` | Context |
| `signal_type` | `data_drift`, `concept_drift`, `performance` |
| `metric_name`, `metric_value`, `threshold` | Evidence |
| `detected_at` | Timestamp |

Triggers `DRIFT_DETECTED` transition.

### Rollback events

| Field (conceptual) | Purpose |
|--------------------|---------|
| `rollback_id` | Unique rollback |
| `from_version_id`, `to_version_id` | Versions |
| `reason` | Operator or policy |
| `initiated_by` | Actor |
| `completed_at` | Timestamp |

### Audit trail

Append-only log spanning all privileged actions:

- State transitions
- Promotion decisions
- Deploy and rollback
- Policy overrides
- Authentication events (planned)

Fields: `audit_id`, `entity_type`, `entity_id`, `action`, `actor`, `payload_json`, `timestamp`.

---

## Access Patterns (Planned)

| Query | Consumer |
|-------|----------|
| Current state by `version_id` | Control plane, console |
| History of transitions by `version_id` | Console, compliance |
| Production version by `model_id` | Deployment manager, monitoring |
| Pending promotions | Console approval queue |
| Drift events by `model_id` since timestamp | Retraining policy |

---

## What We Are Not Building Yet

- Database migrations
- ORM models
- Connection pooling config
- Backup/restore automation

See `infra/db/` when implementation begins.

---

## Future Work

- Time-series store for high-volume inference metrics (Prometheus long-term storage or TSDB)
- Separate read model (CQRS) for console aggregations if query load requires it
- Encryption at rest and column-level encryption for PII in audit payloads
