---
title: "Phase 1 — Naive Proactive Prefetch Implementation Guide"
date: "2026-08-11"
type: implementation-guide
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
phase: "1 — naive proactive prefetching (toy)"
status: "draft"
depends-on: "[[Methodology/01 - Experiment Definition]] (Phase 1)"
baseline: "[[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]]"
workload: "semianalysisai/cc-traces-weka-061526"
codebase: "vllm-project/vllm @ main (inspected 2026-08-11)"
---

# Phase 1 — Naive Proactive Prefetch of N Blocks

This guide explains, step by step, how to implement the first proactive-fetching step of the ABC project: a toy prefetcher that proactively promotes **N** KV cache chunks from a secondary tier (NVMe / CephFS) to the CPU primary tier before they are requested, and measures whether request-visible latency improves.

It is the implementation counterpart of **Phase 1** in [[Methodology/01 - Experiment Definition]]. It is deliberately a toy: no prediction model, no cost-benefit gate, no session awareness. The only knob is **N**, the number of chunks to read-ahead.

> **Code references.** All file paths, class names, and method names below are verified against `vllm-project/vllm` at `main`, inspected 2026-08-11 via the GitHub connector. Code blocks are illustrative sketches, not copy-paste patches — line numbers drift; re-read each file before editing.

## 1. How fetching works today (the reactive baseline)

Understanding the reactive path is the whole prerequisite, because the toy reuses its machinery. The path spans three files:


| Layer                            | File                                                                   | Class / function               |
| -------------------------------- | ---------------------------------------------------------------------- | ------------------------------ |
| Connector (scheduler side)       | `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py` | `OffloadingConnectorScheduler` |
| Tiering manager (scheduler side) | `vllm/v1/kv_offload/tiering/manager.py`                                | `TieringOffloadingManager`     |
| Secondary tier interface         | `vllm/v1/kv_offload/tiering/base.py`                                   | `SecondaryTierManager`         |


### 1.1 The per-request sequential scan

When the scheduler considers a request, it calls `OffloadingConnectorScheduler.get_num_new_matched_tokens()` (`vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`). That builds the request's full list of `offload_keys` from `req.block_hashes` (`RequestOffloadState.update_offload_keys`) and then calls `_lookup()` → `_lookup_complete_chunks()` → `_maximal_prefix_lookup()`.

`_maximal_prefix_lookup` (`vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`) is the heart of the reactive behavior:

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

`TieringOffloadingManager.lookup()` (`vllm/v1/kv_offload/tiering/manager.py`) checks the CPU primary tier first. On a primary MISS it walks the secondary tiers; on the first secondary HIT it calls `_initiate_promotion()`:

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

At the end of each scheduler step, `build_connector_meta()` (`vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`) calls `manager.on_schedule_end()` (`vllm/v1/kv_offload/tiering/manager.py`), which calls `_flush_pending_promotions()`:

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

Putting 1.1–1.3 together: **only the one chunk that caused the MISS is promoted per step.** A request that needs K chunks from a secondary tier is deferred ~K times, and each deferral pays the secondary-tier lookup delay. The Nemotron baseline report ([[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]]) measured CephFS/NVMe P50 lookup delays of ~2.2 s and P99 hitting the 10 s histogram ceiling. That multi-second, per-chunk, sequential cost is exactly what this toy targets.

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

`TieringOffloadingSpec` reads its behavior from `kv_connector_extra_config` (see the docstring at the top of `vllm/v1/kv_offload/tiering/spec.py`). Add a new key:

- `prefetch_chunks` (int, default `0`): number of additional chunks to proactively promote on the first secondary-tier miss. `0` reproduces the reactive baseline exactly — this is the control value for the sweep.

Plumb it through:

