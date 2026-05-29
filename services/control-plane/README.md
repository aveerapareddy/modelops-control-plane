# Control Plane

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Authoritative lifecycle state machine and transition API. Validates all state changes, persists lifecycle and audit data, coordinates workflow events to downstream services.

## Boundaries

| Owns | Does not own |
|------|--------------|
| Lifecycle state per `version_id` | Model artifact storage |
| Transition validation | Training execution |
| Audit append for transitions | Inference serving |
| Idempotency and concurrency control | Drift statistics computation |

## Planned Interfaces

- REST (or gRPC) transition API
- Event emission on successful transitions
- Read API for current state and transition history

## Documentation

- [system-overview.md](../../docs/architecture/system-overview.md)
- [runtime-model.md](../../docs/architecture/runtime-model.md)

## Implemented

Nothing.

## Planned

Service scaffold, state machine, persistence layer, transition API.

## Future Work

- HA replication
- Event bus publisher
