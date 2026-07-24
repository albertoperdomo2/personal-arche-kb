---
title: CephFS performance tuning for KV cache offloading
date: 2026-07-24
type: learning
---

# CephFS Performance Tuning for KV Cache Offloading

When using CephFS as a secondary KV-cache offloading tier for vLLM's `TieringOffloadingSpec`, default Rook-Ceph configurations cause severe performance regressions: 7–40% throughput loss versus CPU-only, 66K–127K `cannot store blocks` warnings per run, and 51% token recomputation instead of the 11% seen with local NVMe.

## Root causes identified (diadochos cluster, 2026-07-24)

Six reinforcing problems in the default Rook deployment:

### 1. 3× replication on ephemeral data (dominant factor)

Every KV block write was replicated to 3 NVMe drives across 3 nodes. For a cache that is ephemeral, recomputable from the model, and deleted between runs, this tripled write I/O and added cross-node network latency to every store operation. This is the single biggest contributor to write latency versus local hostPath NVMe (which has zero replication).

**Fix:** `replicated.size: 1` on the data pool. Metadata pool stays at `size: 3`.

### 2. OSD memory starvation

OSDs limited to 8 GiB with the default `osd_memory_target` of 4 GiB. BlueStore cache autotuning was squeezed, causing excessive metadata reads from disk during I/O.

**Fix:** Raise OSD pod limits to 12 GiB, set `osd_memory_target` to 8 GiB via `ceph config set`.

### 3. Under-resourced single MDS

Only 1 active MDS with 500m–2 CPU and 1–4 GiB memory. MDS is single-threaded and CPU-bound. The KV cache workload creates/deletes millions of small block files — a metadata-intensive pattern that saturates a single MDS.

**Fix:** Increase to 2 active MDS, raise resources to 2–4 CPU / 4–8 GiB, set `mds_cache_memory_limit` to 4 GiB.

### 4. No CephFS client mount options

Default `atime` updates generated unnecessary write traffic on every read. Default `rsize`/`wsize` buffers were untuned. No readahead optimization.

**Fix:** Add StorageClass `mountOptions: [noatime, nodiratime, wsize=16777216, rsize=16777216]`. Set `client_readahead_max_bytes` to 32 MiB and `client_cache_size` to 65536 via `ceph config set`.

### 5. vLLM FS tier thread defaults

The `--kv-transfer-config` did not specify `n_read_threads` or `n_write_threads`, using vLLM defaults. For a high-latency backend like CephFS (5–50 ms per operation vs <1 ms for local NVMe), many more I/O threads are needed to pipeline concurrent operations and hide latency.

**Fix:** Set `n_read_threads: 64`, `n_write_threads: 32` in the secondary tier config.

### 6. Squid v19 `bdev_async_discard_threads` bug

Ceph Squid v19.2.0–v19.2.1 has a known bug where `bdev_async_discard_threads > 1` causes high OSD CPU consumption.

**Fix:** `ceph config set osd bdev_async_discard_threads 1`.

## The cascade mechanism

CephFS write latency → CPU primary tier fills faster than it drains → vLLM starts refusing new block stores ("cannot store blocks") → refused stores force token recomputation → GPU loaded with compute instead of serving cached KV → increased contention → more evictions → positive feedback loop. One request was retried 4,082 times in a single run.

## Changes applied (2026-07-24)

All changes applied to the diadochos cluster manifests in `clusters/psap-diadochos-h100/rook-ceph/` and live on the cluster.

| File | Change |
|------|--------|
| `02-ceph-cluster.yaml` | OSD resources: 2–4 CPU, 8–12 GiB |
| `03-cephfs.yaml` | Data pool `size: 1`; MDS activeCount 2, resources 2–4 CPU / 4–8 GiB; StorageClass mountOptions |
| `RUNBOOK.md` | Added Step 5 (runtime `ceph config set` commands), fixed OSD count 24→21, updated embedded YAML examples |

BenchFlow deployment profile change:

| File | Change |
|------|--------|
| `profiles/deployment/rhoai/multi-tier-offloading-cephfs.yaml` | `n_read_threads: 64`, `n_write_threads: 32` |

## Runtime commands applied via tools pod

```bash
ceph config set osd osd_memory_target 8589934592          # 8 GiB
ceph config set osd bdev_async_discard_threads 1           # Squid bug workaround
ceph config set osd ms_async_op_threads 5                  # Network I/O threads (was 3)
ceph config set mds mds_cache_memory_limit 4294967296      # 4 GiB
ceph config set client client_readahead_max_bytes 33554432 # 32 MiB
ceph config set client client_cache_size 65536
ceph osd pool set kvcache-fs-data0 size 1 --yes-i-really-mean-it
ceph osd pool set kvcache-fs-data0 min_size 1
ceph health mute POOL_NO_REDUNDANCY
```

## Networking investigation: 200G NICs for Ceph (2026-07-24)

### Cluster network topology

- **OVN overlay**: Geneve tunnels on `enp3s0` (10G management NIC) via `br-ex`, using 10.243.65.0/24 node IPs, MTU ~1400
- **200G NICs**: 8× ConnectX-7 per node (`enp163s0`–`enp233s0`), each on separate /16 subnets (10.0.0.0/16 through 10.7.0.0/16), linked at 200 Gb/sec (4X HDR), MTU 9000 set on enp163s0
- **CephFS kernel client**: Runs in the host network namespace — must reach OSD pods directly. This is the key constraint that makes all approaches fail.

