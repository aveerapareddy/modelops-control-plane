# Operator Console

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Minimal operational UI for lifecycle visibility, promotion approval queue, and rollback confirmation. Platform tool for actions—not a vanity metrics dashboard.

## Boundaries

| Owns | Does not own |
|------|--------------|
| Presentation of lifecycle state | Lifecycle state authority |
| Operator-initiated API calls | Deep metrics visualization (link to Grafana) |
| Approval and rollback UX | Drift algorithm implementation |

## Implemented

Nothing.

## Planned

Web UI consuming control plane and registry read APIs; privileged mutations via control plane only.

## Future Work

- SSO integration
- Bulk operations across model families
