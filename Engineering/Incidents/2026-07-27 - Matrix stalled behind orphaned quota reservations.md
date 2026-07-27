# Matrix stalled behind orphaned quota reservations

## Symptom

The matrix PipelineRun `kv-cache-wins-matrix-73bf71` submitted four child reservations, but only `kv-cache-wins-b63da4` ran. After it completed, the matrix supervisor remained running and the other three children stayed queued with:

```
target cluster psap-h100-diadochos is locked to rhoai:RHOAI-3.5.0-ea.2
```

The target cluster's `benchflow-platform-state` was correctly `llm-d:v0.8.0:istio`.

## Environment

- Management cluster namespace: `benchflow`
- Target cluster: `psap-h100-diadochos` / Diadochos
- Affected matrix: `kv-cache-wins-matrix-73bf71`
- Date: 2026-07-27

## Root Cause

Three old Mooncake RHOAI reservation Workloads were still `spec.active: true` with Kueue quota reservations and `benchflow-remote-capacity` stuck `Pending`:

- `mooncake-offloading-64g-7e9360-reservation-178476218463`
- `mooncake-roce-64g-63167c-reservation-178482149988`
- `mooncake-roce-64g-e6c1d6-reservation-178482171674`

Their originating PipelineRuns and submission ConfigMaps had both been removed. The remote-capacity controller therefore treated the orphaned quota reservations as active RHOAI setup-key holders and blocked the llm-d wave.

Kueue uses `status.admission` for quota reservation before the controller creates a PipelineRun. It is not safe to wait for `Admitted=True` in this BenchFlow flow because the controller's requeue step resets the admission check to `Pending`.

## Resolution

1. Confirmed the three submission ConfigMaps were absent.
2. Deleted only the three confirmed orphaned Workloads.
3. The next child, `kv-cache-wins-41188e`, was immediately admitted and started.
4. Updated `src/benchflow/kueue.py` so the remote-capacity controller releases any reservation whose PipelineRun and submission ConfigMap are both absent before it participates in setup-key locking.

## Prevention

Keep the quota-reservation setup-key behavior, but reclaim orphaned reservations during every controller reconciliation. Do not replace the reservation check with Kueue `Admitted=True`; that condition is not stable across BenchFlow's requeue-based admission flow.

## Validation

- `ruff check src/benchflow/kueue.py`
- `ruff format --check src/benchflow/kueue.py`
- `python3 -m compileall -q src/benchflow/kueue.py`
- Focused quota-reservation behavior check passed.
- Live verification: `kv-cache-wins-41188e` was running after orphan cleanup.

## Related

- [[2026-07-22 - Matrix children used stale remote controller image]]