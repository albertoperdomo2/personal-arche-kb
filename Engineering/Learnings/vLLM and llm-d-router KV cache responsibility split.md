---
title: vLLM and llm-d-router KV cache responsibility split
date: 2026-08-02
type: learning
topic: KV cache offloading
repos: vllm-project/vllm, llm-d/llm-d-router
---

# vLLM and llm-d-router KV cache responsibility split

Which component owns what in the KV cache stack, verified by direct inspection of `vllm-project/vllm` and `llm-d/llm-d-router` at `main` (2026-08-02). This is the map used to decide where new cache-management features (e.g. the Activity-Based KV Cache Tier Placement / ABC proposal) should be implemented.

## Headline: vLLM owns the blocks, llm-d-router consumes placement state

vLLM is the only component that owns physical KV cache blocks across all storage tiers and can move data between them. The llm-d router is a control plane: it ingests vLLM KV events into a global block-location index and uses that index to score endpoints for routing, but it cannot move a single byte of KV data.

Consequence for new features: anything that changes **where blocks live or when they move** (placement policy, migration, prefetch execution) belongs in core vLLM; the router contributes only consumers and triggers (scorers, routing decisions, migration coordination).

## vLLM side — engine owns all four tiers

| Tier | Code | Notes |
|---|---|---|
| GPU HBM | `vllm/v1/core/block_pool.py`, `vllm/v1/core/single_type_kv_cache_manager.py` | ref_cnt + free-queue eviction, demand-driven; no temperature-aware retention |
| CPU DRAM (primary tier) | `vllm/v1/kv_offload/cpu/manager.py` | `CPUOffloadingManager` with **pluggable `CachePolicy`**: built-in `lru`/`arc`, or out-of-tree via `eviction_policy` + `cache_policy_module_path` (`vllm/v1/kv_offload/cpu/policies/base.py`: `get/insert/remove/touch/evict`) |
| NVMe / CephFS (fs secondary tier) | `vllm/v1/kv_offload/tiering/fs/manager.py` | `FileSystemTierManager` — the `/mnt/nvme-kv-cache` and `/mnt/kv_cache` fs tiers used in the offload benchmarks |
| Distributed (secondary tiers) | `vllm/v1/kv_offload/tiering/p2p/` (NIXL), `tiering/obj/` (object store) | |

- **Orchestration:** `TieringOffloadingManager` (`vllm/v1/kv_offload/tiering/manager.py`) — cascade-all-on-store, staged promotion, but promotion is **demand-driven only** (triggered by `lookup()` at request admission). No cost-benefit model, no priority ordering, no proactive/prefetch placement.
- **Config surface:** `TieringOffloadingSpec` (`vllm/v1/kv_offload/tiering/spec.py`) — `eviction_policy`, `cache_policy_module_path`, `secondary_tiers` (types: `example`/`fs`/`p2p`/`obj`).

Telemetry already close to what activity-based placement needs:

- `KVCacheMetricsCollector` (`vllm/v1/core/kv_cache_metrics.py`) — per-block `record_access()`, `get_idle_time_seconds()`, `get_reuse_gaps_seconds()`; sampled (default 0.01). This is most of the *temporal* feature set for temperature prediction.
- The `OffloadingManager` interface sees every block access (`lookup`, `touch`, `prepare_load`, `complete_load`) and every placement change (`prepare_store`, `complete_store`, eviction) in real time.

## llm-d-router side — control plane, consumer of placement state

- **Global KV-block index:** `pkg/kvcache/kvblock/index.go` — tracks which pod (`PodEntry`) holds which KV-block on which `DeviceTier` (gpu/cpu/disk) with a last-update timestamp; populated from KV events via `pkg/kvevents` (ZMQ subscriber).
- **Tier weights for routing scores:** `pkg/kvcache/backend.go` (`KVCacheBackendConfig`, e.g. gpu=1.0, cpu=0.8) used by `LongestPrefixScorer`.
- **EPP plugins relevant to KV locality:** filters/scorers `sessionaffinity`, `mmcacheaffinity`, `kvcacheutilization`; dataproducers `preciseprefixcache` (uses `kvcache.NewKVCacheIndexer` to score pods by precise prefix-cache hit), `sessionid`, `predictedlatency`.
- **Disaggregation sidecar:** `pkg/sidecar/proxy` *coordinates* P/D and E/P/D KV/embedding transfers via headers (e.g. `x-prefiller-host-port`); the actual data movement is executed by vLLM's connectors (NIXL / OffloadingConnector).
- **XGBoost ML infrastructure already present:** `pkg/epp/framework/plugins/requestcontrol/dataproducer/predictedlatency/latencypredictorclient/` — native XGBoost trees parsing (`UseNativeXGBoost`), async prediction client, Python training server via `TrainingURL`/`PredictionURLs`. This is the concrete home of the "existing llm-d infrastructure" (`predicted_latency_model.py`) referenced by the ABC meeting notes.

