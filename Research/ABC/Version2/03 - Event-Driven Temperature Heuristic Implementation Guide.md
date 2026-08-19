---
title: "V2.1 — Event-Driven Temperature Heuristic Implementation Guide"
date: "2026-08-18"
type: implementation-guide
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
phase: "V2.1 — temperature-gated admission prefetch"
status: "active"
codebase: "vllm-project/vllm @ v0.27.0 (commit 4bdc8a788d2e2ce9165d552b3d4d8b72604626bf)"
workload: "semianalysisai/cc-traces-weka-with-subagents-060826"
---

# V2.1 — Event-Driven Temperature Heuristic Implementation Guide

Build guide for the V2.1 demonstrable heuristic: scored, residency-checked, cost-gated prefetch of KV chunks at request admission. All file paths and line numbers were verified against `vllm-project/vllm` tag `v0.27.0` (commit `4bdc8a78`) via the GitHub Connector on 2026-08-18. Line numbers marked ≈ are ±3.

> **Base tree.** This guide assumes the local V1 working tree (the `v0.27.0-prefetch-p1` lineage with the repaired manager `admission_prefetch_chunks` property and the `kv_transfer_params.abc_admission_prefetch` channel). V2.1 *replaces* the V1 blind first-N policy; it keeps the event channel, the counter discipline, and the bench harness.

## 1. Grounding: what exists at v0.27.0

Upstream `vllm/v1/kv_offload` contains **no prefetch code** (verified: zero `prefetch` matches in the module). What exists is a complete demand-driven promotion engine we reuse:

| Asset | Location | What it gives V2.1 |
|---|---|---|
| `OffloadKey` = `block_hash + group_idx` | `vllm/v1/kv_offload/base.py` ≈L26–44 | Chunk identity; one key = `blocks_per_chunk` GPU blocks (e.g. 256 tokens via `block_size`). **Chunk-level temperature is the native granularity.** |
| `ReqContext` | `base.py` ≈L90 | Carries `kv_transfer_params` and a per-request scratch dict (`set_state`/`get_state`) — the event channel needs **no signature changes**. |
| `ScheduleEndContext(new_req_ids, preempted_req_ids)` | `base.py` ≈L130 | The per-step admission event already delivered to the manager. |
| `OffloadingManager` ABC | `base.py` ≈L224 | The scheduler-side contract: `lookup`, `prepare_load`, `touch`, `prepare_store`, `on_new_request`, `on_schedule_end`, … |
| `TieringOffloadingManager` | `tiering/manager.py` ≈L161 | The promotion engine: `lookup()` ≈L282, `_initiate_promotion()` ≈L380, `_flush_pending_promotions()` ≈L429, `_process_finished_jobs()` ≈L246, `on_schedule_end()` ≈L728. |
| `SecondaryTierManager` ABC | `tiering/base.py` | `lookup` / `submit_load` / `submit_store` / `get_finished_jobs`; `fs` tier has non-blocking `AsyncLookupManager` (`tiering/async_lookup.py`). |
| `LRUCachePolicy.evictable_blocks` | `cpu/policies/lru.py` | OrderedDict of evictable keys — the AET stack-position data source. |
| `CachePolicyFactory` (`"lru"`, `"arc"`, `cache_policy_module_path`) | `cpu/policies/factory.py` | **The plugin idiom V2.1 mirrors for prefetch policies.** |
| `OffloadingConnectorScheduler` | `kv_connector/v1/offloading/scheduler.py` L447 | Owns `RequestOffloadState` (L275); `update_offload_keys()` builds the **full future key list at admission** from `req.block_hashes`; `on_new_request` L806; `get_num_new_matched_tokens` L818; `build_connector_meta` L1220 → `manager.on_schedule_end` L1228. |
| Scheduler waiting loop | `vllm/v1/core/sched/scheduler.py` L685–L689 | The lead-time window: requests sit in `waiting` while tokens are budgeted; V1 measured avg 6.22 waiting at concurrency 64. |

**Hard constraints (respect these):**

1. `get_num_new_matched_tokens` must remain side-effect free (base.py L465–470 docstring; called multiple times per request). **Prefetch triggers do not belong in the lookup path.**
2. `build_connector_meta` must not modify `scheduler_output` (base.py L526–529).
3. `_maybe_process_finished_jobs` is gated to once per step (`_processed_jobs_this_step`); prefetch submissions must land before the `on_schedule_end` flush.
4. `primary_tier.prepare_write` returns `None` when the CPU tier is full — handle as `capacity_skip`, never block.
5. `OffloadingConnector` is `SupportsHMA` — do not break `request_finished_all_groups`.
6. V2.1 is entirely **scheduler-side**: no worker/layerwise changes, so no `requires_piecewise_for_cudagraph` impact.

## 2. Design

