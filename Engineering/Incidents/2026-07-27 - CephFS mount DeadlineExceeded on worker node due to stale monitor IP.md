# 2026-07-27 — CephFS mount DeadlineExceeded on worker node due to unreachable Ceph public network

## Symptom
Pod `kv-cache-wins-m3-render` (and later `kv-cache-wins-m1-render`, `kv-cache-wins-m2-render`) on worker node `diadochos-hqxzk-worker-1-8p9vb` failed with:
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
- CephCluster `network.public: 10.0.0.0/16` (original)
- Monitors: e (gpu-h100-fx7c8), k (gpu-h100-gjfjh), l (gpu-h100-6kl5z) — all hostNetwork
- MDS: all 4 daemons on gpu-h100-gjfjh
- Dual-homed GPU nodes: Kubernetes IPs on `10.243.65.0/24`, secondary storage interface on `10.0.0.0/16`
- Worker nodes on `10.243.1.0/24`: can reach `10.243.65.0/24` but NOT `10.0.0.0/16`

## Root Cause
The CephCluster CR had `spec.network.addressRanges.public: ["10.0.0.0/16"]`. All Ceph daemons (monitors, MDS, OSDs) bound to the `10.0.0.x` interface — a secondary/storage network on the GPU nodes. This network is **only routable between GPU nodes**, not from worker nodes on the `10.243.1.x` subnet.

The mount failure chain:
1. CSI configmap had `10.0.0.6` for mon-e → unreachable from worker → TCP SYN hangs → CSI operation lock stuck → kubelet retries rejected → CSI crash loop (27 restarts)
2. Even after fixing monitor addresses, MDS daemons were bound to `10.0.0.8` (unreachable from worker)
3. OSDs were also bound to `10.0.0.x` addresses
4. The kernel CephFS client connects to monitors → gets MDS/OSD addresses on `10.0.0.x` → can't reach them → mount hangs → DeadlineExceeded

Additional issues:
- Mon-e's ClusterIP service endpoint pointed to `10.243.65.5` where mon-e is NOT listening (daemon bound to `10.0.0.6`) — connection refused
- Mon-k and mon-l had no ClusterIP services at all
- The tools pod's `ceph.conf` used three broken ClusterIP addresses
- Rook operator was stuck in an OSD "not ok-to-stop" loop because `kvcache-fs-data0` pool has `size 1` (no replication)
- CSI `ceph-csi-config` ConfigMap is separate from `rook-ceph-mon-endpoints` — must patch both

## Resolution

### 1. Monitor ConfigMap fix
Patched `rook-ceph-mon-endpoints` ConfigMap — replaced `10.0.0.6` with `10.243.65.5` for mon-e in all three fields (`data`, `csi-cluster-config-json`, `mapping`). Fast-fail (connection refused) instead of TCP hang.

### 2. OSD rolling restart
Enabled `skipUpgradeChecks: true` on CephCluster CR to bypass the `ok-to-stop` check (blocked by `size 1` pool). All 21 OSDs rolled. Disabled `skipUpgradeChecks` afterward.

### 3. CephCluster public network change (root cause fix)
Changed `spec.network.addressRanges.public` from `["10.0.0.0/16"]` to `["10.243.65.0/24"]`. This makes all Ceph daemons bind to Kubernetes-routable IPs instead of the secondary storage interface.

After the patch, force-restarted all MDS and OSD deployments:
```bash
oc rollout restart deployment -n rook-ceph -l app=rook-ceph-mds
oc rollout restart deployment -n rook-ceph -l app=rook-ceph-osd
```

Verified:
- MDS rebinding to `10.243.65.15` (routable)
- OSDs rebinding to `10.243.65.x` (routable)
- Cluster HEALTH_OK, all 21 OSDs up, 130 PGs active+clean

### 4. CSI config fix
Patched `ceph-csi-config` ConfigMap (the one CSI pods actually read) to remove mon-e (`10.243.65.5`) since the daemon is NOT listening on that IP. Only mon-k (`10.243.65.15`) and mon-l (`10.243.65.9`) remain. 2/3 monitors is sufficient for quorum.

