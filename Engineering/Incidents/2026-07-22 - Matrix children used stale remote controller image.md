# 2026-07-22 - Matrix child PipelineRuns used stale BenchFlow image

## Symptom
A matrix was submitted with:

```bash
bflow experiment run ... --benchflow-image ghcr.io/albertoperdomo2/benchflow:main-29cd140
```

The matrix supervisor TaskRun received `main-29cd140`, but child `resolve-run-plan` TaskRuns pulled `ghcr.io/albertoperdomo2/benchflow:main-840a075` and failed before execution.

Affected matrices included `cpu-offloading-matrix-e36da8` and `cpu-offloading-matrix-7d0dd3`.

## Environment
- Management cluster: aperdomo-lab, namespace `benchflow`.
- Requested image: `main-29cd140`.
- Failed child PipelineRuns: `cpu-offloading-3ac929`, `cpu-offloading-70fb49`, `cpu-offloading-aa2c1e`, and `cpu-offloading-e0fe86`.

## Root Cause
Two independent stale-image paths were present.

1. The remote-capacity controller that materializes Kueue-admitted PipelineRuns had been running a floating `:latest` image since 2026-07-17. It was older than the pinned matrix-supervisor image and emitted child manifests using `main-840a075`.
2. The emergency rollback of `benchflow-e2e` used a resolved successful PipelineRun snapshot. Tekton had already substituted the old `BENCHFLOW_IMAGE` value in that snapshot, so applying it baked `main-840a075` into every task `IMAGE` parameter. PipelineRun-level `--benchflow-image` values could no longer override those literals.

## Resolution
Pinned the management controller deployment to the same verified image as the matrix run:

```bash
oc -n benchflow set image deployment/benchflow-remote-capacity-controller \
  controller=ghcr.io/albertoperdomo2/benchflow:main-29cd140
oc -n benchflow rollout status deployment/benchflow-remote-capacity-controller
```

Repaired the active e2e Pipeline by replacing every baked `main-840a075` task image value with `$(params.BENCHFLOW_IMAGE)`. The active `resolve-run-plan` task now has:

```
IMAGE=$(params.BENCHFLOW_IMAGE)
```

Also deleted orphaned Workload `cpu-offloading-d99c77-reservation-178458512365` from 2026-07-20. Its submission ConfigMap no longer existed, but it held two GPU quota units.

## Local Source Validation
Current local source is correct:
- Matrix parent rendering preserves an explicit `BENCHFLOW_IMAGE` parameter.
- Child PipelineRun rendering preserves the same explicit parameter.
- `src/benchflow/` and `tekton/` contain no hard-coded `main-840a075` value.
- `compileall` and Ruff passed.

## Prevention / Runbook
- Do not restore a Pipeline from `PipelineRun.status.pipelineSpec` without replacing resolved parameter values with their original Tekton substitutions.
- When using a development `--benchflow-image` for management-cluster matrix runs, bootstrap or explicitly update `benchflow-remote-capacity-controller` to the same immutable image first.
- Verify a repaired Pipeline task references `$(params.BENCHFLOW_IMAGE)` before retrying.

## Related
- Project: [[BenchFlow]]
- Related: [[2026-07-22 - Kueue webhook no endpoints rejected post-benchmark TaskRuns]]