## The two control channels between the components

- **vLLM → llm-d:** KV events over ZMQ — `BlockStored`, `BlockRemoved`, `AllBlocksCleared` — tagged with `medium` (GPU/CPU/STORAGE) and `locality` (LOCAL/REMOTE), plus `token_ids`, `block_size`, `group_idx`, `kv_cache_spec_kind`. Canonical form: [[vLLM KV Events canonical form]].
- **llm-d → vLLM:** per-request `kv_transfer_params` via `sampling_params.extra_args` (OpenAI-compatible `extra_body`), e.g. `transfer_id`, `remote_engine_id`, and tier filters like `{"kv_load_tiers": [{"medium": "STORAGE", "locality": "LOCAL"}]}` — parsed in the offloading connector scheduler and matched against tier metadata.

## Where ABC goes (2026-08-02 verdict)

| ABC component | Where | Integration point |
|---|---|---|
| **Temperature prediction** (XGBoost, temporal/spatial/session features) | **core vLLM** (`vllm/v1/kv_offload`) | Per-block access telemetry and placement changes are visible only in the engine; temporal features (frequency, recency, inter-access gaps) already exist in `KVCacheMetricsCollector`. Session features are net-new (see gap below). |
| **Cost-aware tier placement** (Hot→GPU / Warm→CPU / Cool→NVMe / Cold→CephFS; `Benefit > N × Cost`; priority scheduler; async multi-hop prefetch) | **core vLLM** | Replaces today's cascade-all + LRU + demand-driven promotion. Plug-in point: `CachePolicy` (`eviction_policy` + `cache_policy_module_path`) for the CPU tier, tiering-manager promotion/cascade logic, and new retention logic in the GPU block pool. |
| **Session-aware prefetching** on conversation migration | **Split**: llm-d triggers, vLLM executes | The migration decision is routing (llm-d-router); the destination engine must pull blocks into its tiers (vLLM fs/p2p). `kv_transfer_params` is the existing channel to pass the directive into vLLM. |

- llm-d-router's contribution: an EPP scorer consuming **exported temperature info** for session-aware routing (the ABC doc's "Temperature information can also be exported to the llm-d endpoint picker"), plus the session-migration trigger. The temperature export direction means vLLM computes temperature and publishes a summary; the router does not compute placement.
- The XGBoost *pattern* (Python server + client with native tree parsing) exists in llm-d-router's `predictedlatency` dataproducer, but for *placement* the model output must reach vLLM's offload managers, so the prediction engine itself belongs in vLLM.

## Gap: session-level features tracked nowhere today

Neither side tracks conversation turn count, reuse ratio, inter-turn intervals, session recency, or session duration. llm-d-router "sessions" are only token-based affinity (pin to pod via `x-session-token`); vLLM sees request-level streams only. ABC's session features are net-new data collection on whichever side owns them.

## Related

- [[vLLM KV offload retrieval path - lookup, promotion, and load]]
- [[vLLM KV Events canonical form]]
- [[vLLM KV block prefetch architecture]]
- [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]]
- Activity-Based KV Cache Offloading (research concept, `Research/KV Cache Offloading/`)

## Provenance

Direct inspection of `vllm-project/vllm` and `llm-d/llm-d-router` clones at `main` (2026-08-02), session `ses_03d89d8f2ffezttCL70TiLlrIv`. Claims spot-checked in the clones: `vllm/v1/kv_offload/cpu/policies/` (lru/arc/factory), `tiering/spec.py` (`eviction_policy`/`cache_policy_module_path`/`secondary_tiers`), `vllm/v1/core/kv_cache_metrics.py` (`record_access`/idle/reuse gaps), llm-d-router `pkg/kvcache/kvblock/index.go` (`DeviceTier`), `latencypredictorclient/` (XGBoost).