---
title: "KV Cache Offloading Research"
date: "2026-07-29"
type: "research-index"
topic: KV Cache Offloading
status: "active"
models: 4
---

# KV Cache Offloading

## Organization

Research is organized by model where the model is known:

- [[Qwen3.6-35B-A3B/00 - Index|Qwen3.6-35B-A3B]]
- [[Nemotron-3-Super-120B/00 - Index|Nemotron 3 Super 120B]]
- [[Llama-3.1-Nemotron-Ultra-253B-v1/00 - Index|Llama 3.1 Nemotron Ultra 253B v1]]
- [[AgentX Cross-Model/00 - Index|AgentX cross-model methodology and mechanism notes]]

The dated reports in each model directory are the canonical organized copies. Earlier theme-level paths remain as historical compatibility records because Arche currently provides article creation/update but no move/rename operation.

## Workload

All standardized runs use the same workload: [[AgentX Workload Definition|AgentX MVP]] (AIPerf `aiperf-agentx-inference`, trace `semianalysisai/cc-traces-weka-with-subagents-060826`). That note characterizes the generation mechanism — trace corpus, session/turn/subagent topology, arrival structure, cache-bust, warmup/profiling phases — distinct from the per-model run outcomes recorded below.

## Current conclusions

- The first Llama 3.1 Nemotron Ultra 253B AgentX comparison found no demonstrated benefit from a 64 GiB CPU tier. The tier accepted about 4.85 TB of GPU-to-CPU writes but returned only 3.78 GB, leaving prompt processing 98.94% recompute-dominated and session completion unchanged at 2 of 32.
- The standardized Nemotron matrix uses 200 GiB `/dev/shm`, `TieringOffloadingSpec`, and a 64 GiB CPU tier across all four runs. NVMe is the fastest tier; CephFS shows a store-refusal/backpressure tail.
- The 2026-07-29 Qwen U0.55 matrix measured 0.578 req/s without offload, 1.172 req/s with CPU-only, 1.291 req/s with CPU+NVMe, and 1.284 req/s with CPU+CephFS. NVMe and CephFS were within 0.56% in these single runs. The CephFS report includes all 141 `kvcache-fs-data0` samples over the complete 35-minute window, including the explicitly identified initial five-minute Prometheus rate-window carry-in.
- All durable reports use YAML frontmatter. Dated notes that are cross-model or historical are linked from the cross-model index.