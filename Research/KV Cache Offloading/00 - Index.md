# KV Cache Offloading

## New cleaned Qwen3.6 matrix — 2026-07-23

See [[2026-07-23 - Qwen3.6 AgentX cleaned TieringOffloadingSpec matrix/00 - Report|Qwen3.6 AgentX cleaned TieringOffloadingSpec matrix report]].

The new matrix uses the same 64 GiB CPU tier, TieringOffloadingSpec, 200 GiB /dev/shm, model, TP=2, U=0.55, and AgentX workload across CPU64, NVMe, and CephFS. The supplied no-offload run is an older directional baseline.

- CPU64: 1.191 req/s.
- CPU64 + NVMe: 1.284 req/s (+7.8% vs CPU64); NVMe traffic is active and device busy is low, so it is not saturated.
- CPU64 + CephFS: 0.711 req/s (-40.3% vs CPU64). The retained log contains at least 66,320 cannot-store-blocks warnings, while PVC used remains 0 GiB and direct Ceph read/write telemetry is absent. The regression is therefore a secondary-tier admission/backpressure failure, not proven raw Ceph bandwidth.
- No pod restarts, CUDA OOMs, or server tracebacks explain the split. Workload prompt shapes are matched; Ceph's long tail reduces closed-loop offered load.

## Standardized Nemotron matrix — 2026-07-23

See [[2026-07-23 - Nemotron 3 Super 120B standardized KV-offload matrix/00 - Report|Nemotron standardized KV-offload matrix report]].

This batch standardizes the Nemotron TP=4/U=0.64 AgentX deployment with matched 64 GiB CPU capacity and 200 GiB shared memory across the offload cells.

- No offload: 1.283 req/s.
- CPU64: 1.293 req/s.
- CPU64 + NVMe: 1.445 req/s, approximately 11.8% above CPU64.
- CPU64 + CephFS: 1.200 req/s, approximately 7.2% below CPU64.
- The report includes native 15-second time series for prompt-token sources, KV occupancy, scheduler backlog, CPU/memory, KV transfers, NVMe, CephFS/PVC telemetry, and warnings, with deployment and workload audits.

The next gate is to repeat the matrix with direct per-tier hit/miss, byte, queue, latency, and complete CephFS telemetry before making storage-causality claims.