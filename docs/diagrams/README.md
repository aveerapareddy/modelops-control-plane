# Diagrams

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Storage for architecture diagrams: system context, lifecycle state machine, deployment topology, and sequence charts for workflows.

Preferred formats: Mermaid (`.md` embed), SVG, or PNG with source files where applicable.

## Implemented

No diagrams yet.

## Planned

- System context diagram (control plane and service boundaries)
- Lifecycle state machine diagram (aligned with [runtime-model.md](../architecture/runtime-model.md))
- Sequence diagram: happy path Train → Monitor
- Sequence diagram: drift → retrain → promote
- Sequence diagram: rollback

## Future Work

- Auto-generated diagrams from state machine definition once implemented
