# Services

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Microservice boundaries for ModelOps Control Plane. Each subdirectory is an independently deployable service with a single domain responsibility.

## Services

| Directory | Role |
|-----------|------|
| `control-plane/` | Lifecycle state machine and audit |
| `training-orchestrator/` | Training job submission and status |
| `registry-service/` | Model version and artifact metadata |
| `deployment-manager/` | Staging and production deployment |
| `monitoring-service/` | Drift and health signals |
| `operator-console/` | Operator UI |

## Implemented

Nothing.

## Planned

Per-service implementation following architecture docs in `docs/`.
