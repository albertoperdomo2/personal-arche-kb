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
| `02-ceph-cluster.yaml` | OSD resources: 2–4 CPU, 8–12 GiB; `network.provider: host` with `addressRanges.public: ["10.0.0.0/16"]`; pinned image to `v19.2.4` |
| `03-cephfs.yaml` | Data pool `size: 1`; MDS activeCount 2, resources 2–4 CPU / 4–8 GiB; StorageClass mountOptions |
| `RUNBOOK.md` | Added Step 5 (runtime `ceph config set` commands), fixed OSD count 24→21, updated embedded YAML examples, added 200G migration procedure |

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
ceph config set global public_network 10.0.0.0/16         # 200G NIC binding
ceph osd pool set kvcache-fs-data0 size 1 --yes-i-really-mean-it
ceph osd pool set kvcache-fs-data0 min_size 1
ceph health mute POOL_NO_REDUNDANCY
```

## 200G NIC networking for Ceph — deployed (2026-07-24)

### Current state

All Ceph data traffic now routes over the 200 Gbps ConnectX-7 NICs instead of the 10 Gbps management NIC. The migration was completed via a big-bang MON migration approach after three earlier approaches failed.

**Verified performance**: 1.3 GB/s sequential write, 1.7 GB/s sequential read over CephFS. Traffic confirmed on `enp163s0` (200G NIC) with 0 bytes on `enp3s0` (management NIC) during I/O.

### Network topology

- **OVN overlay** (management): Geneve tunnels on `enp3s0` via `br-ex`, 10.243.65.0/24 node IPs, ~10 Gbps
- **200G NICs** (data): 8× ConnectX-7 per node (PCI passthrough VFs from hypervisor), `enp163s0`–`enp233s0`
- **200G IPs**: fx7c8=10.0.0.6, mt46x=10.0.0.7, 6kl5z=10.0.0.4, gjfjh=10.0.0.8 (10.0.0.0/16 subnet, MTU 9000)

### How it works

All Ceph daemons run with `hostNetwork: true`. The CephCluster CR sets `network.provider: host` and `addressRanges.public: ["10.0.0.0/16"]`, which tells Ceph to use `public_network = 10.0.0.0/16`. Daemons bind to the 200G NIC IP matching that subnet.

**Traffic routing:**
- OSD-to-OSD replication, client-to-OSD data, client-to-MDS metadata → 200G (10.0.0.x)
- MON heartbeats and map updates → management (10.243.65.x) — acceptable since MON traffic is tiny
- MGR → 200G (binds to 10.0.0.x based on `public_network`)
- MDS → 200G (same)

### Why MONs use management IPs

The Rook operator determines MON addresses from the Kubernetes Node object's `InternalIP`, which is always the management NIC IP. The 200G IPs are configured on the host but not reflected in `node.status.addresses`. MON-e was manually patched to 10.0.0.6 via monmap repair, but operator-created MONs (k, l) use management IPs. This is acceptable because MON traffic is only heartbeats and OSD map updates.

### Key gotchas from the migration

1. **`--public-addr=$(ROOK_POD_IP)`** in MON and MGR deployment args: With hostNetwork, `ROOK_POD_IP` resolves to the node's management IP, overriding `public_network`. Must be removed from MGR args; must be replaced with explicit 200G IP for MON.

2. **`rook-ceph-config` secret**: Contains `mon_host` used by ALL daemons. Must be updated when MON addresses change. Stale values cause init containers to hang for 5 minutes trying dead addresses.

3. **Operator version upgrade deadlock**: If the Ceph image tag (`v19`) resolves to a newer version than running daemons, the operator tries rolling-upgrade before creating new MONs. With <3 MONs, it can't stop the existing MON. Fix: pin image to exact version (e.g., `v19.2.4`).

4. **`--setuser-match-path`** in MON deployment args: Contains a hardcoded path like `/var/lib/ceph/mon/ceph-k/store.db`. Must be updated when creating MON deployments from templates.

### Earlier failed approaches

**1. Multus ipvlan L2** — Host cannot ARP-resolve its own ipvlan children (kernel limitation). CephFS kernel client → OSD path breaks.

**2. Host networking without big-bang MON migration** — OVN pods have no route to 10.0.0.0/16.

**3. Host networking with management subnet** — OVN firewall blocks Ceph ports on node IPs. IBM Cloud VPC security groups add a second blocking layer.

### Cluster damage and recovery during investigation

The initial host networking attempt broke MON quorum — new MONs on `hostNetwork` couldn't be reached by old MON on pod network. Recovery required monmap injection repair: extract monmap from surviving MON's data directory, remove broken MON entries, inject fixed map, update endpoints configmap, delete broken MON deployments. Full procedure documented in RUNBOOK.md.

## Further opportunities

- **Same-node CRUSH rule**: Could eliminate cross-node data traffic entirely for single-node benchmarks
- **GPUDirect Storage (GDS)**: CephFS does not support `O_DIRECT`, so full GDS bypass is unavailable
- **Erasure coding**: Lower storage overhead with less write amplification, but adds CPU overhead

## Related

- [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]]
- [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]
- [[Ceph orphaned OSDs after node disruption]]
- Research: [[KV Cache Offloading]]