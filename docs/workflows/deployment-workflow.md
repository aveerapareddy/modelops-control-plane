# Deployment Workflow

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

---

## Implemented

No runtime behavior is implemented.

---

## Planned

Blue/green staging and production cutover per [deployment-strategy.md](../architecture/deployment-strategy.md).

---

## Future Work

- Canary progressive delivery
- Automated rollback on SLO burn

---

## Purpose

Deploy `PROMOTION_APPROVED` versions to staging, validate, cut over to production, and arm monitoring.

---

## Actors

| Actor | Role |
|-------|------|
| `service:deployment-manager` | Apply artifacts, switch slots |
| `service:control-plane` | State transitions |
| `user:operator` | Retry failed deploys |
| `service:monitoring-service` | Arm monitors after prod |

---

## Inputs

- `model_version_id`, `artifact_uri`, checksum
- `deployment_id` (new per attempt)
- Target environment (staging / production slot)

---

## Outputs

- Deployment history record
- Active production `model_version_id`
- States: `STAGING`, `PRODUCTION`, `MONITORING`, or failure states

---

## Required Evidence

| Step | Evidence |
|------|----------|
| Staging deploy | `deployment_id`, target ref, deploy logs URI |
| Health check | Probe results URI |
| Cutover | Slot switch timestamp, route config ref |
| Production | Post-deploy validation URI |

---

## State Transitions

```
PROMOTION_APPROVED → STAGING → DEPLOYING → PRODUCTION → MONITORING
                              ↓
                       DEPLOYMENT_FAILED → DEPLOYING (retry)
```

---

## Events Emitted

`StagingStarted` → `DeploymentStarted` → `DeploymentSucceeded` | `DeploymentFailed` → `MonitoringStarted`

---

## Failure Modes

| Failure | State | Production impact |
|---------|-------|-------------------|
| Staging deploy error | `DEPLOYMENT_FAILED` | None |
| Health check fail | `DEPLOYMENT_FAILED` | None |
| Cutover timeout | `DEPLOYMENT_FAILED` | Blue unchanged if auto-revert |
| Post-cutover validation fail | `DEPLOYMENT_FAILED` or rollback trigger | May revert |

---

## Recovery Paths

| From | Action |
|------|--------|
| `DEPLOYMENT_FAILED` | `retry_deploy` → `DEPLOYING` |
| `DEPLOYMENT_FAILED` | `revert_to_staging` → `STAGING` |
| Repeated failure | `FAILED` + operator incident |

---

## Health Check (Planned)

Staging: readiness + smoke inference batch.  
Production: readiness + 5m error rate + p99 latency vs policy.

---

## Current Implementation Status

Documentation only. No deployment manager, no runtime targets, no blue/green slots.
