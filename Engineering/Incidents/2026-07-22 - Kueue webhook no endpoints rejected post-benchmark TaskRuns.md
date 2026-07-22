# 2026-07-22 — Kueue webhook no endpoints rejected post-benchmark TaskRuns

## Symptom
BenchFlow matrix parent `cpu-offloading-matrix-0e8dc9` failed after its first child. The child `cpu-offloading-1e966e` had already deployed, reached its endpoint, and completed the benchmark, but its post-benchmark TaskRuns failed with:

```
failed to create task run pod "...-collect-artifacts": Internal error occurred:
failed calling webhook "mpod.kb.io": failed to call webhook:
Post "https://kueue-webhook-service.kueue-system.svc:443/mutate--v1-pod?timeout=10s":
no endpoints available for service "kueue-webhook-service"
```

The matrix supervisor then failed creating child 2's Kueue Workload with the same service-endpoint error.

## Environment
- Management cluster: `aperdomo-lab`, namespace `benchflow`
- Target cluster: `psap-h100-diadochos`
- Kueue: `registry.k8s.io/kueue/kueue:v0.16.4`
- BenchFlow image: `ghcr.io/albertoperdomo2/benchflow:main-eb00d74`
- Run started: 2026-07-22 07:59:48Z

## Timeline
- 08:00Z: first child `cpu-offloading-1e966e` admitted and started.
- 08:44:20Z: Kueue controller lost leader election and exited.
- 08:44:49Z: post-benchmark TaskRuns began; pod creation was rejected because the Kueue webhook service had no endpoints.
- 08:44:57Z: matrix supervisor attempted child 2; Kueue Workload creation failed for the same reason.
- 08:44:52Z to 08:45:05Z: Kueue restarted and the webhook endpoint returned.

## Root Cause
The single Kueue controller-manager lost its Kubernetes API leader lease after API requests timed out:

```
Failed to renew lease ... context deadline exceeded
Could not run manager: leader election lost
```

Its only webhook endpoint disappeared during the restart. The Kueue mutating webhook is failure-policy blocking for Pods and Workloads, so Kubernetes rejected both the child's post-benchmark TaskRun Pods and the matrix supervisor's next Workload. This was a management-cluster control-plane outage, not a Diadochos GPU, model, CephFS, or BenchFlow deployment failure.

## Resolution
Kueue recovered automatically: `kueue-controller-manager-8bd4f7c6-rlq9j` became Ready and `kueue-webhook-service` again had endpoint `10.129.0.102:9443`.

Read-only verification commands:

```bash
oc --kubeconfig /Users/aperdomo/workspace/redhat/clusters/aperdomo-lab/auth/kubeconfig \
  -n kueue-system get endpoints kueue-webhook-service -o yaml
oc --kubeconfig /Users/aperdomo/workspace/redhat/clusters/aperdomo-lab/auth/kubeconfig \
  -n kueue-system logs kueue-controller-manager-8bd4f7c6-rlq9j --previous --tail=120
```

The child cleanup TaskRun never created a pod. `cpu-offloading-m1` remains deployed on Diadochos and must be cleaned up deliberately before re-running.

## Prevention / Runbook
- Treat `no endpoints available for service "kueue-webhook-service"` as a management-cluster Kueue outage.
- Before submitting or retrying a matrix, require the controller-manager Pod to be Ready and the webhook service to have at least one endpoint.
- Investigate management API-server latency/availability when Kueue logs show leader-lease update timeouts; Kueue's singleton controller makes the webhook unavailable during a restart.
- A failed post-benchmark collection/cleanup means the benchmark is incomplete even if the benchmark TaskRun succeeded.

## BenchFlow Follow-up
BenchFlow commit `c5eac91` isolates per-child submission errors in the matrix supervisor. A transient API or Kueue rejection now records that child as failed and continues submitting later matrix children; the parent reports the aggregate failure only after every child was attempted. This does not mask the Kueue outage, but prevents it from suppressing independent remaining experiments.

## Related
- Project: [[BenchFlow]]
- Related execution: `cpu-offloading-matrix-0e8dc9`, child `cpu-offloading-1e966e`