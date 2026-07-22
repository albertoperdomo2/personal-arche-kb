# 2026-07-22 - Matrix child PipelineRuns used stale BenchFlow image

## Symptom
A matrix was submitted with:

```bash
bflow experiment run ... --benchflow-image ghcr.io/albertoperdomo2/benchflow:main-29cd140
```

The matrix supervisor TaskRun received `main-29cd140`, but created child PipelineRuns used `ghcr.io/albertoperdomo2/benchflow:main-840a075`. Child `resolve-run-plan` tasks then failed because that older tag could not be pulled.

## Environment
- Management cluster: aperdomo-lab, namespace `benchflow`.
- Matrix: `cpu-offloading-matrix-e36da8`.
- Affected children: `cpu-offloading-3ac929`, `cpu-offloading-70fb49`.
- Requested image: `main-29cd140`.

## Root Cause
The remote-capacity controller that materializes Kueue-admitted PipelineRuns had been running the floating `:latest` image since 2026-07-17. It was older than the pinned image used by current matrix supervisor tasks and emitted the stale `main-840a075` child image.

The requested image propagated correctly into the matrix parent and supervisor Pod. The stale controller was the divergent execution component.

## Resolution
Pinned the management controller deployment to the same verified image as the matrix run:

```bash
oc -n benchflow set image deployment/benchflow-remote-capacity-controller \
  controller=ghcr.io/albertoperdomo2/benchflow:main-29cd140
oc -n benchflow rollout status deployment/benchflow-remote-capacity-controller
```

Also deleted orphaned Workload `cpu-offloading-d99c77-reservation-178458512365` from 2026-07-20. Its submission ConfigMap no longer existed, but it still held two GPU quota units and caused the restarted controller to retry it.

## Prevention / Runbook
When using a development `--benchflow-image` for management-cluster matrix runs, bootstrap or explicitly update `benchflow-remote-capacity-controller` to the same immutable image first. Do not leave the controller at a floating `:latest` tag while run tasks use a newer pinned image.

## Related
- Project: [[BenchFlow]]
- Related: [[2026-07-22 - Kueue webhook no endpoints rejected post-benchmark TaskRuns]]