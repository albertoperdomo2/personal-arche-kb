---
title: "V2.1 — Residency- and Deadline-Gated Admission Prefetch Implementation Guide"
date: "2026-08-19"
type: implementation-guide
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
phase: "V2.1 — residency/deadline admission prefetch"
status: "active"
supersedes: "2026-08-18 initial version (event-driven temperature heuristic)"
revised-after: "[[04 - Theoretical Validation]]"
codebase: "vllm-project/vllm @ v0.27.0 (commit 4bdc8a788d2e2ce9165d552b3d4d8b72604626bf)"
workload: "semianalysisai/cc-traces-weka-062126 (pinned)"
---

# V2.1 — Residency- and Deadline-Gated Admission Prefetch Implementation Guide

Build guide for the V2.1 demonstrable heuristic: prefetch of **ordered, contiguous prefix bundles** at request admission, gated by verified residency and predicted lead time, using non-evicting speculative allocation. Revised 2026-08-19 after [[04 - Theoretical Validation]] — the eight start gates in [[02 - Phased Plan]] must be satisfied before implementation begins.

All file paths and line numbers verified against `vllm-project/vllm` tag `v0.27.0` (commit `4bdc8a78`) via the GitHub Connector on 2026-08-18. Line numbers marked ≈ are ±3.

> **Base tree.** The local V1 working tree (`v0.27.0-prefetch-p1` lineage: repaired manager `admission_prefetch_chunks` property, `kv_transfer_params.abc_admission_prefetch` channel). V2.1 *replaces* the V1 blind first-N policy; it keeps the channel, the counter discipline, and the bench harness.

## 1. Grounding: what exists at v0.27.0

Upstream `vllm/v1/kv_offload` contains **no prefetch code** (verified: zero `prefetch` matches in the module). What exists is a complete demand-driven promotion engine we reuse:

| Asset | Location | What it gives V2.1 |
|---|---|---|
| `OffloadKey` = `block_hash + group_idx` | `vllm/v1/kv_offload/base.py` ≈L26–44 | Chunk identity; one key = `blocks_per_chunk` GPU blocks (e.g. 256 tokens via `block_size`). |
| `ReqContext` | `base.py` ≈L90 | Carries `kv_transfer_params` and per-request scratch (`set_state`/`get_state`) — admission metadata needs **no signature changes**. |
| `ScheduleEndContext(new_req_ids, preempted_req_ids)` | `base.py` ≈L130 | Per-step admission/preemption events delivered to the manager; preemption ids drive cancellation. |
| `OffloadingManager` ABC | `base.py` ≈L224 | Scheduler-side contract: `lookup`, `prepare_load`, `touch`, `prepare_store`, `on_new_request`, `on_request_finished`, `on_schedule_end`. |
| `TieringOffloadingManager` | `tiering/manager.py` ≈L161 | Promotion engine: `lookup()` ≈L282, `_initiate_promotion()` ≈L380, `_flush_pending_promotions()` ≈L429, `_process_finished_jobs()` ≈L246, `on_schedule_end()` ≈L728. |
| `SecondaryTierManager` ABC + `AsyncLookupManager` | `tiering/base.py`, `tiering/async_lookup.py` | `lookup` may return `RETRY`/`None` while a batched async lookup is pending; batches flush at `on_schedule_end`; `cleanup(req_id)` on request finish. **Residency is asynchronous — design for it.** |
| `CPUOffloadingManager.prepare_store` | `cpu/manager.py` ≈L169 | **Evicts LRU blocks to make room**; returns `None` only when eviction cannot free enough. The speculative path must not use this unmodified — see §3.4. |
| `LRUCachePolicy.evictable_blocks` | `cpu/policies/lru.py` | OrderedDict of evictable keys. Rank-by-scan is O(n) — no per-key eviction countdowns from this structure. |
| `CachePolicyFactory` (`"lru"`, `"arc"`, `cache_policy_module_path`) | `cpu/policies/factory.py` | **The plugin idiom V2.1 mirrors for prefetch policies.** |
| `OffloadingConnectorScheduler` | `kv_connector/v1/offloading/scheduler.py` L447 | `RequestOffloadState` (L275); `update_offload_keys()` builds the **full future key list at admission** from `req.block_hashes`; `on_new_request` L806; `get_num_new_matched_tokens` L818; `_maximal_prefix_lookup` L547 (sequential per-chunk `manager.lookup` L560, **breaks at first MISS**); `build_connector_meta` L1220 → `manager.on_schedule_end` L1228. |
| Scheduler waiting loop | `vllm/v1/core/sched/scheduler.py` L685–L689 | The lead-time window: requests wait here before first execution; V1 measured avg 6.22 waiting at concurrency 64. |