1. Read it in `TieringOffloadingSpec.__init__` from `self.extra_config.get("prefetch_chunks", 0)`, validate it is a non-negative int, and store as `self.prefetch_chunks`.
2. Pass it into the `TieringOffloadingManager` constructor (`vllm/v1/kv_offload/tiering/spec.py`, `get_manager()`), and store as `self._prefetch_chunks` on the manager.
3. Expose it on the manager as a read-only property so the connector scheduler can read it without a second plumbing path: `manager.prefetch_chunks`.

> **Why default 0:** the experiment is a comparison against the Phase 0 baseline. `N = 0` must be byte-for-byte the existing behavior so the sweep has a clean control. Confirm this with a paired `N = 0` run in the batch.

### Step 2 — Add a `_try_promote` helper and a `prefetch` method on the manager

**File:** `vllm/v1/kv_offload/tiering/manager.py`, on `class TieringOffloadingManager(OffloadingManager)`.

Add a new `_try_promote` helper and a batched `prefetch()` method. These are **separate from `lookup()**` — `lookup()` stays unchanged. The two methods share the promotion primitive (`_initiate_promotion`) and the tier filter check, but have intentionally different semantics (see "Why `lookup()` stays unchanged" below).

```python
# EXISTING — unchanged. Demand path. Full four-way semantics + metrics + poll.
def lookup(self, key, req_context, *, exclude_tier_idx=None) -> LookupResult:
    self._maybe_process_finished_jobs()
    primary_hit = self.primary_tier.lookup(key, req_context)
    self._metrics.on_lookup(...)                          # metrics
    if primary_hit is LookupResult.HIT: return LookupResult.HIT
    if primary_hit is LookupResult.HIT_PENDING: return LookupResult.HIT_PENDING
    any_retry = False
    for i, tier in enumerate(self.secondary_tiers):
        ...
        result = tier.lookup(key, req_context)
        self._metrics.on_lookup(...)                      # metrics
        if result is LookupResult.HIT:
            promoted = self._initiate_promotion(i, key, req_context)
            return LookupResult.MISS if not promoted else LookupResult.RETRY
        if result is LookupResult.RETRY: any_retry = True   # retry tracking
    return LookupResult.RETRY if any_retry else LookupResult.MISS

# NEW — proactive path. Bool semantics only. Called by prefetch().
def _try_promote(self, key, req_context, exclude_tier_idx=None) -> bool:
    """Return True if a promotion was initiated (or already in-flight/hit).

    Shares _initiate_promotion and the tier filter with lookup(), but omits
    metrics, job polling, and any_retry tracking — see the note below.
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

# NEW — batched wrapper, called by the connector scheduler.
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

#### Why `lookup()` stays unchanged

`lookup()` does three things `_try_promote` deliberately omits:

1. **Polls finished jobs first** (`self._maybe_process_finished_jobs()`): so completed promotions are reflected as `HIT` before the scan. The prefetch path doesn't need this — the poll already ran in the `lookup()` call that triggered the prefetch, earlier in the same step.
2. **Records per-tier metrics** (`self._metrics.on_lookup(...)`): for primary, for each secondary `HIT`, and for each secondary non-hit. The prefetch path intentionally omits these so prefetched blocks' secondary lookups are **not counted** in `BLOCK_QUERIES` / `BLOCK_HITS` — this keeps the "lookup events per request" signal (Section 4.1) measuring only demand-driven lookups, so the sweep shows the demand-lookup count dropping as N grows. If prefetch lookups were also counted, the metric would conflate "fewer demand lookups" with "more prefetch lookups" and the U-curve signal would be muddied.
3. **Tracks `any_retry` across all secondary tiers**: if tier 0 returns `RETRY` and tier 1 returns `MISS`, `lookup()` returns `RETRY` — the block *might* be in tier 0 once its async lookup resolves. `_try_promote` just returns `False` on the first non-hit. This is fine for prefetch: it doesn't need to defer anything, doesn't feed a scheduling decision, and a block that's not yet confirmed in any tier is simply skipped (it may be promoted on a later step's prefetch or by the demand path).

The two methods share `_initiate_promotion()` (the actual promotion primitive — single implementation) and the `load_tier_filter.allows(...)` check, so the promotion behavior cannot drift. The differences above are intentional, not drift.

#### Other notes

- The helper **respects `load_tier_filter**` exactly as `lookup()` does, so per-request tier restrictions (e.g. `kv_load_tiers` in `kv_transfer_params`) are honored by prefetch too.
- It **tolerates primary-full**: `_initiate_promotion` returns `False` when `primary_tier.prepare_write()` returns `None`; `prefetch` just skips that key. No exception, no deadlock.
- It **does not double-promote**: a block already in-flight (`HIT_PENDING`, `ref_cnt = -1`) returns `True` without calling `prepare_write` again, so it never allocates a second primary slot for the same key.
- Because `_try_promote` does not record metrics, the `prefetch_chunks` counter (Step 4) is the **only** record of prefetch activity — it is load-bearing for the analysis, not just nice-to-have.

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

> **Sliding-window groups:** `_sliding_window_lookup` (`vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`) scans from the end and has different semantics. Do **not** add the prefetch hook there in Phase 1 — restrict the toy to full-attention groups (`_maximal_prefix_lookup`). Sliding-window read-ahead needs its own analysis and belongs in Phase 2.

### Step 4 — Add prefetch counter metrics

This step adds the **only** record of prefetch activity, since `_try_promote` deliberately does not call `self._metrics.on_lookup()` (see Step 2's "Why `lookup()` stays unchanged"). Without these counters, the sweep cannot distinguish "prefetch is working but not helping" from "prefetch is not firing at all." Three files are touched, each mirroring an existing pattern.

#### 4a — Register the metric names

**File:** `vllm/v1/kv_offload/tiering/base.py`, in `class TieringOffloadingMetrics`.

Add three counter names next to the existing `PROMOTION_ALLOCATION_FAILURES`:

```python
class TieringOffloadingMetrics:
    # ... existing names ...
    PROMOTION_ALLOCATION_FAILURES = (
        "vllm:kv_offload_tiering_promotion_allocation_failures"
    )
    # NEW — Phase 1 prefetch process counters
    PREFETCH_ATTEMPTED = "vllm:kv_offload_tiering_prefetch_attempted"
    PREFETCH_PROMOTED = "vllm:kv_offload_tiering_prefetch_promoted"
    PREFETCH_SKIPPED = "vllm:kv_offload_tiering_prefetch_skipped"
    # NEW — Phase 1 prefetch outcome counters
    PREFETCH_USEFUL = "vllm:kv_offload_tiering_prefetch_useful"
    PREFETCH_WASTED = "vllm:kv_offload_tiering_prefetch_wasted"
