# Promotion Policy Model

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

Semantics for when a model version may move from `REGISTERED` or post-`VALIDATING` into `PROMOTION_PENDING` and subsequently `PROMOTION_APPROVED`.

---

## Implemented

No runtime behavior is implemented. No policy engine, no threshold evaluation code.

---

## Planned

Control plane evaluates policy before `PROMOTION_APPROVED`. Operator approval required when automatic conditions not met.

---

## Future Work

- OPA/Rego policy bundles
- Per-model policy versions
- Time-windowed promotion freezes

---

## Policy Purpose

Promotion policy answers: **Is this version allowed to proceed toward production deployment?**

It does not deploy models. It produces evaluable evidence consumed by:

- Automated gate (`policy:auto` → `PROMOTION_APPROVED` without human)
- Manual gate (`PROMOTION_PENDING` until `user:approver` acts)

Direct promotion without evidence is forbidden. See [transition-rules.md](transition-rules.md).

---

## Evaluation Thresholds (Planned)

| Metric class | Example checks |
|--------------|----------------|
| Quality | `auc >= policy.min_auc`, `f1 >= policy.min_f1` |
| Calibration | `ece <= policy.max_ece` |
| Latency | `p99_latency_ms <= policy.max_p99` |
| Stability | `score_variance <= policy.max_variance` on holdout |

Thresholds are versioned per `model_id` in policy config (not hardcoded in code paths).

---

## Baseline Comparison

Challenger path (`VALIDATING` → `PROMOTION_PENDING`) requires comparison against **current production version** for the same `model_id`:

| Check | Rule |
|-------|------|
| Primary metric | Challenger must exceed baseline by `policy.min_delta` |
| Regression guard | No metric may degrade beyond `policy.max_regression` |
| Sample size | Evaluation dataset must meet `policy.min_samples` |

Baseline `model_version_id` resolved from deployment history active production slot.

---

## Drift Constraints

Versions entering promotion from drift response must additionally satisfy:

| Constraint | Purpose |
|------------|---------|
| `drift_event_id` linked | Proves retrain was triggered by monitored drift |
| Drift feature group documented | Reproducibility |
| Retrain `parent_version_id` matches production at drift time | Lineage integrity |

Logging drift without `DRIFT_DETECTED` → `RETRAINING` transition is insufficient per [drift-retraining-workflow.md](../workflows/drift-retraining-workflow.md).

---

## Approval Requirements

| Path | Conditions |
|------|------------|
| **Automatic promotion** | All metric thresholds pass AND baseline comparison pass AND `policy.allow_auto_promote=true` AND no active promotion freeze |
| **Manual approval** | Any threshold requires human review per policy OR `allow_auto_promote=false` OR challenger from drift |
| **Rejection** | Any hard-fail threshold OR approver denial |

Approver must hold `approver` role. Submitter cannot be sole approver when `policy.require_separation_of_duties=true`.

---

## Rejection Conditions

- Any mandatory metric below threshold
- Baseline comparison fail
- Missing `evaluation_id` or `evidence_uri`
- Artifact checksum mismatch
- `config_hash` mismatch vs training record
- Approver explicit deny with `comment`

Rejected versions remain `PROMOTION_REJECTED` until new `PromotionRequested` event after remediation.

---

## Exception Handling

| Exception type | Handling |
|----------------|----------|
| Policy config missing | Fail closed → `PROMOTION_PENDING` manual only |
| Baseline version not found | Reject promotion; `reason=NO_PRODUCTION_BASELINE` |
| Eval metrics incomplete | Reject; no `PROMOTION_APPROVED` |
| Emergency break-glass | `user:admin` override with `OVERRIDE` audit event and post-incident review |

---

## Audit Requirements

Every promotion decision record includes:

- `promotion_request_id`, `decision_id`
- Policy version ID evaluated
- Metric snapshot URI (`evidence_uri`)
- Baseline `model_version_id`
- Approver identity (if manual)
- Timestamp

Stored append-only; referenced by `PromotionApproved` / `PromotionRejected` events in [event-model.md](event-model.md).

---

## Conceptual Policy Example (Not Implemented)

```yaml
# conceptual only — not loaded by any runtime
model_id: fraud_detector
policy_version: "2026-05-28.1"
min_auc: 0.88
min_delta_auc: 0.01
max_p99_latency_ms: 120
max_regression: 0.02
require_separation_of_duties: true
allow_auto_promote: false
drift_requires_manual_promote: true
```

A version with `auc=0.91` vs baseline `0.89` passes metric checks but still requires approver because `allow_auto_promote: false`.

---

## Related Documents

- [../workflows/promotion-workflow.md](../workflows/promotion-workflow.md)
- [lifecycle-state-machine.md](lifecycle-state-machine.md)
- [security-and-guardrails.md](security-and-guardrails.md)