**Hard constraints (respect these):**

1. `get_num_new_matched_tokens` must remain side-effect free (base.py L465–470; called multiple times per request). **Prefetch triggers do not belong in the lookup path.**
2. `build_connector_meta` must not modify `scheduler_output` (base.py L526–529).
3. `_maybe_process_finished_jobs` is gated to once per step; prefetch submissions must land before the `on_schedule_end` flush.
4. `OffloadingConnector` is `SupportsHMA` — do not break `request_finished_all_groups`.
5. V2.1 is entirely **scheduler-side**: no worker/layerwise changes, no `requires_piecewise_for_cudagraph` impact.
6. **Prefix-chained hashes**: vLLM block hashes chain on their prefix, and the demand scan breaks at the first MISS. Selecting discontinuous chunks produces promotions the demand path can never use. The selection unit is therefore an ordered contiguous prefix bundle, always.

## 2. Definitions

- **Primary-residency frontier `m`**: the index of the first key in a request's ordered key list that is not already primary-resident (`HIT` or `HIT_PENDING`). `[k_0, …, k_{m-1}]` is guaranteed resident and needs no work.
- **Prefix bundle** `B = [k_m, …, k_j]`: an ordered, contiguous slice of the request's key list **beginning at the frontier** `m` and extending through the last contiguous secondary-resident key `j`. The policy chooses the bundle end `j` and which bundles earn budget — it never selects interior or discontinuous keys, and never re-promotes what is already resident.
- **Lead time `H` / remaining lead time `H_remaining`**: `H` is the predicted time from admission to first demand (V2.1 estimator: queue position × calibrated mean admission interval, constants from V2.0, bounded below by 0). `H_remaining = H − elapsed` is recomputed at every async re-drive; all gate and deadline decisions use `H_remaining`, never the stale admission-time estimate.
- **Residency**: per-key membership state on each secondary tier, resolved asynchronously.

