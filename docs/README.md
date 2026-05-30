# Documentation

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Structure

| Path | Contents |
|------|----------|
| `overview/` | Constitution and end-state definition |
| `architecture/` | System design, lifecycle state machine, transitions, events, deployment, promotion |
| `workflows/` | Training, promotion, deployment, drift-retraining, rollback |
| `runbooks/` | Operational procedures |
| `diagrams/` | Lifecycle state machine (Mermaid) |
| `examples/` | Conceptual payload contracts |

## Reading Order

1. [project-constitution.md](overview/project-constitution.md)
2. [system-overview.md](architecture/system-overview.md)
3. [lifecycle-state-machine.md](architecture/lifecycle-state-machine.md) — Session 1 canonical states
4. [transition-rules.md](architecture/transition-rules.md)
5. [event-model.md](architecture/event-model.md)
6. [deployment-strategy.md](architecture/deployment-strategy.md)
7. [promotion-policy-model.md](architecture/promotion-policy-model.md)
8. [model-lifecycle-workflow.md](workflows/model-lifecycle-workflow.md) — Session 0 overview
9. [project-end-state.md](overview/project-end-state.md)

Session 1 workflows: [training-workflow.md](workflows/training-workflow.md), [promotion-workflow.md](workflows/promotion-workflow.md), [deployment-workflow.md](workflows/deployment-workflow.md), [drift-retraining-workflow.md](workflows/drift-retraining-workflow.md), [rollback-workflow.md](workflows/rollback-workflow.md).

Diagram: [lifecycle-state-machine.md](diagrams/lifecycle-state-machine.md).

## Implemented

No runtime behavior is implemented.

## Planned

Session 0–1 architecture documentation complete. Next: API contracts (Session 2).

## Future Work

ADRs, OpenAPI specs derived from event model.