### Approaches tested — all failed

**1. Multus ipvlan L2 (NAD: `ceph-public-200g` on `enp163s0`)**

Created a `NetworkAttachmentDefinition` to give Ceph pods a second interface on the 200G NIC.

**Result: FAILED.** Host cannot ARP-resolve its own ipvlan children — fundamental Linux kernel limitation. The host sends ARP for the ipvlan child IP and gets zero responses. This breaks the CephFS kernel client → OSD path.

Also tested:
- **ipvlan L3 mode**: Same limitation at a different layer
- **Separate subnet (192.168.200.0/24)**: OVN hijacked routing — `ip route get 192.168.200.x` went through `br-ex` instead of `enp163s0`

**2. Host networking with `addressRanges` targeting 200G subnet**

Set `network.provider: host` + `public_network = 10.0.0.0/16` so Ceph daemons bind to 200G NIC IPs.

**Result: FAILED.** OVN-networked pods have NO route to 10.0.0.0/16. `ip route get 10.0.0.7` from inside any OVN pod returns "no route to host." New MGR/MDS pods advertised 200G IPs and became unreachable. PGs went "unknown."

Additionally discovered that Rook operator does NOT update existing MON/OSD deployments with `hostNetwork: true` — it only applies to newly created deployments. Had to manually patch all 21 OSD deployments with `oc patch deployment --type strategic -p '{"spec":{"template":{"spec":{"hostNetwork":true,"dnsPolicy":"ClusterFirstWithHostNet"}}}}'`.

**3. Host networking with management subnet**

Set `public_network = 10.243.65.0/24` so Ceph binds to the management IP (reachable from OVN pods).

**Result: FAILED.** OCP's OVN firewall blocks pod-to-nodeIP traffic on non-standard ports. Confirmed with `nc -zv`:
- Port 22 (SSH): reachable cross-node ✓
- Port 6789 (MON): blocked ✗
- Port 6800 (OSD): blocked ✗
- Port 3300 (MON v2): blocked ✗

IBM Cloud VPC security groups add a second layer of blocking on the same ports cross-node.

### Cluster damage and recovery

The host networking attempt broke MON quorum — new MONs (f, g) on `hostNetwork` couldn't be reached by old MON (e) on pod network. Recovery required:

1. Scale down operator and all MONs
2. Privileged repair pod with `quay.io/ceph/ceph:v19`, mounting MON data directory from host
3. Extract monmap: `ceph-mon --extract-monmap /tmp/monmap --mon-data /var/lib/rook/mon-e/data`
4. Remove broken MONs: `monmaptool --rm f`, `--rm g`, `--rm c`
5. Inject fixed map: `ceph-mon --inject-monmap /tmp/monmap --mon-data /var/lib/rook/mon-e/data`
6. Update `rook-ceph-mon-endpoints` configmap to single surviving MON
7. Delete broken MON deployments, scale up surviving MON
8. Remove stale `public_network` from Ceph config database (OSDs crashed with "unable to find any IPv4 address in networks '10.243.65.0/24'")
9. Restart all OSD pods, scale up operator — operator auto-created 2 new MONs

Cluster recovered to HEALTH_OK with all 109.92k objects (426 GiB) intact.

### Conclusion

**200G networking is NOT viable for Rook-Ceph on OCP with OVN-Kubernetes on IBM Cloud.** Three independent blockers:

1. **ipvlan kernel limitation**: Host cannot communicate with its own ipvlan children — breaks the CephFS kernel client → OSD path
2. **OVN firewall**: Blocks Ceph ports (6789, 6800+) on node IPs from pods — would require OCP network-policy changes or custom OVN ACLs
3. **IBM Cloud VPC security groups**: Block cross-node host-to-host traffic on non-standard ports — would require VPC security group modifications

The cluster runs Ceph over the OVN overlay (~10 Gbps management NIC). For higher throughput, the remaining options are:

- **Same-node CRUSH rule**: Pin data to local OSDs, eliminating all cross-node data traffic
- **hostPath NVMe tier**: `nvme0n1` is already reserved on each node for this

### Same-node CRUSH rule (not yet applied)

Since benchmarks are pinned to specific nodes and the data pool uses `size: 1`:

```bash
ceph osd crush rule create-replicated kvcache-local default host nvme
ceph osd pool set kvcache-fs-data0 crush_rule kvcache-local
```

This turns every CephFS data write into a local NVMe write with zero network overhead. The tradeoff: RWX reads from other nodes still incur a network hop through OVN, and only 1/3 of total NVMe capacity is available per node. For KV cache (a few TB at most), capacity is not a concern.

## Further opportunities not yet applied

- **Same-node CRUSH rule**: eliminates cross-node data traffic entirely (see above)
- ~~**Multus 200G networking**~~: NOT VIABLE — see investigation above
- ~~**Host networking**~~: NOT VIABLE on OCP/OVN — see investigation above
- ~~**RDMA messenger**~~: Requires 200G networking to be viable first
- **GPUDirect Storage (GDS)**: CephFS does not support `O_DIRECT`, so full GDS bypass is unavailable; `bb_read_write` bounce-buffer mode is unverified
- **Erasure coding**: lower storage overhead than replication with less write amplification, but adds CPU overhead

## Related

- [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]]
- [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]
- [[Ceph orphaned OSDs after node disruption]]
- Research: [[KV Cache Offloading]]