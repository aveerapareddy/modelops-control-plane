# Training Workflow

**Status:** Architecture Phase  
**Implementation Status:** Not Started  
**Session:** 1 - Lifecycle Runtime Semantics

---

## Implemented

No runtime behavior is implemented.

---

## Planned

End-to-end training and evaluation handoff per [transition-rules.md](../architecture/transition-rules.md).

---

## Future Work

- Distributed training adapter
- Hyperparameter sweep as sub-workflow

---

## Purpose

Move a model version from validated training inputs through training, evaluation, and registration eligibility.

---

## Actors

| Actor | Role |
|-------|------|
| `user:operator` / CI | Submit training, retry failures |
| `service:training-orchestrator` | Execute job, report status |
| `service:control-plane` | State transitions, eval gates |
| `service:registry-service` | Persist version on register |

---

## Inputs

- `model_id`, `model_version_id`
- Data manifest URI, `config_hash`
- Training job spec (resources, entrypoint ref)

---

## Outputs

- `run_id`
- Training metrics artifact
- `evaluation_id` with pass/fail
- Lifecycle state: `REGISTERED` or failure state

---

## Required Evidence

| Stage | Evidence |
|-------|----------|
| Start | Data manifest validation record |
| Complete | `run_id`, artifact staging URI |
| Eval pass | `evaluation_id`, metrics URI, `passed=true` |
| Eval fail | metrics URI, `passed=false` |
| Fail | `reason`, error classification |

---

## State Transitions

```
DATA_PREPARED → TRAINING → EVALUATING → REGISTERED
                 ↓            ↓
          TRAINING_FAILED  EVALUATION_FAILED
                 ↓            ↓
              TRAINING      TRAINING (retry)
```

---

## Events Emitted

`DataPrepared` → `TrainingStarted` → `TrainingCompleted` | `TrainingFailed` → `EvaluationStarted` → `EvaluationCompleted` | `EvaluationFailed` → `ModelRegistered` (on pass)

See [event-model.md](../architecture/event-model.md).

---

## Failure Modes

| Failure | State | Impact |
|---------|-------|--------|
| Job OOM / timeout | `TRAINING_FAILED` | No artifact |
| Invalid config | `TRAINING_FAILED` | Retry after fix |
| Eval gate fail | `EVALUATION_FAILED` | Not registered |
| Orchestrator unreachable | No transition | Remain `DATA_PREPARED` or `TRAINING` |

---

## Recovery Paths

| From | Action |
|------|--------|
| `TRAINING_FAILED` | `retry_training` → `TRAINING` |
| `EVALUATION_FAILED` | `retrain_after_eval_fail` → `TRAINING` |
| Either | `abandon_version` → `FAILED` |

---

## Current Implementation Status

Documentation only. No orchestrator, no training jobs, no eval gate code.
