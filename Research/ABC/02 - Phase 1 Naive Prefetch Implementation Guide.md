---
title: "Phase 1 — Naive Proactive Prefetch Implementation Guide"
date: "2026-08-11"
type: implementation-guide
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
phase: "1 — naive proactive prefetching (toy)"
status: "draft"
depends-on: "[[01 - Experiment Definition]] (Phase 1)"
baseline: "[[2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]]"
codebase: "vllm-project/vllm @ main (inspected 2026-08-11)"
---

# Phase 1 — Naive Proactive Prefetch of N Blocks

This guide explains, step by step, how to implement the first proactive-fetching step of the ABC project: a toy prefetcher that proactively promotes **N** KV cache chunks from a secondary tier (NVMe / CephFS) to the CPU primary tier before they are requested, and measures whether request-visible latency improves.

It is the implementation counterpart of **Phase 1** in [[01 - Experiment Definition]]. It is deliberately a toy: no prediction model, no cost-benefit gate, no session awareness. The only knob is **N**, the number of chunks to read-ahead.

> **Code references.** All file paths, class names, and method names below are verified against `vllm-project/vllm` at `main`, inspected 2026-08-11 via the GitHub connector. Code blocks are illustrative sketches, not copy-paste patches — line numbers drift; re-read each file before editing.

## 1. How fetching works today (the reactive baseline)

Understanding the reactive path is the whole prerequisite, because the toy reuses its machinery. The path spans three files:

| Layer | File | Class / function |
|---|---|---|
| Connector (scheduler side) | `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py` | `OffloadingConnectorScheduler` |
| Tiering manager (scheduler side) | `vllm/v1/kv_offload/tiering/manager.py` | `TieringOffloadingManager` |
| Secondary tier interface | `vllm/v1/kv_offload/tiering/base.py` | `SecondaryTierManager` |

### 1.1 The per-request sequential scan

When the scheduler considers a request, it calls `OffloadingConnectorScheduler.get_num_new_matched_tokens()` (scheduler.py). That builds the request's full list of `offload_keys` from `req.block_hashes` (`RequestOffloadState.update_offload_keys`) and then calls `_lookup()` → `_lookup_complete_chunks()` → `_maximal_prefix_lookup()`.

`_maximal_prefix_lookup` (scheduler.py) is the heart of the reactive behavior:

```python
def _maximal_prefix_lookup(self, keys, req_context, req, group_config, start_chunk_idx):
    hit_count = 0
    defer_lookup = False
    for local_idx, key in enumerate(keys):
        result = self.manager.lookup(key, req_context)
        match result:
            case LookupResult.HIT:        hit_count += 1
            case LookupResult.HIT_PENDING: defer_lookup = True; hit_count += 1
            case LookupResult.RETRY:      defer_lookup = True      # keep scanning
            case LookupResult.MISS:       break                    # STOP the scan
    return hit_count if not defer_lookup else None
```

Two properties matter:

1. **The full future key list is known.** `keys` is the request's complete `offload_keys` slice — every chunk the request will eventually need is already in this list.
2. **The scan stops at the first MISS.** Only the prefix already resident in the primary tier is counted as hit; the first missing chunk terminates the scan and defers the request.

### 1.2 What `lookup()` does on a primary miss

`TieringOffloadingManager.lookup()` (manager.py) checks the CPU primary tier first. On a primary MISS it walks the secondary tiers; on the first secondary HIT it calls `_initiate_promotion()`:

```python
def _initiate_promotion(self, tier_idx, key, req_context) -> bool:
    # Allocate a primary slot NOW (ref_cnt = -1 => "in-flight"),
    # so later lookups in the same step see the slot as pending.
    primary_write_result = self.primary_tier.prepare_write([key], req_context)
    if primary_write_result is None:
        return False                       # primary full -> treat as unavailable
    # Defer the actual submit_load() to on_schedule_end() so all blocks
    # queued during one step are submitted as ONE batched job per (tier, req).
    entry = self._pending_load_submissions.setdefault(tier_idx, {})\
                                           .setdefault(req_context.req_id, ...)
    entry.keys.extend(primary_write_result.keys_to_store)
    entry.block_ids.extend(store_spec.block_ids)
    return True
```

Then `lookup()` returns `RETRY`, which makes `_maximal_prefix_lookup` set `defer_lookup = True` and return `None`, which makes `get_num_new_matched_tokens` return `(None, ...)`, which **defers the request** — it is not scheduled this step.

### 1.3 The promotion is flushed at end of step

At the end of each scheduler step, `build_connector_meta()` (scheduler.py) calls `manager.on_schedule_end()` (manager.py), which calls `_flush_pending_promotions()`:

