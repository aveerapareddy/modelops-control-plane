# Security and Guardrails

**Status:** Architecture Phase  
**Implementation Status:** Not Started

Planned security and governance controls.

---

## Implemented

Nothing.

---

## Planned

Controls below apply at implementation; local demo may use simplified auth with explicit `DEMO_MODE` flag.

---

## Operator Permissions

| Role | Permissions |
|------|-------------|
| `viewer` | Read lifecycle, audit, metrics |
| `operator` | Request promotion, trigger retraining, initiate rollback |
| `approver` | Approve/reject promotion (`PROMOTION_PENDING` → `PROMOTION_*`) |
| `admin` | Policy config, break-glass override (audited) |
| `service` | Scoped service accounts per microservice |

Principle of least privilege. Service accounts cannot approve their own promotions.

---

## Approval Gates

- `REGISTERED` → `PROMOTION_PENDING` may be operator-initiated
- `PROMOTION_PENDING` → `PROMOTION_APPROVED` requires `approver` role
- Separation of duties: training submitter ≠ sole approver (configurable policy)
- All decisions persist `approver`, `timestamp`, `comment` in promotion decision store

Rejected promotions (`PROMOTION_REJECTED`) block deployment paths until a new promotion request is evaluated.

---

## Promotion Governance

- Evaluation gate must pass before `REGISTERED` is reachable from `EVALUATING`
- Challenger path requires `VALIDATING` with explicit baseline comparison before re-promotion
- Artifact `checksum` at register must match at deploy
- Config hash at training must be stored for reproducibility disputes

---

## Rollback Permissions

| Initiator | Allowed from state |
|-----------|-------------------|
| `operator` | `PRODUCTION`, `MONITORING` |
| `automated_policy` | `MONITORING` when SLO breach or drift severity exceeds threshold (planned) |
| `service` (deployment manager) | Only on deploy failure callback to `FAILED`, not arbitrary rollback |

Rollback target must be a **previously successful production version** recorded in deployment history.

---

## Audit Logging

Append-only audit trail for:

- Every lifecycle transition
- Promotion approve/reject
- Deploy start/complete/fail
- Rollback start/complete
- Admin overrides
- Authentication failures (planned)

Audit records are queryable by `model_id`, `version_id`, `actor`, time range. Retention policy TBD; demo uses indefinite local retention.

---

## Artifact Integrity

| Check | When |
|-------|------|
| Checksum verify on register | `EVALUATING` → `REGISTERED` |
| Checksum verify before deploy | `DEPLOYING` |
| URI allowlist | Only `artifact_uri` schemes registered in config (e.g., `s3://`, `file://` for demo) |
| Immutable URI | No in-place artifact overwrite for a given `version_id` |

---

## Configuration Integrity

- Training and deployment configs stored with `config_hash` (SHA-256 of canonical JSON)
- Promotion and deploy actions reference `config_hash`; mismatch fails transition
- Secrets never in lifecycle DB; references via secret manager IDs (production) or env injection (local)

---

## Network and API Guardrails (Planned)

- mTLS or JWT between services
- Rate limits on transition API
- Input validation on all mutation payloads via `common-schemas`
- No arbitrary SQL or script execution from operator console

---

## Future Work

- OIDC integration for operator identities
- OPA/Rego policy engine for promotion rules
- Signed audit log chain (hash chain per day)
- VPC-scoped artifact access for production artifacts