### 2.1 Components (new file `vllm/v1/kv_offload/tiering/temperature.py`)

```
TemperaturePolicy          # owns the other three; the pluggable unit
├── SessionRegistry        # session_id → SessionState(keys, status, last_event)
├── AETTracker             # per-key LRU stack-depth samples → time-to-eviction estimate
├── TemperatureScorer      # event override > session affinity > AET background → score ∈ [0,1]
└── CostGate               # admit iff Benefit > N(load) × Cost
```

- **`SessionRegistry`.** Keyed by `abc_session_id` from `kv_transfer_params`. Records the chunk working set per session (the union of `offload_keys` seen for that session), status (`ACTIVE` / `TOOL_CALL` / `DORMANT`), and last event time. Chunk-level cohesion (AgentKVShift locality): temperature is assigned to whole session working sets, not isolated blocks.
- **`AETTracker`.** On every `touch` / policy position change, sample the key's index in `LRUCachePolicy.evictable_blocks`. Travel speed down the LRU stack is monotone non-increasing on average (kinetic model, USENIX ATC'16); estimate drain rate from the OrderedDict length over time and predict `time_to_eviction(key)`. Used only for blocks with **no** event information.
- **`TemperatureScorer`.** Precedence: (1) **event override** — admission of request R ⇒ R's keys are HOT; `tool_call_end` for session S ⇒ S's working set is HOT; (2) **session affinity** — keys of a session with a queued/active request are WARM; (3) **AET background** — everything else scored by predicted time-to-eviction. Output normalized to [0, 1].
- **`CostGate`.** $\text{Benefit} = P_{access} \times (T_{prefill} - T_{fetch})$ vs. $N(load) \times (\frac{S_{chunk}}{B_{link}} \times \lambda_{queue} + C_{evict})$. V2.1 ships with static constants measured in the repaired-image run; V2.0's sweep replaces them with the measured cost curve; V2.2 makes $N$ dynamic.

### 2.2 Data flow at admission (the V2.1 critical path)

```
Scheduler.add_request
  → connector.on_new_request(request)                      # scheduler.py L2235 → offloading/scheduler.py L806
      → build/refresh RequestOffloadState.offload_keys     # full future key list (existing code)
      → manager.register_session_keys(keys, req_context)   # NEW: policy registers + scores
          → per candidate chunk: residency check           # tier.lookup(key, req_context) on each secondary tier
              → HIT    → _initiate_promotion(tier, key)    # existing ≈L380 (prepare_write + defer)
              → MISS   → count residency_skip, drop        # fixes V1's 87% load-failure mode
              → RETRY  → defer to next step's pass         # async lookup pending
  ... request waits (lead-time window) ...
  → end of step: manager.on_schedule_end(context)          # existing ≈L728
      → _flush_pending_promotions()                        # existing ≈L429: batched tier.submit_load
  ... promotion lands (complete_write) ...
  → first schedule attempt: _maximal_prefix_lookup         # existing L547 sees HIT/HIT_PENDING
      → load_kv_async path → WAITING_FOR_REMOTE_KVS → resume
```

The only new manager API is `register_session_keys(keys, req_context)`; everything downstream reuses existing machinery. The once-per-step batching, dedup, and capacity guards come for free.

### 2.3 Event schema (`kv_transfer_params`)

Consistent with the V1 `abc_admission_prefetch` convention:

```json
{
  "abc_prefetch": true,
  "abc_session_id": "sess-123",
  "abc_turn": 4,
  "abc_event": "tool_call_start | tool_call_end | agent_handoff | null",
  "abc_expected_resume_s": 5.0
}
```

V2.1 consumes `abc_prefetch`, `abc_session_id`, `abc_turn` only. `abc_event` / `abc_expected_resume_s` are parsed and logged but acted on in V2.2. The manager reads them from `req_context.kv_transfer_params` (base.py ≈L90) — no connector or API changes.

### 2.4 Config surface (`kv_connector_extra_config`)

```json
{
  "kv_connector": "OffloadingConnector",
  "kv_role": "kv_both",
  "kv_connector_extra_config": {
    "spec_name": "TieringOffloadingSpec",
    "cpu_bytes_to_use": 274877906944,
    "block_size": 256,
    "secondary_tiers": [{"type": "fs", "root_dir": "/mnt/kvcache", "n_read_threads": 16}],
    "temperature_prefetch": {
      "enabled": true,
      "max_promotions_per_step": 64,
      "cost_gate_n": 1.0,
      "aet_window_s": 60,
      "policy_module_path": null
    }
  }
}
```

`TieringOffloadingSpec.__init__` (`tiering/spec.py`) reads the dict and passes it to `TieringOffloadingManager.__init__` (≈L177), mirroring how `secondary_tiers` is plumbed today.