```

Five counters, split into **process** (was the prefetch mechanism firing?) and **outcome** (did the GPU actually use the prefetched blocks?):

| Counter | Type | When incremented | Answers |
|---|---|---|---|
| `PREFETCH_ATTEMPTED` | process | once per key passed to `prefetch()` | "how many blocks did we try to read-ahead?" |
| `PREFETCH_PROMOTED` | process | per key where `_try_promote` started a promotion (secondary `HIT` + `_initiate_promotion` succeeded) | "how many read-aheads actually moved data?" |
| `PREFETCH_SKIPPED` | process | per key where `_try_promote` returned `False` (not in any secondary tier, or primary full) | "how much read-ahead was wasted at the promotion stage?" |
| `PREFETCH_USEFUL` | outcome | a demand `lookup()` returns `HIT` for a key that was prefetched | "did the GPU actually use a prefetched block?" |
| `PREFETCH_WASTED` | outcome | a prefetched block was evicted before use (demand `lookup()` returns `MISS`), or the request finished without the demand scan ever reaching it | "was the prefetch wasted after promotion?" |

The process counters diagnose the promotion stage; the outcome counters diagnose whether the promotion helped. The full measurement chain is:

```
PREFETCH_ATTEMPTED
  ├── PREFETCH_SKIPPED     (not in any secondary tier, or primary full)
  └── PREFETCH_PROMOTED    (promotion initiated — the "total prefetched")
        ├── PREFETCH_USEFUL  (demand lookup HIT on a prefetched block)
        └── PREFETCH_WASTED  (evicted before use, or never reached)
