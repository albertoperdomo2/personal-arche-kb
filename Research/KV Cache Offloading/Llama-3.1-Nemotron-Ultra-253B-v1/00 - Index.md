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

Revision 2 confirms the initial report's durable conclusion at a lower pressure point. Halving AgentX concurrency from 32 to 16 roughly halves waiting depth and mean request latency, but 64 GiB CPU offload still does not improve throughput or completed sessions. In Revision 2, CPU offload is 1.04% lower in request throughput, 9.50% worse in mean TTFT, and completes the same 3 of 18 root sessions.

The mechanism remains write-dominated: Revision 2 stores 3,743.3 GiB to CPU and loads only 1.562 GiB, a roughly 2,396:1 ratio. External-KV prompt attribution remains zero. Several small CPU effects reverse sign between revisions, so neither pair supports a stable performance ranking; both support treating the CPU tier as an eviction sink for this workload.

## Next experiment

Validate the restoration path with an exact-prefix smoke test and block-level load-hit/miss/lookup telemetry. Only after that passes, run at least three paired, randomized baseline/CPU repetitions on the same node at C8 and C16.

## Run registry

| Revision | Configuration | Run |
|---|---|---|
| Revision 2 · C16 | No offload | [MLflow `5e6ea113`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/5e6ea11348e5401086cfe71599306b37?workspace=benchflow) |
| Revision 2 · C16 | CPU offload 64 GiB | [MLflow `43521e84`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/43521e84d39b41cc8fe59e52c6b95598?workspace=benchflow) |
| Revision 1 · C32 | No offload | [MLflow `c2720531`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/c272053125704ddea1941213a8e84f5d?workspace=benchflow) |
| Revision 1 · C32 | CPU offload 64 GiB | [MLflow `98586342`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/9858634252fa410da396f69a3eaf6816?workspace=benchflow) |