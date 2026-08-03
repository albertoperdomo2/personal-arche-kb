---
title: "KV Cache Offloading Research"
date: "2026-08-03"
type: "research-index"
topic: "KV Cache Offloading"
status: "active"
models: 6
---

# KV Cache Offloading

## Organization

Research is organized by model where the model is known:

- [[Qwen3.6-35B-A3B/00 - Index|Qwen3.6-35B-A3B]]
- [[Gemma-4-31B-it/00 - Index|Gemma 4 31B IT]]
- [[Nemotron-3-Super-120B/00 - Index|Nemotron 3 Super 120B]]
- [[Llama-3.1-Nemotron-Ultra-253B-v1/00 - Index|Llama 3.1 Nemotron Ultra 253B v1]]
- [[Llama-3.1-Nemotron-Ultra-253B-v1-FP8/00 - Index|Llama 3.1 Nemotron Ultra 253B v1 FP8]]
- [[AgentX Cross-Model/00 - Index|AgentX cross-model methodology and mechanism notes]]

The dated reports in each model directory are the canonical organized copies. Earlier theme-level paths remain as historical compatibility records because Arche currently provides article creation/update but no move/rename operation.

## Methodology

- [[Experiment Methodology]] — run structure and acceptance rules.
- [[01 - Calibration Protocol]] — finding the pressure point per model.
- [[02 - Per-Model Methodology Template]] — per-model run template.
- [[03 - KV Transfer Metrics and PromQL]] — verified metric inventory, PromQL for retrieval/stall latency, interpretation identities, measurement traps, and the minimum reporting set. Names verified against `vllm@4ee9702`.

## Workload

All standardized runs use the same workload: [[AgentX Workload Definition|AgentX MVP]] (AIPerf `aiperf-agentx-inference`, trace `semianalysisai/cc-traces-weka-with-subagents-060826`). That note characterizes the generation mechanism — trace corpus, session/turn/subagent topology, arrival structure, cache-bust, warmup/profiling phases — distinct from the per-model run outcomes recorded below.

## Current conclusions

- The 2026-08-03 Gemma TP2/U0.92/C8 matrix establishes a strong offload benefit with CPU256. CephFS reaches 0.1844 req/s and 145.4 output tok/s, 3.10× and 4.41× the no-offload baseline, while CPU-only reaches 0.1202 req/s. The NVMe cell is conditionally accepted because 15 store refusals and deferred requests create an extreme long tail despite fast median latency and healthy raw device bandwidth.
- The 2026-07-31 FP8 Nemotron Revision 2 matrix at TP8/U0.80/C32 successfully forces offload. CPU256 reaches 0.437 req/s (3.31× the 0.132 req/s baseline); NVMe reaches 0.759 req/s and CephFS 0.765 req/s (5.74–5.78× baseline). External tiers supply 60.9–65.7% of prompt tokens instead of the baseline recomputing 97.2%. NVMe and CephFS are at parity; the next clean center is TP8/U0.82/C32. The CephFS cell remains conditional because two requests disconnected and Ceph was HEALTH_WARN.
- The first Llama 3.1 Nemotron Ultra 253B AgentX comparison found no demonstrated benefit from a 64 GiB CPU tier. The tier accepted about 4.85 TB of GPU-to-CPU writes but returned only 3.78 GB, leaving prompt processing 98.94% recompute-dominated and session completion unchanged at 2 of 32.
- The 2026-07-30 FP8 Nemotron matrix at U0.90/C16 was underfilled and showed parity. Its 2.31-million-token HBM shelf peaked at only ~50%, motivating the pressure change tested successfully in Revision 2.
- The standardized Nemotron matrix uses 200 GiB `/dev/shm`, `TieringOffloadingSpec`, and a 64 GiB CPU tier across all four runs. NVMe is the fastest tier; CephFS shows a store-refusal/backpressure tail.
- The 2026-07-29 Qwen U0.55 matrix measured 0.578 req/s without offload, 1.172 req/s with CPU-only, 1.291 req/s with CPU+NVMe, and 1.284 req/s with CPU+CephFS. NVMe and CephFS were within 0.56% in these single runs. The CephFS report includes all 141 `kvcache-fs-data0` samples over the complete 35-minute window, including the explicitly identified initial five-minute Prometheus rate-window carry-in.
- All durable reports use YAML frontmatter. Dated notes that are cross-model or historical are linked from the cross-model index.