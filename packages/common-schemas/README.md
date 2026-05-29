# Common Schemas

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Shared event payloads, API request/response models, and lifecycle enums used across all services. Single source of truth for contract versioning.

## Planned Contents

- Lifecycle state enum
- Transition request/response types
- `LifecycleTransitionEvent`, drift, rollback, promotion payloads
- JSON schema or Pydantic models (TBD at implementation)

## Implemented

Nothing.

## Planned

Package scaffold and initial schema definitions aligned with [lifecycle-examples.md](../../docs/examples/lifecycle-examples.md).

## Future Work

- Code generation for client SDKs
- Backward compatibility policy and version negotiation
