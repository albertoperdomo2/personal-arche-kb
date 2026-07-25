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

## Relevant infrastructure

- vLLM KV Events (BlockStored, BlockRemoved, AllBlocksCleared) provide raw signal.
- llm-d predicted_latency_model.py has XGBoost training pipeline to extend.
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
- CephFS performance tuning for KV cache offloading

## Source

Ramesh 1:1 meeting, 2026-07-24.