```python
def _flush_pending_promotions(self):
    for tier_idx, pending_by_ctx in self._pending_load_submissions.items():
        tier = self.secondary_tiers[tier_idx]
        for entry in pending_by_ctx.values():
            job = TransferJob(job_id=..., keys=entry.keys,
                              block_ids=..., is_promotion=True, ...)
            self._register_job(job, tier_idx)
            tier.submit_load(job)           # async secondary -> primary copy
    self._pending_load_submissions.clear()
```

`tier.submit_load()` is non-blocking; the copy runs asynchronously. Completion is polled on later steps by `_maybe_process_finished_jobs()` → `_complete_promotion()` → `primary_tier.complete_write()`, which flips the slot from in-flight (`ref_cnt = -1`) to readable (`ref_cnt = 0`). Only then does `lookup()` return `HIT` for that chunk.

### 1.4 The reactive cost

Putting 1.1–1.3 together: **only the one chunk that caused the MISS is promoted per step.** A request that needs K chunks from a secondary tier is deferred ~K times, and each deferral pays the secondary-tier lookup delay. The Nemotron baseline report ([[2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]]) measured CephFS/NVMe P50 lookup delays of ~2.2 s and P99 hitting the 10 s histogram ceiling. That multi-second, per-chunk, sequential cost is exactly what this toy targets.

## 2. The toy design: spatial read-ahead of N chunks on first miss

### 2.1 Why read-ahead (not a background scan)

The experiment definition lists two candidate toy strategies: prefetch the N most-recently/frequently-accessed blocks (temporal), or read-ahead. **Read-ahead is the right first cut** because:

- It **piggybacks on real demand**: the trigger is an actual miss for a block the request is about to need. It can never waste a promotion on a block no live request wants.
- It **reuses the exact existing promotion machinery** (`_initiate_promotion` → `_pending_load_submissions` → `_flush_pending_promotions` → `submit_load`). No new transfer path, no background thread, no new worker.
- The connector scheduler **already holds the full future key list** (`offload_keys`), so "the next N chunks" is a trivial slice — no access-history tracking needed yet (that comes in Phase 2).
- It is a **single knob** (N), which is exactly what Phase 1's exit criterion requires sweeping.

### 2.2 The change in one sentence

When `_maximal_prefix_lookup` hits the first secondary-tier MISS for a request, instead of promoting only that one chunk, proactively promote the **next N chunks** of the same request from the secondary tier to the primary tier in the same step — collapsing ~K deferred steps into ~⌈K/N⌉.

### 2.3 Where the hook goes