**Selection algorithm (explicitly O(n) in the request's key count n):**

1. **Frontier scan**: walk the ordered key list once; per key, `primary_tier.lookup` (O(1) dict get). Stop at the first key that is neither `HIT` nor `HIT_PENDING` → frontier `m`. Contiguity is free: the demand scan (`_maximal_prefix_lookup`, L547) breaks at the first MISS, so nothing before `m` needs work and nothing after a gap is usable.
2. **Candidacy scan**: from `m`, walk once more; per key, issue the secondary tier `lookup` (async where the tier provides `AsyncLookupManager`). Keys resolving `RESIDENT` extend the candidate bundle; the first `ABSENT` key ends it at `j`. Pending keys stay `PENDING_LOOKUP` and are re-driven (§3.2); each key is visited once per pass and re-drives are bounded by deadline expiry.
3. **Gate**: evaluate the deadline and $U([k_m…k_j])$ once per bundle (§3.3).

No LRU rank scans, no nested loops, no per-key eviction countdowns anywhere in the design.

## 3. Design

### 3.1 Components (new file `vllm/v1/kv_offload/tiering/prefetch.py`)

```
PrefetchPolicy             # pluggable unit (mirrors CachePolicyFactory idiom)
├── BundleBuilder          # request key list → candidate prefix bundles
├── ResidencyTracker       # async lookup state machine (§3.2)
├── DeadlineGate           # promote iff H > L_prefetch(B) and U(B) > 0 (§3.3)
├── SpeculativeAllocator   # non-evicting primary reservation (§3.4)
└── Accounting             # terminal partition + transition log (§3.5)
```

`PrefetchPolicy` is registered in a `PrefetchPolicyFactory` (`"admission"` registered; `prefetch_policy_module_path` for external policies), mirroring `CachePolicyFactory`. This is the shape the upstream RFC proposes: a prefetch heuristic is an interchangeable policy, not a fork of the manager.

### 3.2 Residency state machine (required, not optional)

Per candidate bundle, owned by `ResidencyTracker` inside the policy:

```
PENDING_LOOKUP → RESIDENT | ABSENT → GATE → SUBMITTED → READY | LATE | FAILED
```

Rules:

- **Ownership**: the policy owns the pending set; `TieringOffloadingManager` only receives submission calls.
- **Async drive**: `tier.lookup(key, req_context)` returning `RETRY`/pending keeps the bundle in `PENDING_LOOKUP`; the state is re-driven on each `on_schedule_end` (after the tier's async flush) until resolved or expired. Lookup pending is **never** counted as absence.
- **Deadline**: a bundle still pending when its `H_remaining` reaches zero transitions to `LATE` and is cancelled (never submitted).
- **Lead-time recomputation**: at every re-drive, `H_remaining` is recomputed (elapsed since admission, or refreshed from current queue position). Lookup delay consumes lead time; a bundle whose lookup resolves late may correctly fail the deadline it would have passed at admission.
- **Cancellation**: on `on_request_finished` (manager ≈L691) and on `ScheduleEndContext.preempted_req_ids`, cancel the request's pending/unsubmitted bundles; call the tier's `AsyncLookupManager.cleanup(req_id)`.
- **Duplicate suppression**: skip bundles already primary-resident (`primary_tier.lookup` short-circuit), already in `_chunks_being_loaded`, or already in the pending set.
- **Bounded state**: `max_pending_bundles` config; overflow transitions to `capacity_skip`.

### 3.3 Deadline and utility gate (common units)

Promote bundle `B` only if both hold:

1. **Deadline**: $H_{\text{remaining}} > L_{\text{prefetch}}(B)$ — the calibrated transfer time for the bundle (V2.0 constants), so the promotion can complete before first demand. V1's 98.50% lateness at concurrency 32 is what this gate exists to prevent.
2. **Utility**: $U(B) > 0$, where

$$U(B) = p_{\mathrm{use}}(B)\,\mathrm{saved\_critical\_path\_ms}(B) - \Delta Q_{\mathrm{active}}(B) - \mathbb{E}[C_{\mathrm{eviction}}(B)] - C_{\mathrm{failure}}(B)$$

$$\mathrm{saved\_critical\_path\_ms}(B) = \max\left(0,\; L_{\mathrm{demand}}(B) - \max(0,\; L_{\mathrm{prefetch}}(B) - H_{\text{remaining}})\right)$$

All terms in milliseconds (or an explicitly declared equivalent), calibrated in V2.0. With the non-evicting allocator (§3.4), $\mathbb{E}[C_{\mathrm{eviction}}] = 0$ by construction. **Shadow mode**: until V2.0 calibration lands, the policy computes and logs scores/gate decisions but does not submit — V2.1 performance claims are never made on placeholder constants.

### 3.4 Non-evicting speculative allocation (reservation → promotion, single allocation)

Two problems must be solved together: the existing promotion path (`_initiate_promotion` ≈L380 → `primary_tier.prepare_write` → `CPUOffloadingManager.prepare_store` ≈L169) **evicts LRU blocks to make room**, and a policy-side reservation followed by an unmodified `_initiate_promotion` would **allocate twice**. A bounded speculative budget alone does not fix either — it throttles volume but the underlying allocation still evicts. The flow is therefore:

1. **Reserve**: `CPUPrimaryTierOffloadingManager.try_reserve_no_evict(keys, req_context) -> PrepareStoreOutput | None` (new, next to `prepare_store` in `cpu/manager.py`). Allocates only from currently-free blocks — never calls `policy.evict`; returns `None` when free blocks are insufficient. The reservation **is** the allocation: blocks are created with `ref_cnt = -1` (in-flight), which keeps them out of `evictable_blocks`, so the reservation itself cannot be evicted while in flight.
2. **Consume**: `_initiate_promotion(tier, key, req_context, *, reserved: PrepareStoreOutput | None = None)` (extended at ≈L380). When `reserved` is provided, it **skips its internal `prepare_write`** and builds the `PendingPromotion` from the reservation's block ids; when absent (the demand path), behavior is unchanged. Exactly one allocation per promoted key; zero speculative evictions.
3. **Release on cancel**: deadline expiry, request finish, or preemption releases the reservation via `complete_write(keys, req_context, success=False)`, which removes and frees the blocks (existing `complete_store` failure behavior).

A `speculative_max_bytes` budget may be added **on top** as a throttle on total outstanding speculative reservations, but it is a volume limiter, never the safety mechanism.

Telemetry counts `capacity_rejected` and any speculative-caused eviction (target: zero; non-zero is a bug). If a later phase allows speculative eviction, its expected cost enters $U(B)$ explicitly.

### 3.5 Accounting (terminal partition)

Every considered bundle ends in exactly one terminal class:

$$\mathrm{considered} = \mathrm{primary\_redundant} + \mathrm{secondary\_absent} + \mathrm{gate\_reject} + \mathrm{capacity\_skip} + \mathrm{submitted} + \mathrm{lookup\_unresolved}$$

Report `useful/considered` as the stable policy-yield metric, plus `useful/submitted`, readiness at first demand, bytes moved, transfer time, evictions, queue delay, TTFT, E2E, errors, preemptions, completed sessions. State transitions (§3.2) are logged separately from the terminal partition so async retries are never double-counted. Counter names keep the V1 vocabulary where the semantics match (`prefetch_promoted`, `prefetch_useful`, `prefetch_late`, `prefetch_load_failed`) for comparability.

### 3.6 Admission metadata and events

V2.1 consumes only admission-scoped metadata via `req_context.kv_transfer_params` (base.py ≈L90 — no signature changes):

```json
{ "abc_prefetch": true, "abc_session_id": "sess-123", "abc_turn": 4 }
```

`abc_session_id`/`abc_turn` feed bundle-level session bookkeeping only. **Tool-call and handoff events are out of scope for V2.1**: request-scoped metadata cannot arrive early enough to exploit the tool window (`tool_call_start` is knowable only after the model emits it; `tool_call_end` arrives with the next request). V2.2 specifies an out-of-band, session-addressed control API and a versioned session prefix registry; see [[02 - Phased Plan]].

### 3.7 Data flow and precise `on_schedule_end` ordering

At admission:

```
Scheduler.add_request
  → connector.on_new_request(request)                       # scheduler.py L2235 → offloading/scheduler.py L806
      → build/refresh RequestOffloadState.offload_keys      # full future key list (existing code)
      → policy.on_admission(keys, req_context)              # NEW: frontier scan → candidacy scan → PENDING_LOOKUP
```

Per step, inside `manager.on_schedule_end` (≈L728), in this exact order:

```
1. _maybe_process_finished_jobs()        # existing, once per step: completed transfers committed (complete_write/complete_read)
2. drain tier async-lookup results       # after the tiers' own flush; resolved lookups become consumable
3. policy re-drive                       # consume RESIDENT/ABSENT; recompute H_remaining for all pending
                                         # and resolved-ungated bundles; expire H_remaining ≤ 0 → LATE (cancelled)
4. gate evaluation                       # RESIDENT bundles: deadline H_remaining > L_prefetch and U(B, H_remaining) > 0
5. reservation                           # try_reserve_no_evict for gate winners, budget-capped; refusal → capacity_skip
6. submission                            # _initiate_promotion(..., reserved=…) → _pending_load_submissions
7. _flush_pending_promotions()           # existing ≈L429: batched tier.submit_load
```

Steps 2–6 are the policy's `on_schedule_end` hook, invoked by the manager **after** its own completion processing (step 1) and **before** the promotion flush (step 7). This ordering guarantees: lookup results from this step's flush are consumable in the same step; gate decisions always use post-lookup `H_remaining`; and submissions land in the same step's flush batch.

After the promotion lands (`complete_write`), the first schedule attempt sees `HIT`/`HIT_PENDING` in `_maximal_prefix_lookup` (L547) and takes the existing `load_kv_async` → `WAITING_FOR_REMOTE_KVS` → resume path.

The only new manager-facing API is the policy's admission/schedule-end hooks plus the `reserved` parameter on `_initiate_promotion`; everything downstream reuses existing machinery. Batching, once-per-step gating, and dedup come for free.

## 4. Config surface (`kv_connector_extra_config`)

```json
{
  "kv_connector": "OffloadingConnector",
  "kv_role": "kv_both",
  "kv_connector_extra_config": {
    "spec_name": "TieringOffloadingSpec",
    "cpu_bytes_to_use": 274877906944,
    "block_size": 256,
    "secondary_tiers": [{"type": "fs", "root_dir": "/mnt/kvcache", "n_read_threads": 16}],
    "prefetch": {
      "enabled": true,
      "policy": "admission",
      "policy_module_path": null,
      "shadow_mode": true,
      "max_pending_bundles": 256,
      "max_promotions_per_step": 64,
      "speculative_max_bytes": 0
    }
  }
}
```

Plumbed through `TieringOffloadingSpec.__init__` → `TieringOffloadingManager.__init__` (≈L177), mirroring `secondary_tiers`. `shadow_mode` defaults **true** until V2.0 calibration is declared.

## 5. Build order

Each step is independently verifiable; do not skip ahead.

1. **Branch** from the local V1 tree at the repaired-image state. Existing tiering + admission/lookup suites green before any change.
2. **`prefetch.py` skeleton**: `PrefetchPolicy` ABC, `BundleBuilder`, `ResidencyTracker` (state machine + deadlines + cancellation), `DeadlineGate`, `SpeculativeAllocator` interface, `Accounting`. Pure logic, unit-testable without vLLM. *Verify: state-machine transition tests, including expiry→LATE and cancel-on-preempt; bundle contiguity tests.*
3. **Non-evicting reservation + single-allocation promotion**: `try_reserve_no_evict` in `cpu/manager.py` + alias on `CPUPrimaryTierOffloadingManager`; extend `_initiate_promotion` with the `reserved` parameter (skips `prepare_write` when provided); cancellation via `complete_write(success=False)`. *Verify: under a full tier, reservation refuses and the LRU OrderedDict is unchanged; a reserved-then-promoted key allocates exactly one block; a cancelled reservation frees its blocks.*
4. **Config plumbing** through `TieringOffloadingSpec` → manager; policy constructed when `enabled`. *Verify: parse test; disabled-by-default no-op.*
5. **Admission hook**: `policy.on_admission` from `OffloadingConnectorScheduler.on_new_request` (L806) after `offload_keys` are built. *Verify: registry/pending contents in a manager-level test with synthetic requests.*
6. **Schedule-end re-drive + submission**: `policy.on_schedule_end` wired in the §3.7 ordering (after completion processing, before `_flush_pending_promotions`); submissions via `_initiate_promotion(..., reserved=…)`. *Verify: with the `example` in-memory tier, resident bundles are submitted, absent bundles count `secondary_absent`, pending bundles survive to the next step, and a bundle whose `H_remaining` expires mid-lookup transitions to `LATE` without submission.*
7. **Accounting + metrics**: terminal partition counters + transition log, surfaced via `get_stats()` (≈L810) and the connector's `build_prom_metrics`. *Verify: partition sums to `considered` in the example-tier test.*
8. **Shadow-mode run**: deploy with `shadow_mode: true` on the pinned workload (C32/C64); collect gate decisions, H estimates, and would-be submissions. *Verify: no transfers occur; decision logs match offline replay.*
9. **Regression test**: real-manager scheduler test mirroring the V1 wiring test. *Verify: full focused suite green.*
10. **Bench**: image `v0.27.0-prefetch-v2`; cells reactive baseline / V1 N=100 / V2.1; ≥3 paired interleaved reps; node swap; report terminal-partition accounting first, then TTFT/ITL/E2E against the predeclared V2.0 bounds.

## 6. Guard rails

- **Never block the scheduler loop.** Bundle scoring is O(chunks per admission); budget < 1 ms; monitor `vllm:kv_offload_tiering_lookup_sync_delay_seconds`.
- **No eviction by speculative work** (§3.4). Capacity refusal is normal operation, not an error.
- **Deadline discipline**: a bundle that cannot land before predicted demand is never submitted — lateness is a design failure, not bad luck.
- **Failure policy**: transfer failures surface through `get_block_ids_with_load_errors`; respect `kv_load_failure_policy` (`recompute` in bench configs).
- **Scope honesty**: V2.1 is secondary→CPU promotion at admission. No Hot/Warm/Cool/Cold placement claims, no lifecycle-event claims, no routing claims.

## 7. What V2.1 deliberately excludes

Lifecycle events via out-of-band control API, versioned session registry, dynamic gate thresholds (V2.2); retention features, AET-like global pressure, and the separate GPU-placement / CPU-retention / secondary-persistence control surfaces (V2.3, vLLM-only); multi-replica routing, EPP export, session migration (V2.4, post-proof scale-out to llm-d); residual correction, quantization tiers, aLoRA reuse (parked — see [[01 - Strategy and Re-sequencing]]). Worker-side/layerwise changes: none.

## 8. References

- Grounding inspection: `vllm-project/vllm` @ `v0.27.0`, commit `4bdc8a788d2e2ce9165d552b3d4d8b72604626bf` (GitHub Connector, 2026-08-18).
- [[04 - Theoretical Validation]] — the corrections this guide implements.
- [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization]] — literature background.
- [[../Methodology/04 - Phase 1 Queued-Request Oracle Prefetch Implementation Guide]] — the V1 wiring this replaces.
- [[../Methodology/05 - Initial versus Admission-Time Proactive Prefetching]] — why the admission-time trigger won.