### 2.5 The "right shape": pluggable policy idiom

The codebase already has the plugin pattern: `CachePolicyFactory` (`cpu/policies/factory.py`) registers `"lru"`/`"arc"` and accepts `cache_policy_module_path`. V2.1 mirrors it: a `PrefetchPolicy` ABC in `temperature.py` + `PrefetchPolicyFactory` with `"temperature"` registered and `policy_module_path` for external policies. This is the shape the upstream RFC will propose — a heuristic is an interchangeable policy, not a fork of the manager.

## 3. Build order

Each step is independently verifiable; do not skip ahead.

1. **Branch** from the local V1 tree at the repaired-image state. Run the existing tiering + admission/lookup suites; all green before any change.
2. **`temperature.py` skeleton**: `PrefetchPolicy` ABC, `TemperaturePolicy`, `SessionRegistry`, `AETTracker`, `TemperatureScorer`, `CostGate`, `PrefetchPolicyFactory`. Pure logic, unit-testable without vLLM. *Verify: unit tests for scorer precedence (event > session > AET) and gate arithmetic.*
3. **Config plumbing**: `temperature_prefetch` dict through `TieringOffloadingSpec` → `TieringOffloadingManager.__init__`; policy constructed when `enabled`. *Verify: config parse test; disabled-by-default no-op.*
4. **Session registration**: `TieringOffloadingManager.register_session_keys(keys, req_context)`; call it from `OffloadingConnectorScheduler.on_new_request` (L806) after `offload_keys` are built. *Verify: registry contents in a manager-level test with synthetic requests.*
5. **Residency-checked prefetch pass**: in `register_session_keys`, score → gate → per-tier `lookup` → `_initiate_promotion` for winners. *Verify: with the `example` in-memory tier, pre-stored keys get promoted, absent keys count `residency_skip`.*
6. **AET tracker**: sample `evictable_blocks` positions on `touch`; feed scores for non-event blocks. *Verify: synthetic LRU drift produces monotone time-to-eviction estimates.*
7. **Metrics**: counters `prefetch_attempted / promoted / useful / late / redundant / load_failed` (V1 names, for comparability) + `prefetch_residency_skip / capacity_skip / gate_reject` + score histogram; surface via `get_stats()` (≈L810) and the connector's `build_prom_metrics`. *Verify: counters move correctly in the example-tier test.*
8. **Regression test**: real-manager scheduler test mirroring the V1 wiring test (the one that caught the `_admission_prefetch_chunks` mismatch). *Verify: full focused suite green.*
9. **Bench image**: build `v0.27.0-prefetch-v2` immutable image; deploy single replica; AgentX Weka concurrency 32 and 64; cells: reactive baseline, V1 N=100, V2 heuristic; ≥3 paired interleaved reps; swap `…-6kl5z` / `…-mt46x` assignments.
10. **Report**: selection accounting first (useful/attempted, load_failed/promoted, late/promoted, redundant/attempted, gate rejects), then TTFT/ITL. Go/no-go against the V2.1 exit criteria in [[02 - Phased Plan]].

## 4. Guard rails

- **Never block the scheduler loop.** The scoring pass is O(chunks per admitted request); budget < 1 ms per admission. Monitor the existing `vllm:kv_offload_tiering_lookup_sync_delay_seconds` histogram.
- **Capacity**: `prepare_write → None` ⇒ `capacity_skip`, try next step. Never evict to make room for a prefetch in V2.1.
- **Dedup**: skip keys already in `_chunks_being_loaded` or primary-resident (`primary_tier.lookup` first — it short-circuits, `tiering/manager.py` ≈L282 step 2).
- **Failure policy**: promotion failures surface through `get_block_ids_with_load_errors`; respect `kv_load_failure_policy` (`recompute` in bench configs).
- **Preemptions**: `ScheduleEndContext.preempted_req_ids` — a preempted request's session stays registered; its keys keep their event temperature (it will be re-admitted).

## 5. What V2.1 deliberately excludes

Tool-call demotion and multi-hop pre-positioning (V2.2); dynamic gate threshold (V2.2); EPP temperature export (V2.3); residual correction, quantization tiers, aLoRA reuse (parked future work — see [[01 - Strategy and Re-sequencing]]). Worker-side/layerwise changes: none.

## 6. References

- Grounding inspection: `vllm-project/vllm` @ `v0.27.0`, commit `4bdc8a788d2e2ce9165d552b3d4d8b72604626bf` (GitHub Connector, 2026-08-18).
- [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization]] — design rationale and literature.
- [[../Methodology/04 - Phase 1 Queued-Request Oracle Prefetch Implementation Guide]] — the V1 wiring this replaces.
- [[../Methodology/05 - Initial versus Admission-Time Proactive Prefetching]] — why the admission-time trigger won.