### 5. Stale state cleanup
- Deleted CSI node plugin pod on worker (multiple times to clear stuck operation locks)
- Removed stale staging directory from `/var/lib/kubelet/plugins/kubernetes.io/csi/`
- VolumeAttachment on worker cleaned up automatically after workload removal

### Verification
- Manual `mount -t ceph` from worker debug pod: **SUCCESS** — mounted, listed files, unmounted
- Manual `mount -t ceph` from inside CSI pod: **SUCCESS**
- Ceph cluster HEALTH_OK with all daemons on routable IPs

### Remaining: mon-e monmap address
Mon-e is still registered at `10.0.0.6` in the Ceph monmap. The daemon binds to the monmap address, not the ConfigMap address. Changing a monitor's IP in the monmap requires the Rook operator's monitor failover process. Since 2/3 monitors is sufficient and mon-e was removed from the CSI config, this is non-blocking. Future fix: let Rook detect the IP mismatch and replace mon-e, or manually update the monmap.

## Prevention / Runbook
- When adding worker nodes on a different subnet, verify ALL Ceph daemon addresses (monitors, MDS, OSDs) are routable from the new subnet — not just the monitors. Use `ceph fs dump` and `ceph osd find <id>` to check MDS and OSD addresses.
- The `network.public` CIDR in the CephCluster CR controls which interface Ceph daemons bind to. If the cluster serves multiple subnets, this CIDR must include the routable address range.
- The CSI reads monitors from `ceph-csi-config` ConfigMap, NOT `rook-ceph-mon-endpoints`. The Rook operator regenerates `ceph-csi-config` from `rook-ceph-mon-endpoints`, so manual patches to `ceph-csi-config` may be overwritten.
- Pools with `size 1` block the Rook operator's OSD rolling restart permanently. Use `skipUpgradeChecks: true` temporarily.
- The CSI per-volume operation lock can get permanently stuck if a mount hangs and the context deadline passes. Deleting the CSI node plugin pod clears the in-memory lock. Also remove stale staging directories.
- Monitor IP changes in the monmap require Rook's failover process — simply restarting the monitor pod does not change its bound address.

## Related
- Prior incident: [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]
- Prior incident: [[2026-07-25 - CephFS diagnostics forbidden and Ceph pool metrics absent on Diadochos]]

## 2026-07-28 Recurrence: precise-prefix renderer scheduled to affected worker

A new `kv-cache-wins-m1` precise-prefix run reproduced the same worker-local failure:

- Renderer pod `kv-cache-wins-m1-render-...-6d4fj` was scheduled to `diadochos-hqxzk-worker-1-8p9vb`.
- It remained `ContainerCreating` for more than 14 minutes.
- The `models-storage` CephFS PVC mounted successfully on the two renderer pods scheduled to `gpu-h100-fx7c8`.
- The affected worker's `csi-cephfsplugin` had restarted 10 times. Its prior logs repeatedly returned `Aborted: an operation with the given Volume ID ... already exists`; pod events also showed `DeadlineExceeded ... RST_STREAM ... CANCEL`.

This confirms the PVC and renderer image are not the differentiator. The failure is isolated to the worker's CSI mount path, and the stale operation lock is a recurrence of the known worker-node CephFS failure mode.

### BenchFlow mitigation

The llm-d precise-prefix renderer overlay previously mounted the model PVC but did not copy `runtime.node_selector`, `runtime.affinity`, or `runtime.tolerations`. Model servers were constrained to healthy GPU nodes while renderer replicas could land on arbitrary workers. BenchFlow now propagates those runtime scheduling constraints to the renderer Deployment, ensuring renderers use the same selected node set as the model servers.

This is a workload-scheduling mitigation, not a replacement for repairing the affected worker's CephFS CSI plugin and underlying network/storage reachability.