```

The prefetch hit rate is computed in PromQL, not stored as a metric:

```
prefetch_hit_rate = PREFETCH_USEFUL / (PREFETCH_USEFUL + PREFETCH_WASTED)
```

where `PREFETCH_USEFUL + PREFETCH_WASTED = PREFETCH_PROMOTED` (every successfully promoted block is eventually either demand-hit or wasted). The process split matters for diagnosing the U-curve's right side: if latency regresses at large N, `PREFETCH_SKIPPED` rising indicates primary-tier pressure (promotions failing on `prepare_write`), while `PREFETCH_PROMOTED` high + `PREFETCH_USEFUL` low indicates eviction churn (promotions succeeded but the blocks were evicted before use — a capacity problem, not a promotion problem).

#### 4b — Declare the metric definitions

**File:** `vllm/v1/kv_offload/tiering/spec.py`, in `TieringOffloadingSpec.build_metric_definitions()`.

Add the three counters to the `metrics` dict, labeled by tier, mirroring the existing `PROMOTION_ALLOCATION_FAILURES` declaration:

```python
metrics[TieringOffloadingMetrics.PREFETCH_ATTEMPTED] = OffloadingCounterMetadata(
    documentation=(
        "Number of KV cache chunks passed to prefetch() for proactive "
        "promotion, labeled by tier. Phase 1 toy read-ahead."
    ),
    labelnames=("tier",),
)
metrics[TieringOffloadingMetrics.PREFETCH_PROMOTED] = OffloadingCounterMetadata(
    documentation=(
        "Number of prefetch chunks that initiated a secondary->primary "
        "promotion, labeled by tier. Subset of PREFETCH_ATTEMPTED."
    ),
    labelnames=("tier",),
)
metrics[TieringOffloadingMetrics.PREFETCH_SKIPPED] = OffloadingCounterMetadata(
    documentation=(
        "Number of prefetch chunks skipped (not in any secondary tier, or "
        "primary tier full), labeled by tier. Subset of PREFETCH_ATTEMPTED."
    ),
    labelnames=("tier",),
)
metrics[TieringOffloadingMetrics.PREFETCH_USEFUL] = OffloadingCounterMetadata(
    documentation=(
        "Number of prefetched chunks that were subsequently demand-hit by "
        "the GPU (lookup() returned HIT on a prefetched block). Labeled by "
        "tier. Subset of PREFETCH_PROMOTED. The prefetch hit rate is "
        "PREFETCH_USEFUL / (PREFETCH_USEFUL + PREFETCH_WASTED)."
    ),
    labelnames=("tier",),
)
metrics[TieringOffloadingMetrics.PREFETCH_WASTED] = OffloadingCounterMetadata(
    documentation=(
        "Number of prefetched chunks that were evicted from the primary tier "
        "before the demand scan reached them, or that the request finished "
        "without ever reaching. Labeled by tier. Subset of "
        "PREFETCH_PROMOTED."
    ),
    labelnames=("tier",),
)
```

`build_metric_definitions()` runs at spec creation; it declares the Prometheus metric so the server exports it. The existing `PROMOTION_ALLOCATION_FAILURES` entry (unlabeled) is the closest pattern; the `READ_BYTES` / `BLOCK_HITS` entries (labeled by tier) show the `labelnames=("tier",)` form.

#### 4c — Record the counters in the tracker

**File:** `vllm/v1/kv_offload/tiering/metrics.py`, in `class TieringMetricsTracker`.

Add a method that `prefetch()` will call, mirroring the existing `on_promotion_allocation_failure()`:

```python
# existing pattern:
def on_promotion_allocation_failure(self) -> None:
    self._stats.increase_counter(
        TieringOffloadingMetrics.PROMOTION_ALLOCATION_FAILURES
    )

