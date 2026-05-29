# Training Orchestrator

**Status:** Architecture Phase  
**Implementation Status:** Not Started

## Purpose

Submit training jobs to external workers or orchestration engines, track run status, and report completion or failure to the control plane for lifecycle transitions (`TRAINING`, `EVALUATING`, `FAILED`).

## Boundaries

| Owns | Does not own |
|------|--------------|
| Training job specs and run status | Lifecycle state mutations (requests only) |
| Adapter to Airflow/Prefect/subprocess | Model registration metadata |
| `config_hash` capture at submit | Evaluation metric definitions |

## Implemented

Nothing.

## Planned

Job submission API, status callbacks, orchestration engine adapter interface.

## Future Work

- GPU queue integration
- Hyperparameter sweep coordination
