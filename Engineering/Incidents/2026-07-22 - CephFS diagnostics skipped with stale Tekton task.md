# 2026-07-22 - CephFS diagnostics skipped despite enabled profile

## Symptom
A benchmark using the `multi-tier-offloading-cephfs` deployment profile completed successfully and uploaded artifacts to MLflow, but no `logs/cephfs/` or `platform-state/cephfs/` artifacts were present.

The run metadata recorded:

```json
"cephfs_diagnostics": {
  "enabled": true,
  "benchmark_start_time": "",
  "benchmark_end_time": "",
  "log_files": 0,
  "state_files": 0,
  "failures": [
    "CephFS diagnostics require valid benchmark start and end timestamps"
  ]
}
```

Affected MLflow run: [1cd063d289f1456da4507382fe284df7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/1cd063d289f1456da4507382fe284df7?workspace=benchflow).

## Environment
- Management cluster: aperdomo-lab, namespace `benchflow`.
- Target: Diadochos H100, CephFS runtime PVC `vllm-kv-cache`.
- PipelineRun: `cpu-offloading-407f77`.
- BenchFlow image: `ghcr.io/albertoperdomo2/benchflow:main-840a075`.

## Root Cause
The benchmark task correctly published `benchmark-start-time=2026-07-21T13:48:51Z` and `benchmark-end-time=2026-07-21T14:23:41Z`, but the management cluster still used the pre-`840a075` `collect-artifacts` Tekton Task. Its recorded TaskRun command included neither `--benchmark-start-time` nor `--benchmark-end-time`.

The image and Tekton task assets were therefore version-skewed. The collector was new enough to enable CephFS diagnostics but received empty timestamps and deliberately skipped capture.

## Resolution
Re-bootstrap the management cluster from the BenchFlow revision that includes `840a075`, using the desired BenchFlow image. Verify the installed task before a CephFS run:

```bash
oc -n benchflow get task collect-artifacts -o yaml
```

Its `collect` step must expose `BENCHMARK_START_TIME` and `BENCHMARK_END_TIME` parameters and pass both corresponding CLI flags.

The affected historical run cannot be made complete by re-bootstrap; it needs a new benchmark run (or an explicit retrospective log capture while cluster log retention still permits it).

## Prevention / Runbook
Whenever a BenchFlow image adds or changes Tekton task or pipeline parameters, apply the same source revision with `bflow bootstrap` to the management namespace before launching a run. Do not rely on the image tag alone to update Tekton Task/Pipeline definitions.

## Related
- Project: [[BenchFlow]]
- Research: [[KV Cache Offloading]]