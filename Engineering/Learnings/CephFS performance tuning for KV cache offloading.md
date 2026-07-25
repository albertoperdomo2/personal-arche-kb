---
title: CephFS performance tuning for KV cache offloading
date: 2026-07-24
type: learning
---

# CephFS Performance Tuning for KV Cache Offloading

When using CephFS as a secondary KV-cache offloading tier for vLLM TieringOffloadingSpec, default Rook-Ceph configurations cause severe regressions: 7-40% throughput loss vs CPU-only, 66K-127K cannot store blocks warnings per run, and 51% token recomputation vs 11% with local NVMe.

## Validated performance (2026-07-24)

After tuning, CephFS achieves parity with local NVMe for TTFD P95 latency:

- Bandwidth: ~13 Gbps (1.3 GB/s) sequential, 4x improvement over pre-tuning ~300 Mbps.
- TTFD P95 latency: on par with CPU+NVMe configuration.
- Config: Single replica, TP2, 3-node diadochos cluster, replication removed for initial testing.

These results confirm CephFS as a viable distributed KV cache tier when properly tuned.

## Root causes identified (diadochos cluster)

Six reinforcing problems in default Rook deployment:

### 1. 3x replication on ephemeral data (dominant factor)
Every KV block write replicated to 3 nodes. Fix: replicated.size: 1 on data pool.

### 2. OSD memory starvation
OSDs limited to 8 GiB, osd_memory_target 4 GiB. Fix: raise to 12 GiB pods, 8 GiB target.

### 3. Under-resourced single MDS
1 active MDS, 500m-2 CPU. Fix: 2 active MDS, 2-4 CPU, 4-8 GiB, mds_cache_memory_limit 4 GiB.

### 4. No CephFS client mount options
Default atime updates on every read. Fix: noatime, nodiratime, wsize/rsize 16M, readahead 32 MiB.

### 5. vLLM FS tier thread defaults
Too few I/O threads for high-latency backend. Fix: n_read_threads 64, n_write_threads 32.

### 6. Squid v19 bdev_async_discard_threads bug
Fix: ceph config set osd bdev_async_discard_threads 1.

## Cascade mechanism

CephFS write latency -> CPU primary fills faster -> vLLM refuses block stores -> token recomputation -> GPU compute contention -> more evictions. One request retried 4082 times in a single run.

## Changes applied (2026-07-24)

Applied to diadochos cluster manifests in clusters/psap-diadochos-h100/rook-ceph/.

Runtime commands:
ceph config set osd osd_memory_target 8589934592
ceph config set osd bdev_async_discard_threads 1
ceph config set osd ms_async_op_threads 5
ceph config set mds mds_cache_memory_limit 4294967296
ceph config set client client_readahead_max_bytes 33554432
ceph config set client client_cache_size 65536
ceph config set global public_network 10.0.0.0/16
ceph osd pool set kvcache-fs-data0 size 1 --yes-i-really-mean-it
ceph osd pool set kvcache-fs-data0 min_size 1
ceph health mute POOL_NO_REDUNDANCY

## 200G NIC networking for Ceph

All Ceph data traffic routes over 200 Gbps ConnectX-7 NICs via hostNetwork and public_network 10.0.0.0/16. Verified: 1.3 GB/s write, 1.7 GB/s read. Big-bang MON migration required after three failed approaches (Multus ipvlan, host net without migration, host net with mgmt subnet).

Key gotchas: ROOK_POD_IP overrides public_network in MON/MGR args; rook-ceph-config secret must update with MON addresses; operator version deadlock if Ceph image tag is unpinned; setuser-match-path has hardcoded MON store paths.

## Further opportunities

- Same-node CRUSH rule for single-node benchmarks
- Erasure coding for lower storage overhead
- GPUDirect Storage not available (CephFS lacks O_DIRECT)

## Related

- vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec
- Ceph orphaned OSDs after node disruption
