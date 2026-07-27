# 2026-07-27 — CephFS mount DeadlineExceeded on worker node due to stale monitor IP

## Symptom
Pod `kv-cache-wins-m3-render` on worker node `diadochos-hqxzk-worker-1-8p9vb` failed with:
```
MountVolume.MountDevice failed for volume "pvc-0c02d4b2-..." : rpc error: code = DeadlineExceeded desc = context deadline exceeded
```
CephFS CSI node plugin on the same worker had **27 restarts** and previous container logs showed:
```
an operation with the given Volume ID ... already exists
GRPC error: code = Aborted desc = an operation with the given Volume ID ... already exists
```
Rook tools pod `ceph status` also failed with:
```
monclient(hunting): authenticate timed out after 300
[errno 110] RADOS timed out (error connecting to the cluster)
```

## Environment
- Cluster: diadochos (IBM Cloud, eu-de-2)
- Node: `diadochos-hqxzk-worker-1-8p9vb` (10.243.1.5, worker, no GPUs)
- Rook-Ceph v19.2.4 (Squid), ceph-csi v3.17.0
- PVC: `models-storage` (2500Gi, RWX, `rook-cephfs` StorageClass, CephFS `kvcache-fs`)
- Monitors: e (gpu-h100-fx7c8), k (gpu-h100-gjfjh), l (gpu-h100-6kl5z) — all hostNetwork

## Timeline
- Mon-e was created when node gpu-h100-fx7c8 had IP `10.0.0.6` (secondary/storage interface). Node's Kubernetes-registered InternalIP is `10.243.65.5`.
- At some point the `10.0.0.6` network became unreachable from worker nodes (different subnet: `10.243.1.x` → `10.0.0.x` has no route), while GPU-to-GPU connectivity on that network still works.
- CSI `csi-cluster-config-json` in `rook-ceph-mon-endpoints` ConfigMap listed `10.0.0.6:6789` as a monitor.
- Worker node CSI plugin attempts CephFS mount → kernel client tries `10.0.0.6` → TCP SYN hangs (no RST) → CSI operation holds the per-volume lock → kubelet retries rejected with "operation already exists" → CSI eventually crashes → restart cycle (27 times).
- Additionally, the tools pod `ceph.conf` used ClusterIP service addresses (`172.30.181.75`, `172.30.67.134`, `172.30.41.40`), but only mon-e's service existed. Mon-e's service endpoint pointed to `10.243.65.5:6789` where mon-e is NOT listening (it's on `10.0.0.6`). Result: all three ClusterIP addresses broken → `ceph status` from tools pod fails.
- Rook operator was also stuck in an unrelated OSD "not ok-to-stop" loop because `kvcache-fs-data0` pool has `size 1` (no replication), so no OSD can be safely stopped for rolling restart.

## Root Cause
Mon-e is bound to `10.0.0.6` (a secondary interface on gpu-h100-fx7c8), which is only routable from the GPU-node network, not from worker nodes. The CSI monitor configmap included this unreachable IP. When the kernel CephFS client on the worker tried to connect to this monitor, the TCP SYN hung (no route, no RST), blocking the CSI operation and preventing mount completion.

## Resolution
1. **Patched `rook-ceph-mon-endpoints` ConfigMap** — replaced `10.0.0.6` with `10.243.65.5` in all three fields (`data`, `csi-cluster-config-json`, `mapping`). Mon-e still listens on `10.0.0.6`, so `10.243.65.5:6789` gets connection refused — but this is a **fast fail** (instant RST) instead of a hang (minutes-long SYN timeout). The CSI client immediately falls through to mon-k and mon-l, which are both reachable.
2. **Deleted stuck CSI node plugin pod** (`pqphj`, 27 restarts) on `worker-1-8p9vb`. DaemonSet recreated it (`glc25`) with 0 restarts and clean logs.
3. Verified `ceph status` works with direct pod IPs and cluster is `HEALTH_OK` with all 130 PGs `active+clean`.

## Remaining items
- **Mon-e IP mismatch** — mon-e still listens on `10.0.0.6` (unreachable from workers). The ConfigMap fast-fails instead of hanging, but a proper fix would be to failover mon-e so it rebinds to `10.243.65.5`, or add a route from worker nodes to `10.0.0.6/24`.
- **Missing mon services** — mon-k and mon-l have no ClusterIP services; only mon-e has one (and its endpoint points to the wrong IP). This breaks the tools pod's `ceph.conf`. The Rook operator should recreate these but is stuck in the OSD loop.
- **OSD operator loop** — `kvcache-fs-data0` pool has `size 1 min_size 1`. The operator cannot safely stop any OSD for rolling restart. This is by design for a KV cache pool but prevents the operator from converging. Consider `continueUpgradeAfterChecksEvenIfNotHealthy: true` in the CephCluster CR or temporarily increasing pool size.

## Prevention / Runbook
- When adding worker nodes on a different subnet, verify all Ceph monitor addresses in `rook-ceph-mon-endpoints` ConfigMap are routable from the new subnet.
- For hostNetwork monitors on multi-homed nodes, ensure the Rook `network` configuration in the CephCluster CR specifies the correct address range so monitors bind to the cluster-wide routable interface.
- Pools with `size 1` will block the Rook operator's OSD rolling restart permanently — document this as expected behavior or set `continueUpgradeAfterChecksEvenIfNotHealthy`.

## Related
- Prior incident: [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]
- Prior incident: [[2026-07-25 - CephFS diagnostics forbidden and Ceph pool metrics absent on Diadochos]]