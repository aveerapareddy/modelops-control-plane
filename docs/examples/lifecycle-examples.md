# Lifecycle Examples

**Status:** Architecture Phase  
**Implementation Status:** Not Started

Conceptual contracts only. These are **not implemented APIs**. Field names and shapes inform future `packages/common-schemas` definitions.

---

## Implemented

Nothing.

---

## Planned

Schemas derived from examples below after API design review.

---

## Training Run Metadata (Conceptual)

```json
{
  "run_id": "run_8f3a2b1c",
  "model_id": "fraud_detector",
  "parent_version_id": null,
  "status": "completed",
  "config_hash": "sha256:abc123...",
  "mlflow_run_id": "optional-external-id",
  "started_at": "2026-05-28T10:00:00Z",
  "completed_at": "2026-05-28T10:42:00Z",
  "correlation_id": "corr_train_001"
}
```

Retraining example sets `parent_version_id` to the production version that triggered drift response.

---

## Model Version Metadata (Conceptual)

```json
{
  "version_id": "ver_20260528_001",
  "model_id": "fraud_detector",
  "run_id": "run_8f3a2b1c",
  "lifecycle_state": "REGISTERED",
  "state_version": 3,
  "artifact_uri": "s3://models/fraud_detector/ver_20260528_001/model.pkl",
  "artifact_checksum": "sha256:def456...",
  "created_at": "2026-05-28T11:00:00Z"
}
```

`lifecycle_state` is readable via API; writable only through valid transitions.

---

## Promotion Decision (Conceptual)

```json
{
  "decision_id": "promo_dec_44aa",
  "version_id": "ver_20260528_001",
  "decision": "APPROVED",
  "approver": "user:jdoe",
  "comment": "AUC 0.91 exceeds gate 0.88; calibration within bounds",
  "decided_at": "2026-05-28T12:00:00Z",
  "evaluation_id": "eval_99ff"
}
```

Rejected example:

```json
{
  "decision_id": "promo_dec_44ab",
  "version_id": "ver_20260528_002",
  "decision": "REJECTED",
  "approver": "user:asmith",
  "comment": "Latency p99 regression vs baseline",
  "decided_at": "2026-05-28T14:00:00Z"
}
```

---

## Drift Event (Conceptual)

```json
{
  "drift_event_id": "drift_7c21",
  "model_id": "fraud_detector",
  "production_version_id": "ver_20260501_010",
  "signal_type": "data_drift",
  "metric_name": "population_stability_index",
  "metric_value": 0.24,
  "threshold": 0.20,
  "detected_at": "2026-06-15T08:00:00Z",
  "feature_group": "transaction_amount_v2"
}
```

Must be producible by monitoring service logic, not manual JSON injection in production paths.

---

## Rollback Event (Conceptual)

```json
{
  "rollback_id": "rb_12dd",
  "model_id": "fraud_detector",
  "from_version_id": "ver_20260610_003",
  "to_version_id": "ver_20260501_010",
  "reason": "error_rate_above_slo",
  "initiated_by": "policy:auto_rollback",
  "started_at": "2026-06-16T02:00:00Z",
  "completed_at": "2026-06-16T02:03:00Z",
  "correlation_id": "corr_rb_001"
}
```

---

## Lifecycle Transition Event (Conceptual)

```json
{
  "event_id": "evt_transition_001",
  "model_id": "fraud_detector",
  "version_id": "ver_20260528_001",
  "from_state": "PROMOTION_PENDING",
  "to_state": "PROMOTION_APPROVED",
  "actor": "user:jdoe",
  "timestamp": "2026-05-28T12:00:01Z",
  "idempotency_key": "idem_promo_44aa",
  "correlation_id": "corr_promo_44aa"
}
```

---

## Future Work

- OpenAPI spec generated from stabilized schemas
- Contract tests between services using these payloads
