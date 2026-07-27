# 2026-07-27 — CephFS mount DeadlineExceeded on worker node due to unreachable Ceph public network

## Symptom
Pod `kv-cache-wins-m3-render` (and later `kv-cache-wins-m1-render`) on worker node `diadochos-hqxzk-worker-1-8p9vb` failed with:
```
MountVolume.MountDevice failed for volume "pvc-0c02d4b2-..." : rpc error: code = DeadlineExceeded desc = context deadline exceeded
```
CephFS CSI node plugin on the same worker had **27 restarts** and logs showed:
```
an operation with the given Volume ID ... already exists
GRPC error: code = Aborted desc = an operation with the given Volume ID ... already exists
```
Manual `mount -t ceph` from the worker also failed:
```
mount error 110 = Connection timed out
```

## Environment
- Cluster: diadochos (IBM Cloud, eu-de-2)
- Node: `diadochos-hqxzk-worker-1-8p9vb` (10.243.1.5, worker, no GPUs, 20GB RAM)
- Rook-Ceph v19.2.4 (Squid), ceph-csi v3.17.0
- PVC: `models-storage` (2500Gi, RWX, `rook-cephfs` StorageClass, CephFS `kvcache-fs`)
- CephCluster `network.public: 10.0.0.0/16`
- Monitors: e (gpu-h100-fx7c8, 10.0.0.6), k (gpu-h100-gjfjh, 10.243.65.15), l (gpu-h100-6kl5z, 10.243.65.9) — all hostNetwork
- MDS: all 4 daemons on gpu-h100-gjfjh, bound to `10.0.0.8:6800-6807`

## Root Cause
The CephCluster CR has `network.public: 10.0.0.0/16`. All Ceph daemons (monitors, MDS, OSDs) bind to the `10.0.0.x` interface — a secondary/storage network on the GPU nodes. This network is **only routable between GPU nodes**, not from worker nodes on the `10.243.1.x` subnet.

The mount failure chain:
1. CSI configmap originally had `10.0.0.6` for mon-e → unreachable from worker → TCP SYN hangs → CSI operation lock stuck → kubelet retries rejected → CSI crash loop (27 restarts)
2. After fixing the configmap (mon-e → `10.243.65.5`), the kernel CephFS client connects to monitors at `10.243.65.15` and `10.243.65.9` (their Kubernetes node IPs, reachable)
3. Monitors direct the client to the MDS at `10.0.0.8` (the Ceph public network address)
4. Worker node cannot reach `10.0.0.8` → mount hangs → `ETIMEDOUT` after 30s
5. CephFS will never mount from worker nodes while MDS/OSD addresses are on `10.0.0.0/16`

Additional issues found during investigation:
- Mon-e's ClusterIP service endpoint pointed to `10.243.65.5` where mon-e is NOT listening (it's on `10.0.0.6`) — connection refused
- Mon-k and mon-l had no ClusterIP services at all
- The tools pod's `ceph.conf` used three broken ClusterIP addresses — all three unreachable
- Rook operator was stuck in an OSD "not ok-to-stop" loop because `kvcache-fs-data0` pool has `size 1` (no replication)

## Resolution
Partial — the monitor-level issues were fixed, but the MDS reachability issue remains:

### Applied
1. **Patched `rook-ceph-mon-endpoints` ConfigMap** — replaced `10.0.0.6` with `10.243.65.5` for mon-e. Fast-fail (connection refused) instead of hang.
2. **Deleted stuck CSI node plugin pod** on `worker-1-8p9vb` — cleared operation lock.
3. **Enabled `skipUpgradeChecks: true`** on CephCluster CR to unblock the OSD rolling restart (all 21 OSDs rolled successfully). Disabled it afterward.

### Not resolved — CephFS unmountable from worker nodes
The MDS daemons are bound to `10.0.0.8` (the `10.0.0.0/16` public network). Worker nodes cannot reach this address. Fix options:
1. **Network fix**: add a route from worker subnet to `10.0.0.0/16`
2. **CephCluster network fix**: change public network CIDR to include `10.243.0.0/16` so daemons rebind to Kubernetes-routable IPs (requires full Ceph daemon restart)
3. **Scheduling fix**: restrict CephFS-dependent pods to GPU nodes via nodeSelector/affinity

## Prevention / Runbook
- When adding worker nodes on a different subnet, verify ALL Ceph daemon addresses (monitors, MDS, OSDs) are routable from the new subnet — not just the monitors. Use `ceph fs dump` and `ceph osd find <id>` to check MDS and OSD addresses.
- The `network.public` CIDR in the CephCluster CR controls which interface Ceph daemons bind to. If the cluster serves multiple subnets, this CIDR must include the routable address range.
- Pools with `size 1` block the Rook operator's OSD rolling restart permanently.

## Related
- Prior incident: [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]
- Prior incident: [[2026-07-25 - CephFS diagnostics forbidden and Ceph pool metrics absent on Diadochos]]