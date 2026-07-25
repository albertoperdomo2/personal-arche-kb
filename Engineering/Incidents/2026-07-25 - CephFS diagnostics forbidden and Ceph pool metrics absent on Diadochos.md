---
title: CephFS diagnostics forbidden and Ceph pool metrics absent on Diadochos
date: 2026-07-25
type: incident
cluster: psap-diadochos-h100
namespace: benchflow
---

# 2026-07-25 - CephFS diagnostics forbidden and Ceph pool metrics absent on Diadochos

## Symptom

The completed CephFS offload run [b3155c819f674f6aaa6086f689a3d182](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/b3155c819f674f6aaa6086f689a3d182?workspace=benchflow) had:

- CephFS diagnostics error: `User system:serviceaccount:benchflow:benchflow-runner cannot get resource pods/log in namespace rook-ceph`.
- Empty `storage_ceph_pool_*`, OSD, and MDS Prometheus series, despite healthy Rook-Ceph pods.
- Valid vLLM, KV-offload, scheduler, and CephFS PVC telemetry. DCGM queries were also empty, but for a separate label-mapping reason.

## Environment

- Target cluster: Diadochos H100.
- BenchFlow namespace and service account: `benchflow/benchflow-runner`.
- Runtime PVC: `vllm-kv-cache` using Rook CephFS.
- Metrics endpoint: in-cluster Thanos Querier.

## Root Cause

The target had been bootstrapped without BenchFlow's dedicated `rook-ceph` diagnostics Role and RoleBinding. It lacked the least-privilege `pods/log` permission required by the artifact collector.

Rook exporter, MDS, OSD, and CSI pods were healthy, but no Rook-Ceph ServiceMonitor or PodMonitor existed. Consequently, Thanos did not scrape the Rook manager's Ceph pool metrics.

This was not Prometheus RBAC: `benchflow-runner` had `cluster-monitoring-view`, could access `prometheuses/api`, and all 102 historical Prometheus queries completed without HTTP failures.

## Resolution

Applied manually to Diadochos:

1. `benchflow-runner-cephfs-diagnostics-benchflow` Role and RoleBinding in `rook-ceph`, granting only pod/event reads, `pods/log` reads, and CephCluster/CephFilesystem reads.
2. `benchflow-rook-ceph-mgr` ServiceMonitor for `service/rook-ceph-mgr` port `http-metrics`, with a 15-second interval.

Validation:

- A real MDS log request as `benchflow-runner` succeeded.
- The exact BenchFlow detailed-profile pool query returned `kvcache-fs-data0` read and write series.

## Prevention

BenchFlow bootstrap now installs the guarded `monitoring/rook-ceph-mgr-servicemonitor.yaml` asset whenever the ServiceMonitor CRD, `rook-ceph` namespace, and `rook-ceph-mgr` service exist. Existing targets need a current `bflow bootstrap` to reconcile the diagnostics RBAC and ServiceMonitor.

DCGM remains a separate observability limitation: the active exporter exposes only exporter pod labels, not workload pod labels, so BenchFlow's workload-filtered DCGM selectors cannot match it.

## Related

- [[2026-07-22 - CephFS diagnostics skipped with stale Tekton task]]
- [[KV Cache Offloading]]