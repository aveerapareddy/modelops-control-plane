# Promotion Workflow

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

---

## Implemented

No runtime behavior is implemented.

---

## Planned

Policy evaluation and approval gates per [promotion-policy-model.md](../architecture/promotion-policy-model.md).

---

## Future Work

- Scheduled promotion windows
- Multi-stakeholder approval chains

---

## Purpose

Govern transition from registered (or validated challenger) version to deployment-eligible `PROMOTION_APPROVED`.

---

## Actors

| Actor | Role |
|-------|------|
| `user:operator` | Request promotion |
| `policy:auto` | Automated threshold checks |
| `user:approver` | Approve or reject |
| `service:control-plane` | Enforce gates, record decisions |

---

## Inputs

- `model_version_id` in `REGISTERED` or entering from `VALIDATING`
- `evaluation_id` (initial) or validation report (challenger)
- Policy version for `model_id`

---

## Outputs

- `decision_id`
- State: `PROMOTION_APPROVED` or `PROMOTION_REJECTED`
- Audit record with evidence URIs

---

## Required Evidence

| Step | Evidence |
|------|----------|
| Request | `promotion_request_id` |
| Policy auto-check | Metric snapshot URI, baseline comparison URI |
| Approve | Approver ID, policy pass proof |
| Reject | `comment`, failing metric IDs |

---

## State Transitions

```
REGISTERED → PROMOTION_PENDING → PROMOTION_APPROVED → STAGING (via deploy workflow)
                              → PROMOTION_REJECTED → PROMOTION_PENDING (new request)

VALIDATING → PROMOTION_PENDING (challenger path, after validation pass)
```

---

## Events Emitted

`PromotionRequested` → (`PromotionApproved` | `PromotionRejected`)

---

## Failure Modes

| Failure | Result |
|---------|--------|
| Policy engine error | Remain `PROMOTION_PENDING` or `FAILED` |
| Missing eval evidence | Request rejected at API |
| Approver timeout | Remain `PROMOTION_PENDING` (no auto-approve unless policy allows) |
| Separation-of-duties violation | Reject request |

---

## Recovery Paths

| From | Action |
|------|--------|
| `PROMOTION_REJECTED` | Remediate metrics, retrain if needed, new `PromotionRequested` |
| `PROMOTION_PENDING` | Cancel request (future); approver acts |

---

## Automated vs Manual

| Condition | Path |
|-----------|------|
| All thresholds pass + `allow_auto_promote` | `policy:auto` → `PROMOTION_APPROVED` |
| Any review flag or drift challenger | Manual approver required |
| Any hard fail | `PROMOTION_REJECTED` |

---

## Current Implementation Status

Documentation only. No policy engine, no approval API, no operator console.
