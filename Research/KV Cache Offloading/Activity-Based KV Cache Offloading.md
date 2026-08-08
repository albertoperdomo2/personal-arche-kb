---
title: Activity-Based KV Cache Offloading
date: 2026-07-24
type: research-concept
topic: KV Cache Offloading
---

# Activity-Based KV Cache Offloading

A predictive KV cache management approach using time series forecasting to pre-load hot blocks from disk to CPU/GPU memory before they are requested.

## Core hypothesis

Approx 20% of KV cache data is accessed 80% of the time (Pareto rule). If a model predicts which 20% will be hot, data is pre-loaded from disk to CPU RAM, eliminating disk read latency for the working set.

## Architecture

Disk (CephFS, compressed) -> CPU RAM (decompressed, predicted hot) -> GPU Memory

Dual optimization:
1. Space: Compress all KV blocks on disk (3-5x savings).
2. Latency: Decompress only predicted hot 20% to CPU tier; microsecond reads.

## Prediction model

Input (multivariate time series): KV block eviction events, timing, frequency, block sizes, reuse patterns. Output: hot/cold classification with configurable forecast window (5s to 1h).

Model progression:
1. Phase 1: XGBoost (leverages existing llm-d predicted_latency_model.py).
2. Phase 2: LSTM, CNN-time-series, or Transformer if XGBoost insufficient.

## Implementation placement (2026-08-02)

Verdict from inspecting `vllm-project/vllm` and `llm-d/llm-d-router` at `main`: **the ABC engine belongs in core vLLM** — specifically the `vllm/v1/kv_offload` layer (pluggable `CachePolicy` + `TieringOffloadingManager` + offloading scheduler). **llm-d-router contributes only the thin consumer/coordination side**: an EPP scorer consuming exported temperature info, and the session-migration trigger that asks vLLM to execute prefetch.

Rationale: vLLM is the only component that owns physical KV blocks across the four tiers (GPU block pool, CPU primary tier, fs secondary tier = NVMe/CephFS, p2p/obj distributed tiers); the router is a routing control plane that cannot move KV data.

| ABC component | Where | Integration point |
|---|---|---|
| Temperature prediction (XGBoost, temporal/spatial/session features) | core vLLM (`vllm/v1/kv_offload`) | Access telemetry lives in the engine — `KVCacheMetricsCollector` already records per-block access, idle time, reuse gaps (sampled 0.01). llm-d-router's `latencypredictorclient` (predictedlatency dataproducer) is the XGBoost *pattern* to reuse, but for placement the output must reach vLLM's offload managers. |
| Cost-aware tier placement (Hot→GPU / Warm→CPU / Cool→NVMe / Cold→CephFS, Benefit > N×Cost, priority scheduler, async multi-hop prefetch) | core vLLM | Replaces today's cascade-all-on-store + LRU/ARC + demand-driven (lookup-triggered) promotion. Plug-in point: `CachePolicy` interface via `eviction_policy` + `cache_policy_module_path`, tiering-manager promotion/cascade logic, and new GPU block-pool retention. |
| Session-aware prefetching on conversation migration | Split: llm-d triggers, vLLM executes | Router picks the destination (routing); destination vLLM engine pulls hottest blocks via fs/p2p tiers before the next turn. Directive passes through per-request `kv_transfer_params` (`sampling_params.extra_args`). |

Temperature export to llm-d: the doc's "Temperature information can also be exported to the llm-d endpoint picker" means vLLM computes temperature and exports a summary; llm-d adds an EPP scorer consuming it for session-aware routing.

**Gap:** session-level features (conversation turn count, reuse ratio, inter-turn intervals, session recency, session duration) are tracked on neither side today — llm-d sessions are token-based affinity only. Net-new data collection.

## Relevant infrastructure

- vLLM KV Events (BlockStored, BlockRemoved, AllBlocksCleared) provide raw signal.
- llm-d-router `latencypredictorclient` (predictedlatency dataproducer) has the XGBoost prediction infrastructure (native tree parsing + Python training server) to extend.
- Block prefetch mechanisms in vLLM (TieringOffloadingManager, LMCache MP Connector).

## Background

Draws on activity-based storage from high-frequency trading systems (Ramesh Doddaiah: 15-20M IOPS, less than 50us latency, 350-400 GB/s).

## Next steps (2026-07-24)

1. Enumerate all KV cache events in llm-d codebase.
2. Define feature set for time series model.
3. Build XGBoost prototype for hot-block prediction.
4. Design ablation study for model comparison.

## Related

- vLLM KV Events canonical form
- vLLM KV block prefetch architecture
- vLLM and llm-d-router KV cache responsibility split
- CephFS performance tuning for KV cache offloading

## Source

Ramesh 1:1 meeting, 2026-07-24. Implementation placement verdict: session `ses_03d89d8f2ffezttCL70TiLlrIv` (2026-08-02).