The call-flow today and after the change:

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Reactive vs. toy-prefetch call flow",
  "width": 720,
  "height": 180,
  "background": "white",
  "data": {
    "values": [
      {"x": 1, "y": 1, "label": "scan keys in order", "flow": "Reactive (today)"},
      {"x": 2, "y": 1, "label": "first MISS", "flow": "Reactive (today)"},
      {"x": 3, "y": 1, "label": "promote 1 chunk", "flow": "Reactive (today)"},
      {"x": 4, "y": 1, "label": "defer request", "flow": "Reactive (today)"},
      {"x": 5, "y": 1, "label": "next step: re-scan", "flow": "Reactive (today)"},
      {"x": 1, "y": 0, "label": "scan keys in order", "flow": "Toy prefetch (Phase 1)"},
      {"x": 2, "y": 0, "label": "first MISS", "flow": "Toy prefetch (Phase 1)"},
      {"x": 3, "y": 0, "label": "promote 1 + next N chunks", "flow": "Toy prefetch (Phase 1)"},
      {"x": 4, "y": 0, "label": "defer request", "flow": "Toy prefetch (Phase 1)"},
      {"x": 5, "y": 0, "label": "next step: N more chunks hit", "flow": "Toy prefetch (Phase 1)"}
    ]
  },
  "layer": [
    {
      "mark": {"type": "line", "point": true, "strokeWidth": 2},
      "encoding": {
        "x": {"field": "x", "type": "ordinal", "title": "Step", "axis": {"labelAngle": 0}},
        "y": {"field": "y", "type": "ordinal", "title": null, "axis": null,
              "scale": {"domain": [0, 1]}},
        "color": {"field": "flow", "type": "nominal",
                  "scale": {"range": ["#1f77b4", "#d62728"]},
                  "legend": {"title": "Flow"}}
      }
    },
    {
      "mark": {"type": "text", "dy": -14, "fontSize": 10, "angle": 0},
      "encoding": {
        "x": {"field": "x", "type": "ordinal"},
        "y": {"field": "y", "type": "ordinal"},
        "text": {"field": "label"}
      }
    }
  ]
}
```

The figure contrasts the two flows: reactive promotes one chunk per deferred step; the toy promotes N, so each subsequent re-scan advances N chunks instead of one.

## 3. Step-by-step implementation

The change touches three files. Each step is independently testable.

### Step 1 — Add the `prefetch_chunks` (N) config knob

**File:** `vllm/v1/kv_offload/tiering/spec.py` (and the config plumbing).

`TieringOffloadingSpec` reads its behavior from `kv_connector_extra_config` (see the docstring at the top of `spec.py`). Add a new key:

- `prefetch_chunks` (int, default `0`): number of additional chunks to proactively promote on the first secondary-tier miss. `0` reproduces the reactive baseline exactly — this is the control value for the sweep.

Plumb it through:

1. Read it in `TieringOffloadingSpec.__init__` from `self.extra_config.get("prefetch_chunks", 0)`, validate it is a non-negative int, and store as `self.prefetch_chunks`.
2. Pass it into the `TieringOffloadingManager` constructor (`spec.py`, `get_manager()`), and store as `self._prefetch_chunks` on the manager.
3. Expose it on the manager as a read-only property so the connector scheduler can read it without a second plumbing path: `manager.prefetch_chunks`.

> **Why default 0:** the experiment is a comparison against the Phase 0 baseline. `N = 0` must be byte-for-byte the existing behavior so the sweep has a clean control. Confirm this with a paired `N = 0` run in the batch.

### Step 2 — Add a `_try_promote` helper and a `prefetch` method on the manager

**File:** `vllm/v1/kv_offload/tiering/manager.py`.

Factor the "find in a secondary tier and initiate promotion" body of `lookup()` into a reusable helper, then expose a batched `prefetch()`:

```python
def _try_promote(self, key, req_context, exclude_tier_idx=None) -> bool:
    """Return True if a promotion was initiated (or already in-flight/hit).

    Mirrors the secondary-scan body of lookup(): primary HIT/HIT_PENDING and
    already-in-flight promotions are no-ops; the first secondary HIT initiates
    a promotion; secondary RETRY/MISS returns False.
    """
    primary_hit = self.primary_tier.lookup(key, req_context)
    if primary_hit in (LookupResult.HIT, LookupResult.HIT_PENDING):
        return True                                   # already primary-resident
    for i, tier in enumerate(self.secondary_tiers):
        if i == exclude_tier_idx:
            continue
        if not req_context.load_tier_filter.allows(tier.medium, tier.locality):
            continue
        result = tier.lookup(key, req_context)
        if result is LookupResult.HIT:
            return self._initiate_promotion(i, key, req_context)
    return False

def prefetch(self, keys, req_context) -> int:
    """Proactively promote up to len(keys) chunks that are in a secondary tier.

    Toy Phase 1: called by the connector scheduler with the next N keys after
    the first miss. Returns the number of promotions actually initiated.
    Skips blocks already primary-resident, blocks not in any secondary tier,
    and blocks the primary tier had no room for.
    """
    initiated = 0
    for key in keys:
        if self._try_promote(key, req_context):
            initiated += 1
    return initiated
```

Notes:

- `_try_promote` is a pure refactor of the secondary-scan branch of `lookup()`; `lookup()` itself should be rewritten to call `_try_promote` for the single-key case so the two paths cannot drift. Keep `lookup()`'s return-type semantics (`HIT` / `HIT_PENDING` / `RETRY` / `MISS`) identical — callers depend on them.
- The helper **respects `load_tier_filter`** exactly as `lookup()` does, so per-request tier restrictions (e.g. `kv_load_tiers` in `kv_transfer_params`) are honored by prefetch too.
- It **tolerates primary-full**: `_initiate_promotion` returns `False` when `primary_tier.prepare_write()` returns `None`; `prefetch` just skips that key. No exception, no deadlock.
- It **does not double-promote**: a block already in-flight (`HIT_PENDING`, `ref_cnt = -1`) returns `True` without calling `prepare_write` again, so it never allocates a second primary slot for the same key.

### Step 3 — Call `prefetch` from the connector scheduler on the first miss

**File:** `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`.

Modify `_maximal_prefix_lookup` so that, on the first `MISS`, it asks the manager to proactively promote the next N keys. The change is localized to the `MISS` branch:

```python
def _maximal_prefix_lookup(self, keys, req_context, req, group_config, start_chunk_idx):
    hit_count = 0
    defer_lookup = False
    keys = list(keys)                   # ensure sliceable for prefetch
    for local_idx, key in enumerate(keys):
        result = self.manager.lookup(key, req_context)
        match result:
            case LookupResult.HIT:         hit_count += 1
            case LookupResult.HIT_PENDING: defer_lookup = True; hit_count += 1
            case LookupResult.RETRY:      defer_lookup = True
            case LookupResult.MISS:
                # ---- Phase 1 toy prefetch hook ----
                n = getattr(self.manager, "prefetch_chunks", 0)
                if n > 0:
                    upcoming = keys[local_idx + 1 : local_idx + 1 + n]
                    if upcoming:
                        self.manager.prefetch(upcoming, req_context)
                break
    return hit_count if not defer_lookup else None
