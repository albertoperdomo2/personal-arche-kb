# 2026-07-22 - benchflow-e2e could not run because init-mlflow-run was absent

## Symptom
Tekton rejected a BenchFlow PipelineRun before execution:

```
Pipeline benchflow/benchflow-e2e can't be Run; it contains Tasks that don't exist:
Couldn't retrieve Task "init-mlflow-run": tasks.tekton.dev "init-mlflow-run" not found
```

## Environment
- Management cluster: aperdomo-lab, namespace `benchflow`.
- Pipeline: `benchflow-e2e`.

## Root Cause
The installed pipeline referenced `init-mlflow-run`, `publish-matrix-result`, and `finalize-mlflow-run`, but those three Task resources were absent. The remaining pipeline Task references were present. This was an incomplete Tekton asset bootstrap.

## Resolution
Applied the matching task manifests from the BenchFlow source tree:

```bash
oc -n benchflow apply \
  -f tekton/tasks/common/init-mlflow-run.yaml \
  -f tekton/tasks/common/publish-matrix-result.yaml \
  -f tekton/tasks/common/finalize-mlflow-run.yaml
```

Confirmed all three resources exist afterward.

## Prevention / Runbook
Use `bflow bootstrap` when updating management-cluster execution assets. If a pipeline cannot be admitted because a Task is missing, compare `Pipeline.spec.tasks[*].taskRef.name` and `Pipeline.spec.finally[*].taskRef.name` with `oc -n benchflow get task`, then restore the missing source manifests.

## Related
- Project: [[BenchFlow]]
- Related: [[2026-07-22 - CephFS diagnostics skipped with stale Tekton task]]