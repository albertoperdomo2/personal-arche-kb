---
title: "Llama-3.1-Nemotron-Ultra-253B-v1 KV Cache Offloading"
date: "2026-07-29"
type: "research-index"
status: "active"
model: "RedHatAI/Llama-3_1-Nemotron-Ultra-253B-v1"
reports: 2
---

# Llama-3.1-Nemotron-Ultra-253B-v1

## Reports

- [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2|Revision 2 — baseline vs CPU offload at concurrency 16]]
- [[2026-07-29 - Baseline vs CPU offload experiment|Revision 1 / initial baseline vs CPU offload at concurrency 32]]

## Current conclusion

Revision 2 now includes No offload, CPU64, and CPU160 at AgentX concurrency 16. Neither CPU tier improves throughput or completed sessions: all remain near 0.066 requests/s and complete 3 of 18 root sessions.

CPU160 materially changes the mechanism. CPU→GPU loads rise from 1.562 GiB at CPU64 to 77.586 GiB at CPU160, external-KV prompt share rises from 0% to 2.06%, and the store/load ratio improves from 2,396:1 to 47.8:1. This proves restoration works and that CPU64 was too small. CPU160 is still insufficient: 96.76% of prompt processing remains local compute, its estimated retention is about 107 seconds, and mean TTFT alone is about 241 seconds.

## Next experiment

Keep `gpu_memory_utilization=0.9` and test CPU512 at C16, with `/dev/shm` increased to at least 600 GiB and node-memory headroom verified. Pair baseline, CPU160, and CPU512 with at least three randomized repetitions. Do not lower GPU utilization; calibrate 0.92 and 0.94 only after the CPU-capacity step.

## Run registry

| Revision | Configuration | Run |
|---|---|---|
| Revision 2 · C16 | No offload | [MLflow `5e6ea113`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/5e6ea11348e5401086cfe71599306b37?workspace=benchflow) |
| Revision 2 · C16 | CPU offload 64 GiB | [MLflow `43521e84`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/43521e84d39b41cc8fe59e52c6b95598?workspace=benchflow) |
| Revision 2 · C16 | CPU offload 160 GiB | [MLflow `45f523ae`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/45f523ae6ebf45cf984282befaac0eeb?workspace=benchflow) |
| Revision 1 · C32 | No offload | [MLflow `c2720531`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/c272053125704ddea1941213a8e84f5d?workspace=benchflow) |
| Revision 1 · C32 | CPU offload 64 GiB | [MLflow `98586342`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/9858634252fa410da396f69a3eaf6816?workspace=benchflow) |