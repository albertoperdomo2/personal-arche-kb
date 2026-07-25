---
title: Cross-model KV Cache Offloading research synthesis
date: 2026-07-25
type: research-synthesis
topic: KV Cache Offloading
---

# Cross-model KV Cache Offloading — Research Synthesis

## Scope

This document synthesizes KV-cache offloading findings across all models tested under the standardized 4-cell matrix (HBM-only, CPU 64 GiB, CPU 64 GiB + NVMe, CPU 64 GiB + CephFS). It is the single-entry-point for the current state of the research.

## Models tested

| Model | TP | GPU | U | Key reports |
|---|---|---|---|---|
| Qwen3.6-35B-A3B | 2 | 2× H100 | 0.55–0.85 | 5 reports (Jul 23–25) |
| Nemotron-3-Super-120B | 4 | 4× H100 | 0.64 | 3 reports (Jul 22–24) |
| AgentX Cross-Model | 2 | 2× H100 | 0.55 | 8 reports + synthesis (Jul 17–21) |
| Gemma-4-31B-it | 4 | 4× H100 | 0.85 | 1 report (Jul 25) |
| DeepSeek-V4-Flash | — | — | — | 1 report (Jul 24) |

## Cross-cutting conclusions

### 1. CPU offload is strongly beneficial

Across Qwen and Nemotron (the two models with clean standardized matrices), adding a 64 GiB CPU tier produces:

- **Qwen**: +106% request throughput, ~50% latency reduction (0.578 → 1.191 req/s)
- **Nemotron**: +0.7% request throughput, but NVMe then adds +12.6% over CPU64 (1.293 → 1.445 req/s)

The AgentX cross-model work showed +103% throughput improvement from CPU offload. CPU offload consistently enables external KV transfer, reducing recomputation from ~97% to ~22% (Qwen) or ~22% (Nemotron).

### 2. NVMe is the strongest secondary tier

- **Qwen**: NVMe adds +7.8% over CPU64 (1.191 → 1.284 req/s), reduces recomputation from 22% to 11%
- **Nemotron**: NVMe adds +11.8% over CPU64 (1.293 → 1.445 req/s), reduces recomputation from 22% to 12.5%
- NVMe retains more KV blocks in the tier hierarchy, reducing demotion churn (store/load ratio 0.18:1 vs 0.38:1 for CPU64)
- NVMe device headroom is large: mean busy 15.8% (Qwen), 21.9% (Nemotron) — far from saturated
- The gain is behavioral (better hierarchy retention), not a faster copy primitive (~20 ms/GiB CPU→GPU across all tiers)

### 3. CephFS has a systemic backpressure issue

CephFS consistently underperforms across all models:

- **Qwen**: 0.711 req/s (vs 1.191 for CPU64), 66,320+ `cannot store blocks` warnings
- **Nemotron**: 1.200 req/s (vs 1.293 for CPU64), 126,998+ store-refusal warnings
- **AgentX**: Two independent CephFS observations both failed with store-refusal floods and tail latency collapse
- **Gemma**: 273 cannot-store-block events, 0 completed sessions

The mechanism is secondary-tier drain/backpressure: slow CephFS writes pin CPU blocks, preventing eviction, which floods the store path with refusals, which forces recomputation, which collapses external KV reuse.

### 4. Gemma-4-31B-it is a complete failure case

Unlike Qwen and Nemotron, Gemma-4-31B-it shows no offload benefit at all:
- No offload and CPU64 both completed only 2–3 sessions
- NVMe and CephFS completed zero sessions
- KV occupancy reached 84–87% but the tiered backends failed to store blocks
- This suggests model-specific factors (architecture, token generation pattern, or memory layout) interact with the offload path

### 5. Current workloads don't fully stress offload

- Nemotron: KV occupancy only ~10% in early runs, ~15–45% in standardized runs
- Qwen: KV occupancy 80–94% in pressure runs, 13–58% in standardized runs
- AgentX: KV occupancy 92–95% at C=32, U=0.55
- Need higher concurrency to amplify tier differences for models with low occupancy

## Standardized experiment configuration

All standardized matrix runs share:

| Parameter | Value |
|---|---|
| Workload | AgentX MVP (WEKA `semianalysisai/cc-traces-weka-with-subagents-060826`) |
| Seed | 42 |
| Concurrency | 32 |
| Duration | 1,800 seconds |
| Context length | 131,072 tokens |
| `/dev/shm` | 200 GiB (all cells) |
| CPU tier | 64 GiB (`TieringOffloadingSpec`) |
| Secondary tiers | none / hostPath NVMe / CephFS PVC |
| Prefix caching | enabled |
| Streaming | enabled |

## Acceptance gates for secondary tiers

A secondary tier (NVMe or CephFS) is accepted when ALL of:

1. Zero `cannot store blocks` warnings
2. No sustained filesystem store queue
3. Stable external-token share (not decaying over time)
4. No p99 tail explosion
5. Direct secondary read/write counters present and non-zero
6. Repeatable session-throughput and latency improvement over capacity-matched CPU-only control
7. At least three paired repetitions per cell

## Telemetry gaps (still open)

- GPU SM utilization, VRAM, PCIe RX/TX: zero series for all runs
- Ceph pool/MDS/OSD metrics: zero series (RBAC or export issue)
- Per-tier hit/miss/latency/queue-depth counters: absent in vLLM
- `prefix_cache_hit_rate`: archived as zero despite non-zero prompt-token-source shares

## Open questions

1. Why does Gemma fail completely where Qwen and Nemotron succeed?
2. Can CephFS be made reliable by tuning `n_write_threads` and drain capacity?
3. What is the NVMe effect size under exact controls (same `/dev/shm`, hash seed, clean state)?
4. How does the offload staircase change with concurrency 32/48/64?
5. Is the Activity-Based predictive preloading approach viable for CephFS?

## Related

- [[00 - Index|KV Cache Offloading index]]
- [[AgentX Cross-Model/00 - Report|AgentX C32 U0.55 tier matrix]]
- Activity-Based KV Cache Offloading concept