# NEW:
def on_prefetch_attempted(self, tier_label: TierLabel) -> None:
    self._stats.increase_counter(
        TieringOffloadingMetrics.PREFETCH_ATTEMPTED, labelvalues=tier_label
    )

def on_prefetch_promoted(self, tier_label: TierLabel) -> None:
    self._stats.increase_counter(
        TieringOffloadingMetrics.PREFETCH_PROMOTED, labelvalues=tier_label
    )

def on_prefetch_skipped(self, tier_label: TierLabel) -> None:
    self._stats.increase_counter(
        TieringOffloadingMetrics.PREFETCH_SKIPPED, labelvalues=tier_label
    )

def on_prefetch_useful(self, tier_label: TierLabel) -> None:
    self._stats.increase_counter(
        TieringOffloadingMetrics.PREFETCH_USEFUL, labelvalues=tier_label
    )

def on_prefetch_wasted(self, tier_label: TierLabel) -> None:
    self._stats.increase_counter(
        TieringOffloadingMetrics.PREFETCH_WASTED, labelvalues=tier_label
    )
```

The `tier_label` comes from `self._metrics.tier_label(i)` — the same helper already used in `lookup()` for the demand-path metrics. For prefetch, the label is the tier where the promotion was attempted (or the first tier scanned, for skipped blocks that were in no tier).

#### 4d — Call the tracker from `prefetch()` and track outcomes in `lookup()`

**File:** `vllm/v1/kv_offload/tiering/manager.py`.

This step has three parts: (1) record the process counters in `prefetch()`, (2) track which keys were prefetched so `lookup()` can detect outcomes, and (3) clean up at request finish. The outcome tracking (`PREFETCH_USEFUL` / `PREFETCH_WASTED`) is what tells you whether the prefetch actually helped — without it, the sweep can only see latency, not whether the prefetched blocks were used by the GPU.

**Part 1 — `__init__`: add the prefetched-key tracking set.**

```python
class TieringOffloadingManager(OffloadingManager):
    def __init__(self, primary_tier, secondary_tiers=None, prefetch_chunks: int = 0):
        ...
        self._prefetch_chunks = prefetch_chunks
        # Per-request set of keys that were prefetched (promoted via
        # prefetch(), not via the demand path). lookup() checks this to
        # detect whether a demand HIT landed on a prefetched block (useful)
        # or whether the block was evicted before the scan reached it (wasted).
        self._prefetched_keys: dict[str, set[OffloadKey]] = {}
        ...
```

**Part 2 — `_try_promote` and `prefetch()`: record process counters and register keys for outcome tracking.**

`_try_promote` must return *why* it returned `False` (not in any tier vs. primary full), so it returns `(promoted, tier_idx)` instead of a bare bool:

```python
def _try_promote(self, key, req_context, exclude_tier_idx=None):
    """Returns (promoted: bool, tier_idx: int | None).

    tier_idx is the secondary tier the promotion was attempted on (for
    metrics labeling), or None if the block was not in any tier.
    """
    primary_hit = self.primary_tier.lookup(key, req_context)
    if primary_hit in (LookupResult.HIT, LookupResult.HIT_PENDING):
        return True, None                      # already primary-resident (not counted)
    for i, tier in enumerate(self.secondary_tiers):
        if i == exclude_tier_idx:
            continue
        if not req_context.load_tier_filter.allows(tier.medium, tier.locality):
            continue
        result = tier.lookup(key, req_context)
        if result is LookupResult.HIT:
            promoted = self._initiate_promotion(i, key, req_context)
            return promoted, i                 # promoted=True/False, tier=i
    return False, None                         # not in any tier

