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

## Networking investigation (2026-07-24)

### Current state

All Ceph traffic (OSD, MDS, client) runs over the OVN pod overlay at MTU 1400 via the management NIC. Each node has 8× ConnectX-7 200 Gbps RDMA NICs (mlx5_0–mlx5_7, `enp163s0`–`enp233s0`) that are:

- Linked at **200 Gb/sec (4X HDR)**
- All on the shared `10.0.0.0/16` subnet with cross-node reachability verified (sub-ms latency)
- **Zero RDMA traffic** — RDMA port counters show 0 bytes transmitted/received
- NICs support up to **MTU 9978** (currently set to 1500)

Multus CNI is deployed on the cluster. The CephCluster has no custom network configuration.

### Opportunity: Multus-based 200G Ceph network

Routing Ceph traffic over the 200G NICs is the single biggest remaining optimization — a potential ~160× bandwidth increase for cross-node Ceph operations. The path:

1. Raise MTU to 9000 on `enp163s0` (all 3 nodes)
2. Create a macvlan `NetworkAttachmentDefinition` in `rook-ceph` on `enp163s0`
3. Update CephCluster with `spec.network.provider: multus` and `spec.network.selectors.public`
4. Attach the same Multus network to vLLM pods (CephFS kernel client needs to reach OSDs on the 200G addresses)

**Risks:** Step 3 restarts all Ceph daemons. Both Ceph pods and CephFS client pods need the secondary interface. If the macvlan network has issues, Ceph could lose quorum.

**Mitigation:** With `size: 1`, a per-host CRUSH rule can pin data to the local node, eliminating cross-node data traffic entirely. Multus is then only needed for MDS metadata (small volume).

### Alternative: same-node CRUSH rule

Since benchmarks are pinned to specific nodes and the data pool uses `size: 1`:

```bash
# Example for pinning data to fx7c8
ceph osd crush rule create-replicated kvcache-local default host nvme
ceph osd pool set kvcache-fs-data0 crush_rule kvcache-local
```

This turns every CephFS data write into a local NVMe write with zero network overhead. The tradeoff is that RWX reads from other nodes would still work but with a network hop, and only 1/3 of total NVMe capacity is available. For KV cache (a few TB at most), capacity is not a concern.

## Further opportunities not yet applied

- **Same-node CRUSH rule**: eliminates cross-node data traffic entirely (see above)
- **Multus 200G networking**: route all Ceph traffic over 200 Gbps RDMA NICs
- **Jumbo frames**: 200G NICs support MTU 9978; raising from 1500 to 9000 reduces per-packet overhead
- **RDMA messenger**: Ceph supports `ms_type = async+rdma` for zero-copy RDMA messaging over the ConnectX-7 NICs
- **GPUDirect Storage (GDS)**: CephFS does not support `O_DIRECT`, so full GDS bypass is unavailable; `bb_read_write` bounce-buffer mode is unverified
- **Erasure coding**: lower storage overhead than replication with less write amplification, but adds CPU overhead

## Related

- [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]]
- [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]
- [[Ceph orphaned OSDs after node disruption]]
- Research: [[KV Cache Offloading]]