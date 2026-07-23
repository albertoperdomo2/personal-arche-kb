# KV Cache Offloading

## New cleaned Qwen3.6 matrix — 2026-07-23

See [[2026-07-23 - Qwen3.6 AgentX cleaned TieringOffloadingSpec matrix/00 - Report|Qwen3.6 AgentX cleaned TieringOffloadingSpec matrix report]].

The new matrix uses the same 64 GiB CPU tier, TieringOffloadingSpec, 200 GiB /dev/shm, model, TP=2, U=0.55, and AgentX workload across CPU64, NVMe, and CephFS. The supplied no-offload run is an older directional baseline.

- CPU64: 1.191 req/s.
- CPU64 + NVMe: 1.284 req/s (+7.8% vs CPU64); NVMe traffic is active and device busy is low, so it is not saturated.
- CPU64 + CephFS: 0.711 req/s (-40.3% vs CPU64). The retained log contains at least 66,320 cannot-store-blocks warnings, while PVC used remains 0 GiB and direct Ceph read/write telemetry is absent. The regression is therefore a secondary-tier admission/backpressure failure, not proven raw Ceph bandwidth.
- No pod restarts, CUDA OOMs, or server tracebacks explain the split. Workload prompt shapes are matched; Ceph's long tail reduces closed-loop offered load.

The next gate is to repeat CephFS with direct tier hit/miss, bytes, queue, and latency telemetry plus complete model logs before making a storage-performance claim.