def prefetch(self, keys, req_context) -> int:
    initiated = 0
    for key in keys:
        promoted, tier_idx = self._try_promote(key, req_context)

        # Every key passed to prefetch() is an attempt, labeled with the
        # aggregate prefetch label (we don't know the tier until _try_promote
        # returns, and absent blocks have no tier at all).
        self._metrics.on_prefetch_attempted(PREFETCH_TIER_LABEL)

        if promoted:
            initiated += 1
            # Promotion succeeded on a specific secondary tier.
            self._metrics.on_prefetch_promoted(self._metrics.tier_label(tier_idx))
            # Register the key for outcome tracking: lookup() will check
            # this set to detect useful (demand HIT) vs wasted (evicted).
            self._prefetched_keys.setdefault(req_context.req_id, set()).add(key)
        elif tier_idx is not None:
            # Block was in a secondary tier but primary was full.
            self._metrics.on_prefetch_skipped(self._metrics.tier_label(tier_idx))
        else:
            # Block was not in any secondary tier.
            self._metrics.on_prefetch_skipped(PREFETCH_TIER_LABEL)
    return initiated
```

**Part 3 — `lookup()`: detect useful vs wasted prefetches.**

Add a check right after the primary-tier lookup, **before** the existing `HIT` / `HIT_PENDING` returns. This is the only modification to `lookup()` — the existing metrics, polling, and secondary-scan logic stay unchanged:

```python
def lookup(self, key, req_context, *, exclude_tier_idx=None) -> LookupResult:
    self._maybe_process_finished_jobs()

    start_time = time.monotonic()
    primary_hit = self.primary_tier.lookup(key, req_context)
    lookup_duration = time.monotonic() - start_time
    self._metrics.on_lookup(
        req_context, key, self._metrics.primary_tier_label,
        primary_hit, lookup_duration,
    )

    # ---- NEW: prefetch outcome tracking ----
    prefetched = self._prefetched_keys.get(req_context.req_id)
    if prefetched and key in prefetched:
        if primary_hit is LookupResult.HIT:
            # Demand scan hit a prefetched block — the prefetch was useful.
            prefetched.discard(key)
            self._metrics.on_prefetch_useful(PREFETCH_TIER_LABEL)
        elif primary_hit is LookupResult.MISS:
            # Block was prefetched but evicted from primary before the
            # demand scan reached it — wasted.
            prefetched.discard(key)
            self._metrics.on_prefetch_wasted(PREFETCH_TIER_LABEL)
        # HIT_PENDING: promotion still in-flight — leave in set, count
        # on a later step when it resolves to HIT or MISS.
    # ---- end prefetch outcome tracking ----

    if primary_hit is LookupResult.HIT:
        return LookupResult.HIT
    if primary_hit is LookupResult.HIT_PENDING:
        return LookupResult.HIT_PENDING
    # ... existing secondary-scan logic unchanged ...
```

The three primary-tier verdicts map cleanly to the three prefetch outcomes:

| Primary `lookup()` returns | For a prefetched key, this means | Counted as |
|---|---|---|
| `HIT` | Block was prefetched, promotion completed, demand scan found it ready | **USEFUL** |
| `HIT_PENDING` | Block was prefetched, promotion still in-flight | not counted yet (leave in set) |
| `MISS` | Block was prefetched but evicted from primary before the demand scan reached it | **WASTED** |

A block with `ref_cnt = -1` (in-flight) cannot be evicted — the negative ref_cnt protects it. So a prefetched block can only become `MISS` after it was promoted and then evicted by the LRU/ARC policy. That's exactly the "wasted by eviction" case.

**Part 4 — `on_request_finished()`: clean up unreached prefetched keys.**

Keys still in the set when the request finishes were never demand-hit — the request completed before the scan reached them. Count them as wasted and clear the set:

```python
@override
def on_request_finished(self, req_context, *, exclude_tier_idx=None):
    # Keys still tracked as prefetched were never demand-hit — the request
    # finished before the scan reached them. Count as wasted.
    remaining = self._prefetched_keys.pop(req_context.req_id, set())
    for _ in remaining:
        self._metrics.on_prefetch_wasted(PREFETCH_TIER_LABEL)

    self.primary_tier.on_request_finished(req_context)
    state = self._req_state[req_context.req_id]
    state.is_finished = True
    self._maybe_finalize_request(req_context.req_id, exclude_tier_idx)
