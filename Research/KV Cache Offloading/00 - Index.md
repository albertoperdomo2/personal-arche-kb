---
title: KV Cache Offloading Research
date: 2026-07-24
type: research-index
topic: KV Cache Offloading
---

# KV Cache Offloading

## Organization

Research is organized by model where the model is known:

- [[Qwen3.6-35B-A3B/00 - Index|Qwen3.6-35B-A3B]]
- [[Nemotron-3-Super-120B/00 - Index|Nemotron 3 Super 120B]]
- [[AgentX Cross-Model/00 - Index|AgentX cross-model methodology and mechanism notes]]

The dated reports in each model directory are the canonical organized copies. Earlier theme-level paths remain as historical compatibility records because Arche currently provides article creation/update but no move/rename operation.

## Current conclusions

- The standardized Nemotron matrix uses 200 GiB /dev/shm, TieringOffloadingSpec, and a 64 GiB CPU tier across all four runs. NVMe is the fastest tier; CephFS shows a store-refusal/backpressure tail.
- The standardized Qwen matrix shows the same broad pattern: CPU offload is useful, NVMe improves on CPU-only, and CephFS requires direct tier telemetry before storage causality can be claimed.
- All durable reports now use YAML frontmatter. Dated notes that are cross-model or historical are linked from the cross-model index.