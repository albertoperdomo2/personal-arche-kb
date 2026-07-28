---
title: Qwen3.6-35B-A3B KV-cache offloading
date: 2026-07-28
type: research-model-index
model: Qwen3.6-35B-A3B
topic: KV Cache Offloading
---

# Qwen3.6-35B-A3B

## Reports

- [[2026-07-28 - Clean U0.85 offload matrix|2026-07-28 — Clean U0.85 offload matrix with NVMe and CephFS telemetry]]
- [[2026-07-25 - Standardized offload matrix batch 3|2026-07-25 — Standardized offload matrix batch 3]]
- [[2026-07-25 - Standardized offload matrix pressure run|2026-07-25 — Standardized offload matrix pressure run]]
- [[2026-07-23 - TieringOffloadingSpec matrix|2026-07-23 — Clean TieringOffloadingSpec matrix]]
- [[2026-07-23 - TieringOffloadingSpec matrix Summary|2026-07-23 — Matrix summary]]
- [[2026-07-24 - Six-run CephFS tuning matrix|2026-07-24 — Six-run comparison with tuned CephFS]]

## Latest working conclusion

At U=0.85 and AgentX C=32, the controlled 2026-07-28 matrix measured 1.085 req/s without offload, 1.426 req/s with CPU 64 GiB, 1.518 req/s with CPU+NVMe, and 1.518 req/s with CPU+CephFS. NVMe and CephFS both completed 52 sessions and had nearly identical prompt-source, KV-transfer, and storage-byte behavior. This establishes equality for the observed single runs, not general equivalence: repetitions, explicit NVMe tier cleaning, and storage-latency telemetry are still required.

CephFS was directly exercised: the PVC grew to 372.6 GiB and the data pool averaged 291.9 MiB/s reads plus 227.5 MiB/s writes. The filesystem remained Ready; cluster health was HEALTH_WARN because of MON_DISK_LOW. No store-refusal storm, slow-op evidence, pod restart, OOM, or traceback was present.

## Current MLflow run registry

| Configuration | Run |
|---|---|
| No offload | [b69da750bfbd43afaf8924c74134146e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/b69da750bfbd43afaf8924c74134146e?workspace=benchflow) |
| CPU 64 GiB | [8e16a5150dab436ca843438e09ea9248](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/8e16a5150dab436ca843438e09ea9248?workspace=benchflow) |
| CPU 64 GiB + NVMe | [80d659b915114e9d9cf8e0d88c45cf89](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/80d659b915114e9d9cf8e0d88c45cf89?workspace=benchflow) |
| CPU 64 GiB + CephFS | [decf13e8407d46c6bd342714b5864feb](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/decf13e8407d46c6bd342714b5864feb?workspace=benchflow) |

## Next experiment

Repeat all four cells at least three times with randomized order and explicit secondary-tier cleaning. Retain 15-second telemetry and add NVMe await/queue depth plus Ceph client, MDS, and OSD latency before the C=64 pressure step.