```

**Memory and overhead.** `_prefetched_keys` holds at most N keys per active request. With 32 concurrent requests and N up to 256, that's at most 8,192 `OffloadKey`s (each ~36 bytes) — ~288 KB. The `lookup()` check is a `dict.get()` + `set.__contains__()` — O(1), negligible vs the primary-tier lookup that already happened.

**Cross-request sharing caveat.** Blocks are content-addressed, so two requests sharing a prefix share `OffloadKey`s. If request A prefetches block X and request B's demand scan hits it, we'd miss the useful count (we check B's set, not A's). For Phase 1, this undercounts the hit rate slightly — acceptable, since the relative comparison across N values is still valid. A global set (not per-request) would fix this but complicates cleanup. Phase 2 can revisit if needed.

`PREFETCH_TIER_LABEL` is a constant defined in `vllm/v1/kv_offload/tiering/metrics.py` next to the existing `PRIMARY_TIER_LABEL`, and imported into `manager.py`:

```python
# vllm/v1/kv_offload/tiering/metrics.py
PRIMARY_TIER_LABEL: TierLabel = ("0:primary",)
PREFETCH_TIER_LABEL: TierLabel = ("prefetch",)   # NEW

# vllm/v1/kv_offload/tiering/manager.py (top of file, extending existing import)
from vllm.v1.kv_offload.tiering.metrics import (
    PREFETCH_TIER_LABEL,
    TieringMetricsTracker,
)
```

**Labeling rationale:** `PREFETCH_ATTEMPTED` and the "not-in-any-tier" skip are labeled with the aggregate `PREFETCH_TIER_LABEL` (the block was scanned across all secondary tiers, so there is no single tier to attribute the attempt to). `PREFETCH_PROMOTED` and the "primary-full" skip are labeled with the actual `tier_label(i)` — the specific tier where the promotion was attempted. This keeps the per-tier counters (`PROMOTED`, primary-full `SKIPPED`) meaningful for per-tier diagnosis while the attempt count and absent-block skip are aggregate. The invariant `ATTEMPTED = PROMOTED + SKIPPED{tier} + SKIPPED{prefetch}` holds, so no counts are lost.

#### 4e — Log at startup that prefetch is active

**File:** `vllm/v1/kv_offload/tiering/manager.py`, in `TieringOffloadingManager.__init__`.

A single `logger.info` at construction confirms the knob took effect, without the per-step accumulators and debug log that would otherwise be needed. The per-step measurement comes from the Prometheus counters (4a–4d), which is where time-series data belongs for a benchmarking sweep:

```python
class TieringOffloadingManager(OffloadingManager):
    def __init__(self, primary_tier, secondary_tiers=None, prefetch_chunks: int = 0):
        ...
        self._prefetch_chunks = prefetch_chunks
        if self._prefetch_chunks > 0:
            logger.info(
                "Phase 1 toy prefetch enabled: prefetch_chunks=%d",
                self._prefetch_chunks,
            )