```

Why this placement is correct:

- `keys[local_idx + 1 : ...]` are the chunks **immediately after** the first miss — the ones the scan would have reached next. They are guaranteed to belong to the same request and the same KV group, so they are valid `OffloadKey`s for the same `req_context`.
- It fires **once per scan-break**, not once per chunk, so a request needing K chunks gets at most ⌈K/N⌉ prefetch calls total (one per deferred re-scan), each promoting up to N+1 chunks (the miss + the read-ahead).
- It runs **before** `on_schedule_end()`, so the prefetched promotions land in `_pending_load_submissions` and are flushed in the **same** `_flush_pending_promotions()` batch as the demand promotion — one batched `submit_load()` per (tier, request), exactly as today.
- `getattr(..., "prefetch_chunks", 0)` keeps the scheduler robust if a non-tiering manager (e.g. `CPUOffloadingSpec`) is in use — it falls back to `0`, i.e. the reactive baseline.

> **Sliding-window groups:** `_sliding_window_lookup` (scheduler.py) scans from the end and has different semantics. Do **not** add the prefetch hook there in Phase 1 — restrict the toy to full-attention groups (`_maximal_prefix_lookup`). Sliding-window read-ahead needs its own analysis and belongs in Phase 2.

### Step 4 — Add a prefetch counter metric

**File:** `vllm/v1/kv_offload/tiering/base.py` (metric name) and `metrics.py` (tracker), mirroring the existing `PROMOTION_ALLOCATION_FAILURES` pattern.

Add a counter `vllm:kv_offload_tiering_prefetch_chunks` labeled by tier, incremented in `prefetch()` for each promotion actually initiated. This lets the run report distinguish "prefetch attempted" from "prefetch succeeded" (the latter already covered by `READ_BYTES` / `BLOCK_HITS`).

Also log a one-line summary per step when `prefetch_chunks > 0` so the model log shows the toy is active (the deployment checklist in [[Experiment Methodology]] requires a complete model log).

### Step 5 — Guardrails (keep the toy safe)

| Guardrail | How |
|---|---|
| Primary-tier pressure | `prefetch` skips any key where `prepare_write` returns `None`. No OOM risk beyond the existing eviction path. Monitor `PROMOTION_ALLOCATION_FAILURES` and `PRIMARY_WRITE_USAGE_PERC`. |
| No wasted promotion on absent blocks | `_try_promote` only promotes blocks that are secondary-`HIT`; blocks in no tier are skipped. |
| No double promotion | `HIT_PENDING` short-circuits without allocating. |
| Honors per-request tier filter | `_try_promote` checks `load_tier_filter.allows(...)`. |
| Bounded N | Validate `prefetch_chunks` ≤ a sane cap (e.g. 64) in `spec.py` to avoid a single miss triggering a multi-hundred-chunk burst. |
| Reversible | `N = 0` is the baseline; the whole feature is gated on `prefetch_chunks > 0`. |

## 4. Telemetry and measurement

Reuse the existing tiering telemetry — no new instrumentation is required to evaluate the toy, beyond the Step 4 counter.

### 4.1 Primary signal (does latency improve?)

From the Nemotron report's metric set, compare across N:

| Metric | Source | Expectation if prefetch helps |
|---|---|---|
| Request latency P50/P90 | AIPerf profile | lower as N grows, then rises when N overshoots |
| TTFT P50 | AIPerf profile | lower (fewer deferred prefill steps) |
| Tiering lookup P50/P90/P99 | `kv_offload_tiering_lookup_async_delay_seconds` | **fewer lookup events per request**; per-event delay unchanged (same backend) |
| Blocked requests (avg concurrent) | tiering telemetry | lower (requests spend fewer steps stalled) |
| External-token share | prompt-token-source counters | unchanged or slightly higher (more secondary hits served) |

The decisive comparison is **lookup-event count per request**, not per-event latency: the toy reduces the *number* of deferred steps, not the speed of each secondary read. Add a derived "lookup events per completed request" = `BLOCK_QUERIES / completed_requests` and plot vs N.

### 4.2 Negative-signal metrics (is prefetch hurting?)

| Metric | Why it matters |
|---|---|
| `PROMOTION_ALLOCATION_FAILURES` | rising ⇒ prefetch is crowding out demand promotions |
| `PRIMARY_WRITE_USAGE_PERC` | sustained near 100% ⇒ primary tier saturated by read-ahead |
| Total throughput P50 | dropping ⇒ prefetch overhead exceeds its benefit (N too large) |
| Recompute share | rising ⇒ prefetched blocks evicted demand blocks (LRU pressure) |

### 4.3 The N sweep

Run the Phase 0 baseline workload (AgentX MVP, Nemotron, TP8/U0.80/C32 — see [[Experiment Methodology]]) with `prefetch_chunks ∈ {0, 1, 2, 4, 8, 16}`. `0` is the within-batch control. Minimum 3 paired repetitions per cell; report mean ± CI and paired-request tail analysis, per the experiment definition's cross-phase measurement commitments.

Plot the latency and lookup-event-count curves against N. The expected shape is a U-curve: too-small N leaves the reactive cost in place; too-large N spends primary-tier capacity and transfer bandwidth on blocks that may be evicted before use. The Phase 1 exit criterion is a **repeatable, attributable** latency improvement at some N > 0 over the N = 0 control.

## 5. Run plan and exit criteria

This maps directly to Phase 1 of [[01 - Experiment Definition]].

1. **Implement** Steps 1–4 on a vLLM fork/branch; keep `N = 0` path identical to `main`.
2. **Unit-test** `_try_promote` and `prefetch` with a mock primary tier and a mock `SecondaryTierManager`: assert no double-promotion, primary-full tolerance, filter honored, `N = 0` no-op.
3. **Deploy** the branch image to the PSAP cluster; record the full image digest (per [[Experiment Methodology]]).
4. **Sweep** N ∈ {0, 1, 2, 4, 8, 16}, 3 repetitions each, same batch, same node class.
5. **Report** as a dated note under `Research/ABC/`: latency curves, lookup-event-count-per-request vs N, negative-signal metrics, and a **go/no-go decision**.
6. **Exit gate** (from the experiment definition): a measurable, repeatable latency change for at least three values of N, reported as mean ± CI with paired-request analysis, plus a recorded decision on whether to proceed to Phase 2.

## 6. Risks and unknowns

- **Primary-tier eviction churn.** Read-ahead fills the CPU tier with blocks the request *will* need soon, but under concurrency 32 several requests' read-aheads compete for the same 64 GiB CPU tier. If prefetched blocks evict each other (or evict demand blocks), latency regresses. The `PRIMARY_WRITE_USAGE_PERC` and recompute-share metrics catch this; the U-curve's right side quantifies it.
- **Interaction with `offload_prompt_only`.** `OffloadingSpec.offload_prompt_only` defaults to `True` (decode blocks are not offloaded). Read-ahead only touches prompt-prefix chunks, so this is consistent — but confirm the prefetched keys are all prompt chunks during the run.
- **Censored P99.** The Nemotron report notes overflow buckets are unavailable, so P99 at 10 s is a lower bound. Use P50/P90 and lookup-event-count as the primary signals; treat P99 as directional only until overflow is exported (a Phase 0 telemetry-gap item).
- **Multi-group models.** `offload_keys` are per KV group; `prefetch` is called within one group's scan. For multi-group models the toy is applied per full-attention group independently. This is fine for Phase 1 but must be revisited if groups have very different chunk sizes.

## 7. Out of scope for Phase 1

Explicitly **not** in this toy (deferred to later phases per [[01 - Experiment Definition]]):

- Any prediction model (XGBoost or otherwise) — Phase 3.
- The cost-benefit migration gate $\text{Benefit} > N \times \text{Cost}$ — Phase 3.
- Access-frequency / recency feature collection — Phase 2 (the toy uses only sequence order, not access history).
- Session-aware prefetching on conversation migration — Phase 3.
- Sliding-window-group read-ahead — Phase 2.
- Temperature export to the llm-d endpoint picker — Phase 3.

## 8. Related

- [[01 - Experiment Definition]] — Phase 1 objective, method, and exit criteria.
- [[2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]] — Phase 0 baseline data and the lookup-delay numbers this toy targets.
- [[Experiment Methodology]] — run structure, acceptance gates, repetition requirements.
- [[Activity-Based KV Cache Offloading]] — concept note; the implementation-placement verdict (prediction + placement live in core vLLM `vllm/v1/kv_offload`; this guide follows that verdict).
- [[00 - Index]] — ABC project index.
