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

Default `atime` updates generated unnecessary write traffic on every read. Default `rsize`/`wsize` buffers (16 MiB kernel default, but potentially lower in some CSI configurations) were untuned. No readahead optimization.

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

All changes applied to the diadochos cluster manifests in `clusters/psap-diadochos-h100/rook-ceph/`:

| File | Change |
|------|--------|
| `02-ceph-cluster.yaml` | OSD resources: 2–4 CPU, 8–12 GiB |
| `03-cephfs.yaml` | Data pool `size: 1`; MDS activeCount 2, resources 2–4 CPU / 4–8 GiB; StorageClass mountOptions |
| `RUNBOOK.md` | Added Step 5 (runtime `ceph config set` commands), fixed OSD count 24→21, updated embedded YAML examples |

BenchFlow deployment profile change:

| File | Change |
|------|--------|
| `profiles/deployment/rhoai/multi-tier-offloading-cephfs.yaml` | `n_read_threads: 64`, `n_write_threads: 32` |

## Runtime commands (apply once via tools pod)

```bash
ceph config set osd osd_memory_target 8589934592
ceph config set osd bdev_async_discard_threads 1
ceph config set mds mds_cache_memory_limit 4294967296
ceph config set client client_readahead_max_bytes 33554432
ceph config set client client_cache_size 65536
```

If data pool was previously `size: 3`:

```bash
ceph osd pool set kvcache-fs-data0 size 1 --yes-i-really-mean-it
ceph osd pool set kvcache-fs-data0 min_size 1
```

## Further opportunities not yet applied

- **Same-node CRUSH rule**: Pin `kvcache-fs-data0` to OSDs on the node running the vLLM pod. Combined with `size: 1`, every CephFS write becomes a local NVMe write with zero network overhead. Tradeoff: RWX reads from other nodes add a network hop.
- **GPUDirect Storage (GDS)**: CephFS does not support `O_DIRECT`, so full GDS bypass is not available. The `bb_read_write` bounce-buffer mode may still help but is unverified.
- **Erasure coding**: Lower storage overhead than replication with less write amplification, but adds CPU overhead and complicates CephFS data pool configuration.

## Related

- [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]]
- [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]
- [[Ceph orphaned OSDs after node disruption]]
- Research: [[KV Cache Offloading]]