```

No per-step logging, no standalone accumulators on the manager. The startup line is sufficient to confirm the feature is active in the model log (the deployment checklist in [[Experiment Methodology]] requires a complete log; this satisfies "is prefetch on?" without per-step spam at 32-way concurrency). If prefetch stops firing mid-run (e.g., after a cache reset), that's visible in Prometheus as the counter rate dropping to zero — which is the right place to look for time-series behavior, not the log.

#### Why five counters, not one

The single-counter version ("increment `PREFETCH_CHUNKS` per promotion") cannot distinguish the failure modes that the U-curve's two sides must diagnose:

- **Process failures** (`PREFETCH_SKIPPED`): the block *was* in secondary but the CPU tier had no room (primary-full skip, labeled with a real tier label), or the block was never offloaded (absent-block skip, labeled with `PREFETCH_TIER_LABEL`). Rising primary-full skips + rising `PROMOTION_ALLOCATION_FAILURES` = the CPU tier is the bottleneck.
- **Outcome failures** (`PREFETCH_WASTED`): the block *was* promoted but evicted before the demand scan reached it, or the request finished without ever reaching it. High `PREFETCH_PROMOTED` + low `PREFETCH_USEFUL` = eviction churn (the right side of the U-curve). This is the signal that distinguishes "prefetch is moving data that gets thrown away" from "prefetch is moving data that gets used."

Without the process/outcome split, a rising total is ambiguous: is the toy wasting work on absent blocks (fine), choking on primary pressure (back off N), or promoting blocks that get evicted (back off N for a different reason)? The five-counter split resolves all three.

#### Relationship to existing metrics


| Existing metric                 | Counts prefetch activity? | Why                                                                                                                                                                                                                                         |
| ------------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `BLOCK_QUERIES` / `BLOCK_HITS`  | **No**                    | `_try_promote` doesn't call `on_lookup()`, so prefetch lookups are excluded. This is deliberate (Step 2) — keeps the demand-lookup signal clean.                                                                                            |
| `READ_BYTES` / `READ_TIME`      | **Yes**                   | Prefetch promotions complete via the same `_complete_promotion` path as demand promotions, so their bytes/time are counted. This means `READ_BYTES` rises with N (more data moved), which is expected and correct.                          |
| `PROMOTION_ALLOCATION_FAILURES` | **Yes**                   | `_initiate_promotion` calls `on_promotion_allocation_failure()` on primary-full, for both demand and prefetch paths. So this metric rises with N when the CPU tier is pressured — use it alongside `PREFETCH_SKIPPED` to confirm the cause. |
| `ACTIVE_PROMOTION_JOBS`         | **Yes**                   | Prefetch promotions register jobs via `_register_job`, so they appear in the active-job gauge. Expect this to rise with N.                                                                                                                  |


So the prefetch-specific counters (`PREFETCH_ATTEMPTED` / `PROMOTED` / `SKIPPED` / `USEFUL` / `WASTED`) are the **only** way to isolate prefetch activity from demand activity. The existing metrics either exclude it (`BLOCK_QUERIES`) or conflate it with demand (`READ_BYTES`, `PROMOTION_ALLOCATION_FAILURES`). The process counters (`ATTEMPTED` / `PROMOTED` / `SKIPPED`) tell you if the mechanism is firing; the outcome counters (`USEFUL` / `WASTED`) tell you if it's helping. This is why Step 4 is load-bearing for the analysis, not just nice-to-have.

### Step 5 — Guardrails (keep the toy safe)


| Guardrail                            | How                                                                                                                                                                                                                                                   |
| ------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Primary-tier pressure                | `prefetch` skips any key where `prepare_write` returns `None`. No OOM risk beyond the existing eviction path. Monitor `PROMOTION_ALLOCATION_FAILURES` and `PRIMARY_WRITE_USAGE_PERC`.                                                                 |
| No wasted promotion on absent blocks | `_try_promote` only promotes blocks that are secondary-`HIT`; blocks in no tier are skipped.                                                                                                                                                          |
| No double promotion                  | `HIT_PENDING` short-circuits without allocating.                                                                                                                                                                                                      |
| Honors per-request tier filter       | `_try_promote` checks `load_tier_filter.allows(...)`.                                                                                                                                                                                                 |
| Bounded N                            | Validate `prefetch_chunks` ≤ a sane cap (e.g. 256) in `vllm/v1/kv_offload/tiering/spec.py` to avoid a single miss triggering a multi-thousand-chunk burst. The sweep tops out at 256; the cap should not be the binding constraint in the experiment. |
| Reversible                           | `N = 0` is the baseline; the whole feature is gated on `prefetch_chunks > 0`.                                                                                                                                                                         |

---

Continue with [[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide/02 - Telemetry, run plan, and risks|Part 2 — telemetry, run plan, and risks]].