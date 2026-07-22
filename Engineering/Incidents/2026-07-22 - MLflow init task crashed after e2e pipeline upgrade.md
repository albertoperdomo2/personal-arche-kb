# 2026-07-22 - MLflow init task crashed after e2e pipeline upgrade

## Symptom
After updating the management-cluster `benchflow-e2e` Pipeline to the current source, new PipelineRuns reached `init-mlflow-run` and failed:

```
AttributeError: 'ResolvedDeployment' object has no attribute 'profile'
```

The failure occurred in `benchflow.mlflow_upload._initial_run_tags()`.

## Environment
- Management cluster: aperdomo-lab, namespace `benchflow`.
- Affected PipelineRuns: `cpu-offloading-38ede1`, `cpu-offloading-16b698`.
- Last known-good pre-upgrade PipelineRun: `cpu-offloading-407f77`.

## Root Cause
The current e2e Pipeline introduced the `init-mlflow-run` Task, but the image code still referenced removed `ResolvedDeployment.profile` and `ResolvedDeployment.type` fields. The preceding pipeline graph created the MLflow run during the benchmark task, so the defect was latent until the new initialization task was enabled.

## Resolution
Restored the active `benchflow-e2e` Pipeline to the stored specification of the last successful PipelineRun. This affects only future PipelineRuns and restores the prior working graph.

Fixed the source contract in `src/benchflow/mlflow_upload.py`:
- `deployment_profile` comes from `plan.profiles.deployment`.
- `deployment_type` is `<platform>-<mode>`.

The source fix requires a new BenchFlow image before reapplying the newer Pipeline and MLflow Task graph.

## Prevention / Runbook
Do not apply a Pipeline revision that activates a new Task path until that path has been exercised with the exact referenced image. In particular, validate a small PipelineRun after any change that introduces Task references or changes the `RunPlan` contract.

## Related
- Project: [[BenchFlow]]
- Related: [[2026-07-22 - benchflow-e2e missing MLflow Tekton tasks]]