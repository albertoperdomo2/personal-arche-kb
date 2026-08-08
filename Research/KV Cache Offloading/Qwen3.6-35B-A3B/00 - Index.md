---
title: "Qwen3.6-35B-A3B KV-cache offloading"
date: "2026-07-29"
type: "research-model-index"
model: "Qwen3.6-35B-A3B"
topic: "KV Cache Offloading"
status: "active"
---

# Qwen3.6-35B-A3B

## Reports

- [[2026-08-08 - CPU offload two-replica external-tier lookup investigation|2026-08-08 — CPU offload two-replica external-tier lookup investigation]]
- [[2026-07-29 - Qwen3.6-35B-A3B - U0.55 KV-cache offload matrix|2026-07-29 — U0.55 four-tier offload matrix with full CephFS pool telemetry]]
- [[2026-07-28 - Clean U0.85 offload matrix|2026-07-28 — Clean U0.85 offload matrix with NVMe and CephFS telemetry]]
- [[2026-07-25 - Standardized offload matrix batch 3|2026-07-25 — Standardized offload matrix batch 3]]
- [[2026-07-25 - Standardized offload matrix pressure run|2026-07-25 — Standardized offload matrix pressure run]]
- [[2026-07-23 - TieringOffloadingSpec matrix|2026-07-23 — Clean TieringOffloadingSpec matrix]]
- [[2026-07-23 - TieringOffloadingSpec matrix Summary|2026-07-23 — Matrix summary]]
- [[2026-07-24 - Six-run CephFS tuning matrix|2026-07-24 — Six-run comparison with tuned CephFS]]

## Latest working conclusion

At U=0.55 and AgentX C=32, the 2026-07-29 matrix measured 0.578 req/s without offload, 1.172 req/s with CPU 64 GiB, 1.291 req/s with CPU+NVMe, and 1.284 req/s with CPU+CephFS. NVMe and CephFS completed 44 sessions each and differed by only 0.56% in request throughput, which is near-equality for these single runs rather than general equivalence.

CephFS was directly exercised over the full 35-minute benchmark: `kvcache-fs-data0` has 141 samples at 15-second cadence. Raw full-window averages were 575.5 MiB/s reads and 198.4 MiB/s writes; after the five-minute rate-window carry-in cleared, they were 269.8 and 214.7 MiB/s. The PVC grew from 0 to 374.0 GiB. The filesystem remained Ready; cluster health was `HEALTH_WARN` because of `MON_DISK_LOW` and `RECENT_CRASH`. No store-refusal storm, slow operation, model restart, OOM, or traceback was present.

## Current MLflow run registry

| Configuration | Run |
|---|---|
| No offload | [6b1884118aec42fbbaf9ed9accc1c05e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/6b1884118aec42fbbaf9ed9accc1c05e?workspace=benchflow) |
| CPU-only offload | [326fb9ac213b4a7facaf7a087f31e623](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/326fb9ac213b4a7facaf7a087f31e623?workspace=benchflow) |
| CPU + NVMe | [2b4bbe4d3919402795ade534cea0e5ff](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/2b4bbe4d3919402795ade534cea0e5ff?workspace=benchflow) |
| CPU + CephFS | [20ea91a83f02424fb391bbd4a44d3cc6](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/20ea91a83f02424fb391bbd4a44d3cc6?workspace=benchflow) |

## Next experiment

Repeat all four U0.55 cells at least three times with randomized order, explicit secondary-tier cleaning, matched node placement, full 15-second Ceph pool telemetry, and working Ceph MDS/OSD latency plus NVMe await/queue-depth metrics.