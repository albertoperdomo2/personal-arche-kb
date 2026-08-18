---
title: "Phase 2 — Heuristic Prefetch Implementation Guide (tentative)"
date: "2026-08-13"
type: implementation-guide
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
phase: "2 — heuristic prefetching"
status: "tentative"
depends-on: "[[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide]] (Phase 1 results)"
workload: "semianalysisai/cc-traces-weka-061526"
codebase: "vllm-project/vllm @ main (inspected 2026-08-11)"
---

# Phase 2 — Heuristic Prefetch (tentative)

This guide outlines the implementation of Phase 2 of the ABC project: replacing the Phase 1 toy's static-N spatial read-ahead with a heuristic prefetcher that adapts how many blocks to prefetch and selects *which* blocks to prefetch based on activity features. It is **tentative** — the concrete thresholds, feature weights, and selection rules must be calibrated against Phase 1's sweep results, which are not yet available. The design and implementation steps are actionable, but the numbers are placeholders.

> **Status convention.** Sections marked **[depends on Phase 1]** cannot be finalized until the Phase 1 sweep produces the U-curve, the measured K, and the prefetch hit rate. Sections marked **[design-complete]** are independent of Phase 1 results.

## 1. What Phase 1 established

Phase 1 ([[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide]]) built and instrumented a toy prefetcher with these properties:

- **Static N** read-ahead: on the first demand `MISS` in `_maximal_prefix_lookup`, proactively promote the next N chunks from secondary to primary.
- **Five Prometheus counters**: `PREFETCH_ATTEMPTED` / `PROMOTED` / `SKIPPED` (process) and `PREFETCH_USEFUL` / `WASTED` (outcome), with a global `_prefetched_keys` set for cross-turn hit-rate tracking.
- **Cross-turn benefit model**: the prefetched blocks help the *next turn's* scan, not the current turn's. The current turn recomputes the MISS block and everything after; the prefetched blocks survive in primary and are found as `HIT` by the next turn's scan — avoiding a deferred step.
- **U-curve hypothesis**: latency as a function of N has a sweet spot where read-ahead amortizes the secondary fetch, with regression at large N due to CPU-tier eviction churn.

Phase 2 builds on all four. The key Phase 1 deliverables that feed Phase 2:

| Phase 1 deliverable | How Phase 2 uses it |
|---|---|
| Sweet-spot N from the U-curve | Starting value for the adaptive N controller |
| Prefetch hit rate at the sweet spot | Baseline the heuristic must beat |
| `PREFETCH_WASTED` breakdown (eviction vs. never-reached) | Tells whether the heuristic should focus on selection (eviction) or timing (never-reached) |
| Measured K (secondary-fetch length per request) | Calibrates the adaptive N's range and the feature thresholds |
| Cross-turn benefit confirmation | The heuristic must predict *next-turn* demand, not current-turn |

## 2. Phase 2 design overview

Phase 2 replaces the Phase 1 toy's two fixed choices — **how many** blocks to prefetch (static N) and **which** blocks to prefetch (next N in sequence) — with adaptive, feature-based versions. Three components, each independently deployable and measurable:

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Phase 2 components and their Phase 1 evolution",
  "width": 720,
  "height": 200,
  "background": "white",
  "data": {
    "values": [
      {"y": 2, "label": "Static N (fixed)", "component": "Phase 1"},
      {"y": 1, "label": "Adaptive N (hit-rate feedback)", "component": "Phase 2"},
      {"y": 2, "label": "Spatial read-ahead (next N in sequence)", "component": "Phase 1"},
      {"y": 1, "label": "Feature-weighted selection (top N by score)", "component": "Phase 2"},
      {"y": 2, "label": "Full-attention groups only", "component": "Phase 1"},
      {"y": 1, "label": "Full-attention + sliding-window groups", "component": "Phase 2"}
    ]
  },
  "layer": [
    {
      "mark": {"type": "line", "point": true, "strokeWidth": 2},
      "encoding": {
        "x": {"field": "y", "type": "ordinal", "title": null, "axis": null, "scale": {"domain": [1, 2]}},
        "y": {"field": "label", "type": "nominal", "title": null, "axis": {"labelAngle": 0, "labelLimit": 300}},
        "color": {"field": "component", "type": "nominal", "legend": {"title": "Phase"},
                  "scale": {"range": ["#1f77b4", "#d62728"]}}
      }
    }
  ]
}
```

| Component | Phase 1 (toy) | Phase 2 (heuristic) | Complexity |
|---|---|---|---|
| **How many** (N) | static, swept manually | adaptive, driven by hit-rate feedback | low — feedback loop on existing counters |
| **Which** (selection) | next N in sequence | top N by feature-weighted score | medium — feature collection + scoring |
| **Where** (coverage) | full-attention groups only | + sliding-window groups | low — extend the hook to `_sliding_window_lookup` |

The components are ordered by implementation cost. Adaptive N is the cheapest evolution and can be deployed first; feature-based selection is the core of Phase 2 and the main investment; sliding-window support is a coverage extension that can be added once the heuristic is validated on full-attention groups.

## 3. The cross-turn benefit model — a design constraint

Phase 1 revealed that the prefetch benefit is **cross-turn, not within-turn**. The current turn always recomputes the MISS block and everything after it; the prefetched blocks help the *next* turn's scan find them already in primary. This has three design implications for Phase 2:

### 3.1 The heuristic must predict next-turn demand

The feature score should estimate "will this block be in the next turn's prefix and need promotion from secondary?" — not "will this block be accessed soon" in the abstract. The features that predict this are:

- **Recency-weighted frequency**: blocks accessed in recent turns are likely in the same session and will be in the next turn's prefix.
- **Neighboring block activity**: if blocks `ki-2`, `ki-1` are hot (recently accessed), block `ki` is likely hot too — the prefix is contiguous.
- **Session reuse ratio**: if the session has high prefix reuse (~93% in the Weka workload), the next turn's prefix overlaps heavily with the current turn's, so the prefetched blocks are likely to be needed.

### 3.2 The store path is the prefetch's competitor

The current turn's recompute path stores blocks `ki..kM` to primary (via `prepare_store` → `complete_store`). These stores compete with the prefetched blocks for CPU-tier residency. If the store path evicts the prefetched blocks before the next turn arrives, the prefetch is wasted (`PREFETCH_WASTED`).

Phase 2 should monitor this conflict but not solve it with a cost-benefit model (that's Phase 3). The heuristic can account for it by:

- **Prefetching blocks that are NOT in the current turn's recompute range.** The current turn recomputes from the MISS block onward; the prefetched blocks are *after* the MISS, so they overlap with the recompute range. But the recompute stores blocks to primary anyway — so the prefetched blocks may be redundant (the store path will put them in primary for free). The heuristic should skip blocks that the store path will naturally reach.
- **[depends on Phase 1]** If Phase 1's `PREFETCH_WASTED` is dominated by eviction (not never-reached), the store-path conflict is the primary failure mode, and the heuristic should prioritize blocks outside the recompute range or delay prefetching until the store path completes.

### 3.3 The benefit is delayed by one turn

The latency improvement from a prefetch on turn N is realized on turn N+1. The U-curve measures aggregate latency, so the benefit is visible — but it's smeared across turns. Phase 2's evaluation should pair turns (N, N+1) and measure the N+1 latency improvement attributable to N's prefetch, not just aggregate latency.

## 4. Component 1 — Adaptive N controller [design-complete]

### 4.1 The feedback loop

The adaptive N controller replaces the static `prefetch_chunks` config knob with a runtime-adjusted value driven by the prefetch hit rate. The three cases from the experiment design:

| Hit rate (sliding window) | Action | Rationale |
|---|---|---|
| > `high_threshold` (e.g. 80%) | increase N: `N = min(N * 2, N_max)` | prefetch is useful, read more |
| < `low_threshold` (e.g. 20%) | decrease N: `N = max(N // 2, 0)` | prefetch is wasted, read less |
| between | hold N | prefetch is marginal, don't change |

The controller starts at the Phase 1 sweet-spot N **[depends on Phase 1]** and adjusts every `window_size` prefetch resolutions (e.g. every 100 USEFUL + WASTED outcomes).

### 4.2 Implementation

**File:** `vllm/v1/kv_offload/tiering/manager.py`, on `TieringOffloadingManager`.

```python
class AdaptiveNController:
    """Hit-rate-driven adaptive prefetch depth.

    Tracks prefetch outcomes (USEFUL / WASTED) over a sliding window and
    adjusts N up or down based on the observed hit rate. The controller is
    stateless across resets — N resets to the initial value on reset_cache().
    """

    def __init__(
        self,
        initial_n: int,           # Phase 1 sweet-spot N [depends on Phase 1]
        n_min: int = 0,           # floor (0 = stop prefetching)
        n_max: int = 256,         # cap (matches Phase 1 sweep ceiling)
        high_threshold: float = 0.8,   # increase N above this hit rate
        low_threshold: float = 0.2,    # decrease N below this hit rate
        window_size: int = 100,        # adjustments per N resolutions
    ):
        self._n = initial_n
        self._n_min = n_min
        self._n_max = n_max
        self._high_threshold = high_threshold
        self._low_threshold = low_threshold
        self._window_size = window_size
        self._useful = 0
        self._wasted = 0
        self._since_adjust = 0

    @property
    def n(self) -> int:
        return self._n

    def record_useful(self) -> None:
        self._useful += 1
        self._since_adjust += 1
        self._maybe_adjust()

    def record_wasted(self) -> None:
        self._wasted += 1
        self._since_adjust += 1
        self._maybe_adjust()

    def _maybe_adjust(self) -> None:
        if self._since_adjust < self._window_size:
            return
        total = self._useful + self._wasted
        if total == 0:
            return
        hit_rate = self._useful / total
        if hit_rate > self._high_threshold:
            self._n = min(self._n * 2, self._n_max)
        elif hit_rate < self._low_threshold:
            self._n = max(self._n // 2, self._n_min)
        # else: hold
        # Reset window
        self._useful = 0
        self._wasted = 0
        self._since_adjust = 0

    def reset(self) -> None:
        """Reset to initial N (called from reset_cache)."""
        self._n = self._initial_n
        self._useful = 0
        self._wasted = 0
        self._since_adjust = 0
```

Integration into `TieringOffloadingManager`:

```python
class TieringOffloadingManager(OffloadingManager):
    def __init__(self, primary_tier, secondary_tiers=None, prefetch_chunks: int = 0,
                 adaptive_n: bool = False):
        ...
        if adaptive_n:
            self._n_controller = AdaptiveNController(
                initial_n=prefetch_chunks,   # Phase 1 sweet spot
            )
        else:
            self._n_controller = None
        self._prefetch_chunks = prefetch_chunks   # static fallback

    @property
    def prefetch_chunks(self) -> int:
        # The connector scheduler reads this to decide how many to prefetch.
        return self._n_controller.n if self._n_controller else self._prefetch_chunks
```

The `lookup()` outcome tracking (from Phase 1 Step 4d) feeds the controller:

```python
# In lookup(), prefetch outcome tracking:
if key in self._prefetched_keys:
    if primary_hit is LookupResult.HIT:
        self._prefetched_keys.discard(key)
        self._metrics.on_prefetch_useful(PREFETCH_TIER_LABEL)
        if self._n_controller:
            self._n_controller.record_useful()
    elif primary_hit is LookupResult.MISS:
        self._prefetched_keys.discard(key)
        self._metrics.on_prefetch_wasted(PREFETCH_TIER_LABEL)
        if self._n_controller:
            self._n_controller.record_wasted()
```

And `reset_cache()` resets the controller:

```python
@override
def reset_cache(self) -> None:
    for _ in self._prefetched_keys:
        self._metrics.on_prefetch_wasted(PREFETCH_TIER_LABEL)
    self._prefetched_keys.clear()
    if self._n_controller:
        self._n_controller.reset()
    ...
```

### 4.3 Config

New `kv_connector_extra_config` keys:

```yaml
kv_connector_extra_config:
  spec_name: TieringOffloadingSpec
  prefetch_chunks: 64              # initial N (Phase 1 sweet spot)
  adaptive_n: true                 # enable adaptive controller
  adaptive_n_high_threshold: 0.8   # [depends on Phase 1]
  adaptive_n_low_threshold: 0.2    # [depends on Phase 1]
  adaptive_n_window: 100           # [depends on Phase 1]
  adaptive_n_max: 256
```

When `adaptive_n: false`, the controller is not created and `prefetch_chunks` is static — identical to Phase 1. This is the control for the Phase 2 comparison.

### 4.4 Telemetry

Add a gauge for the current adaptive N value so the run report can plot N over time:

```python
# In TieringOffloadingMetrics (base.py):
ADAPTIVE_N = "vllm:kv_offload_tiering_adaptive_n"

# In spec.py build_metric_definitions:
metrics[TieringOffloadingMetrics.ADAPTIVE_N] = OffloadingGaugeMetadata(
    documentation="Current adaptive prefetch depth (N).",
)

# In metrics.py TieringMetricsTracker:
def set_adaptive_n(self, n: int) -> None:
    self._stats.set_gauge(TieringOffloadingMetrics.ADAPTIVE_N, n)
```

Call `self._metrics.set_adaptive_n(self._n_controller.n)` in `on_schedule_end()` when the controller exists. Plot this alongside the hit rate to see the feedback loop in action.

## 5. Component 2 — Feature-based block selection [depends on Phase 1]

This is the core of Phase 2. Instead of prefetching the next N blocks in sequence, the heuristic scores candidate blocks and prefetches the top N by score. The score estimates "will this block be demand-hit by the next turn's scan?"

### 5.1 Feature inventory

The experiment definition calls for temporal, spatial, and session features. Here is what's available, what's derivable, and what's net-new:

| Feature | Category | Source | Available? | Phase 2? |
|---|---|---|---|---|
| Access frequency | temporal | `KVCacheMetricsCollector` (sampled 0.01) | yes | yes |
| Recency (time since last access) | temporal | LRU/ARC cache policy ordering | yes | yes |
| Inter-access statistics | temporal | derive from access time history | derivable | yes |
| Block type (prompt vs. decode) | temporal | `offload_prompt_only` flag | yes | yes |
| Current storage tier | temporal | tiering manager's tracking | yes | yes |
| Sequence position | spatial | block hash chain (index in `offload_keys`) | yes | yes |
| Neighboring block activity | spatial | derive from adjacent blocks' access history | derivable | yes |
| Sibling access ratios | spatial | derive across KV groups | derivable | yes (multi-group only) |
| Conversation turn count | session | not tracked in vLLM | **net-new** | tentative |
| Reuse ratio | session | derivable from prefix overlap | derivable | yes |
| Inter-turn intervals | session | derivable from request arrival times | derivable | yes |
| Session recency | session | derivable from last request in session | derivable | yes |
| Session duration | session | derivable from first-to-last request | derivable | yes |

**Session features caveat.** The concept note flags that session-level features are "tracked on neither side today — net-new data collection." For Phase 2, the session features marked "derivable" use a **prefix-overlap proxy**: two requests sharing a prefix of N blocks are assumed to be in the same session (turn N and turn N+1 of the same conversation). This is an approximation — it works for the Weka workload's ~93% reuse but may misattribute in workloads with heavy cross-session prefix sharing. Full session tracking (with explicit conversation IDs from the workload) is deferred to Phase 3.

### 5.2 Block access history tracker

**File:** `vllm/v1/kv_offload/tiering/manager.py` (new class, composed into `TieringOffloadingManager`).

The tracker records per-block access events and derives the temporal features. It hooks into the existing `lookup()` and `touch()` paths:

```python
@dataclass(slots=True)
class BlockAccessRecord:
    """Per-block access history for heuristic scoring."""
    access_count: int = 0           # frequency
    last_access_time: float = 0.0   # recency (time.monotonic)
    first_access_time: float = 0.0  # for inter-access stats
    prev_access_time: float = 0.0   # for inter-access gap
    tier_idx: int = 0               # current storage tier
    # Derived (computed on demand, not stored):
    # - inter_access_gaps: list of gaps (or mean/variance)
    # - recency_weight: decays with time since last_access


class BlockAccessTracker:
    """Tracks per-block access history for heuristic scoring.

    Hooks into lookup() and touch() to record access events. Maintains
    a bounded history (evicts records for blocks no longer in any tier).
    """

    def __init__(self, max_blocks: int = 100_000):
        self._records: dict[OffloadKey, BlockAccessRecord] = {}
        self._max_blocks = max_blocks

    def record_access(self, key: OffloadKey, tier_idx: int) -> None:
        """Called from lookup() on any HIT (primary or secondary)."""
        now = time.monotonic()
        rec = self._records.get(key)
        if rec is None:
            rec = BlockAccessRecord(
                access_count=1,
                first_access_time=now,
                last_access_time=now,
                prev_access_time=now,
                tier_idx=tier_idx,
            )
            self._records[key] = rec
        else:
            rec.prev_access_time = rec.last_access_time
            rec.last_access_time = now
            rec.access_count += 1
            rec.tier_idx = tier_idx

    def record_touch(self, key: OffloadKey) -> None:
        """Called from touch() — updates recency without incrementing count."""
        rec = self._records.get(key)
        if rec is not None:
            rec.last_access_time = time.monotonic()

    def get_record(self, key: OffloadKey) -> BlockAccessRecord | None:
        return self._records.get(key)

    def evict(self, key: OffloadKey) -> None:
        """Called when a block is removed from all tiers."""
        self._records.pop(key, None)

    def reset(self) -> None:
        self._records.clear()
```

Integration: call `record_access()` from `lookup()` on any `HIT` (primary or secondary), and `record_touch()` from `touch()`. The tracker is memory-bounded by `max_blocks` (100K records × ~72 bytes ≈ 7 MB — trivial).

### 5.3 Spatial feature derivation

Neighboring block activity is derivable from the block hash chain. In `_maximal_prefix_lookup`, the `keys` list is the request's `offload_keys` in sequence — so `keys[i-1]` and `keys[i+1]` are the spatial neighbors of `keys[i]`. The heuristic can query the access tracker for the neighbors' records:

```python
def neighbor_activity(self, keys: list[OffloadKey], idx: int, window: int = 2) -> float:
    """Average access_count of blocks within ±window of idx in the sequence."""
    start = max(0, idx - window)
    end = min(len(keys), idx + window + 1)
    counts = []
    for i in range(start, end):
        if i == idx:
            continue
        rec = self.get_record(keys[i])
        if rec is not None:
            counts.append(rec.access_count)
    return sum(counts) / len(counts) if counts else 0.0
```

This is O(window) per candidate — cheap.

### 5.4 Session feature derivation (prefix-overlap proxy)

The session features use a prefix-overlap proxy: requests sharing a long prefix are assumed to be in the same conversation. The tracker derives:

| Feature | Derivation |
|---|---|
| Reuse ratio | fraction of the current request's `offload_keys` that have `access_count > 0` (were seen in a prior turn) |
| Inter-turn interval | time between the current request's first access and the most recent prior access of any of its prefix blocks |
| Session recency | time since the most recent prior access of any prefix block |
| Session duration | time between the first and last access of any prefix block |
| Turn count | number of distinct access bursts (clustered in time) for the prefix blocks |

These are computed per-request at `on_new_request` time, from the access tracker's records. They're approximate but don't require new infrastructure.

### 5.5 Heuristic scoring function

**File:** `vllm/v1/kv_offload/tiering/manager.py` (new method on `TieringOffloadingManager`).

The scoring function combines the features into a single "temperature" estimate — a poor man's version of the Phase 3 XGBoost predictor. The weights are hand-tuned during benchmarking:

```python
def _heuristic_score(
    self,
    key: OffloadKey,
    keys: list[OffloadKey],
    idx: int,
    req_context: ReqContext,
) -> float:
    """Estimate the probability that this block will be demand-hit next turn.

    Higher score = more likely to be useful to prefetch. The score combines
    temporal, spatial, and session features with hand-tuned weights.
    """
    rec = self._access_tracker.get_record(key)

    # Temporal features
    frequency = rec.access_count if rec else 0
    recency = (time.monotonic() - rec.last_access_time) if rec else float('inf')
    # Recency weight: exponentially decays with time (half-life ~60s, tunable)
    recency_weight = math.exp(-recency / 60.0) if rec else 0.0

    # Spatial features
    neighbor_act = self._access_tracker.neighbor_activity(keys, idx)

    # Session features (prefix-overlap proxy)
    reuse_ratio = self._session_reuse_ratio(keys)
    inter_turn_interval = self._session_inter_turn_interval(keys)

    # Weighted score [depends on Phase 1 for weight calibration]
    score = (
        self._w_frequency * frequency
        + self._w_recency * recency_weight
        + self._w_neighbor * neighbor_act
        + self._w_reuse * reuse_ratio
        + self._w_interval * (1.0 / (1.0 + inter_turn_interval))  # recent sessions score higher
    )
    return score
```

The weights (`_w_frequency`, `_w_recency`, etc.) are config knobs with initial values **[depends on Phase 1]** — the Phase 1 hit-rate data and `PREFETCH_WASTED` breakdown inform which features matter most. If Phase 1 shows eviction-dominated waste, recency and neighbor activity (which predict survival in primary) should be weighted higher. If Phase 1 shows never-reached waste, reuse ratio (which predicts next-turn demand) should be weighted higher.

### 5.6 Selection: top-N by score, not next-N in sequence

The prefetch hook in `_maximal_prefix_lookup` changes from "slice the next N keys" to "score all candidate keys and select the top N":

```python
# In _maximal_prefix_lookup, MISS branch:
case LookupResult.MISS:
    n = getattr(self.manager, "prefetch_chunks", 0)
    if n > 0:
        # Phase 1: upcoming = keys[local_idx + 1 : local_idx + 1 + n]
        # Phase 2: score candidates and select top N
        candidates = keys[local_idx + 1 : local_idx + 1 + n * 4]  # over-fetch 4x, then filter
        scored = [
            (self.manager._heuristic_score(k, keys, local_idx + 1 + i, req_context), k)
            for i, k in enumerate(candidates)
        ]
        scored.sort(reverse=True)  # highest score first
        upcoming = [k for _, k in scored[:n]]
        if upcoming:
            self.manager.prefetch(upcoming, req_context)
    break
```

The over-fetch factor (4x) is a tunable knob. It scores 4N candidates and selects the top N, so the scoring overhead is O(4N) per miss — cheap for N ≤ 256. The factor trades scoring work for selection quality; 4x is a starting point **[depends on Phase 1]**.

### 5.7 Ablation: feature importance

Phase 2's exit criterion requires "a documented feature set with observed predictive value." Run ablations by zeroing each feature weight in turn and measuring the hit-rate change:

| Ablation | What it tests |
|---|---|
| `_w_frequency = 0` | does access frequency matter? |
| `_w_recency = 0` | does recency matter? |
| `_w_neighbor = 0` | does neighboring block activity matter? |
| `_w_reuse = 0` | does session reuse matter? |
| `_w_interval = 0` | does inter-turn interval matter? |
| all zero (score = 0, random selection) | is the heuristic better than random? |
| all equal (score = sum, unweighted) | does weighting matter? |

The ablation results feed Phase 3: features with high predictive value become inputs to the XGBoost model; features with low predictive value are candidates for pruning.

## 6. Component 3 — Sliding-window group support [design-complete]

Phase 1 restricted the prefetch hook to `_maximal_prefix_lookup` (full-attention groups). Phase 2 extends to `_sliding_window_lookup`, which scans from the end and has different semantics.

### 6.1 The difference

`_sliding_window_lookup` (in `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`) scans from the **end** of the keys list, looking for a run of `sliding_window_size` consecutive hits. It returns the end index of the last run. The read-ahead for sliding-window groups must account for the window: only blocks within the sliding window are useful, so prefetching beyond the window is wasted.

### 6.2 The hook

```python
# In _sliding_window_lookup, on the first MISS that breaks the consecutive run:
case LookupResult.MISS:
    n = getattr(self.manager, "prefetch_chunks", 0)
    if n > 0:
        # Only prefetch within the sliding window — blocks beyond the
        # window are never useful to this group.
        window_remaining = sliding_window_size - consecutive_hits
        n_effective = min(n, window_remaining)
        if n_effective > 0:
            upcoming = keys[idx - n_effective : idx]  # before the miss (scan is right-to-left)
            if upcoming:
                self.manager.prefetch(upcoming, req_context)
    consecutive_hits = 0
```

The direction is reversed (right-to-left scan), so the read-ahead is *before* the miss index, not after. And the count is clamped to the window remaining — no point prefetching beyond the sliding window.

### 6.3 Interaction with the heuristic

The feature-based scoring (Component 2) applies the same way — the candidates are scored and the top N (clamped to the window) are selected. The spatial feature (neighboring block activity) is especially relevant for sliding-window groups, since the window creates a spatial locality pattern.

## 7. Telemetry and measurement

### 7.1 Reuse of Phase 1 counters

All five Phase 1 counters (`PREFETCH_ATTEMPTED` / `PROMOTED` / `SKIPPED` / `USEFUL` / `WASTED`) are still the primary signals. Phase 2 adds:

| New metric | Type | What it measures |
|---|---|---|
| `ADAPTIVE_N` | gauge | current adaptive N value (Component 1) |
| `PREFETCH_SCORE` | histogram | distribution of heuristic scores for prefetched blocks (Component 2) — high scores should correlate with USEFUL |
| `FEATURE_WEIGHT_*` | gauge | current weight of each feature (for ablation runs) |

### 7.2 The comparison matrix

Phase 2's exit criterion is "the heuristic matches or beats the best Phase 1 N." The comparison matrix:

| Configuration | N | Selection | Groups | Purpose |
|---|---|---|---|---|
| Phase 1 best (static N*) | static | spatial read-ahead | full-attn only | baseline to beat |
| Phase 2a (adaptive N) | adaptive | spatial read-ahead | full-attn only | isolates adaptive N's contribution |
| Phase 2b (heuristic selection) | static N* | feature-weighted | full-attn only | isolates feature selection's contribution |
| Phase 2c (both) | adaptive | feature-weighted | full-attn only | the full Phase 2 system |
| Phase 2d (full coverage) | adaptive | feature-weighted | full-attn + sliding-window | coverage extension |

This matrix isolates each component's contribution. The exit criterion is that 2c beats 2a (feature selection helps) and 2a beats Phase 1 best (adaptive N helps), or that 2c beats Phase 1 best overall with the improvement attributable to the heuristic.

### 7.3 Feature importance ablation

Run the ablation matrix (Section 5.7) at the best configuration (2c). Report the hit-rate change per ablation. Features with large hit-rate drops are the most predictive; features with no change are candidates for pruning.

### 7.4 Cross-turn pairing

Since the benefit is cross-turn, pair turns (N, N+1) by `conversation_id` and measure the N+1 latency improvement attributable to N's prefetch. Compare:
- Phase 1 best: N+1 latency with static-N prefetch on turn N.
- Phase 2c: N+1 latency with heuristic prefetch on turn N.

The heuristic should show a larger N+1 improvement because it selects blocks more likely to survive until turn N+1's scan.

## 8. Run plan and exit criteria

This maps to Phase 2 of [[Methodology/01 - Experiment Definition]].

1. **[depends on Phase 1]** Finalize the adaptive N thresholds, feature weights, and over-fetch factor from the Phase 1 sweep results.
2. **Implement** Component 1 (adaptive N) — deploy and measure against Phase 1 best (row 2a in the comparison matrix).
3. **Implement** Component 2 (feature collection + heuristic scoring) — deploy and measure (rows 2b, 2c).
4. **Implement** Component 3 (sliding-window support) — deploy and measure (row 2d).
5. **Run the comparison matrix** (Section 7.2), 3 repetitions per cell, same batch, same node class, on the `cc-traces-weka-061526` workload.
6. **Run the feature ablation** (Section 5.7) at the best configuration.
7. **Report** as a dated note under `Research/ABC/`: comparison matrix results, feature importance table, cross-turn pairing analysis, failure modes, and a **handoff note for Phase 3**.
8. **Exit gate** (from the experiment definition):
   - The heuristic (2c) matches or beats the best Phase 1 N across the tested conditions, with the improvement attributable to the heuristic (not noise).
   - A documented feature set with observed predictive value (from the ablation), feeding Phase 3 training.
   - Failure modes of the heuristic identified (the cases a learned model is expected to fix).

## 9. Risks and unknowns

- **Feature weight calibration.** The initial weights are guesses until Phase 1 data informs them. If the weights are wrong, the heuristic may perform worse than spatial read-ahead. Mitigation: the ablation matrix identifies which features matter, and the weights can be re-tuned between runs.
- **Session feature approximation.** The prefix-overlap proxy may misattribute sessions in workloads with heavy cross-session prefix sharing. The Weka workload's ~93% reuse makes this unlikely, but it's a risk for generalization. Mitigation: the ablation tests whether session features matter at all; if they don't, the proxy's inaccuracy is moot.
- **Store-path conflict.** The current turn's recompute stores may evict prefetched blocks before the next turn. Phase 2 monitors this via `PREFETCH_WASTED` but doesn't solve it (the cost-benefit model is Phase 3). If the conflict dominates, the heuristic's selection quality may not matter — the blocks are evicted regardless. Mitigation: the heuristic can deprioritize blocks in the recompute range (Section 3.2), but this is a partial fix.
- **Scoring overhead.** The heuristic scores 4N candidates per miss. For N=256, that's 1024 scoring calls per miss, each O(1) (dict lookups + arithmetic). At 32-way concurrency with frequent misses, this is ~30K scoring calls per step — negligible vs the secondary-tier lookup latency. But if the over-fetch factor is too high, it wastes CPU. Mitigation: the over-fetch factor is tunable.
- **Adaptive N oscillation.** The hit-rate feedback loop may oscillate if the window is too small or the thresholds are too tight. Mitigation: the window size (100) and thresholds (0.8/0.2) are conservative starting points; the hysteresis (only adjust every `window_size` resolutions) dampens oscillation.

## 10. Out of scope for Phase 2

Explicitly **not** in Phase 2 (deferred to Phase 3 per [[Methodology/01 - Experiment Definition]]):

- The XGBoost temperature prediction model — Phase 3. The heuristic is the hand-tuned precursor that de-risks the feature engineering.
- The cost-benefit migration gate $\text{Benefit} > N \times \text{Cost}$ — Phase 3. Phase 2 monitors the store-path conflict but doesn't model it.
- Full session tracking with explicit conversation IDs — Phase 3. Phase 2 uses the prefix-overlap proxy.
- Temperature export to the llm-d endpoint picker — Phase 3.
- Multi-hop prefetching through intermediate tiers — Phase 3. Phase 2 prefetches secondary→primary only.
- The four-tier placement policy (Hot→GPU / Warm→CPU / Cool→NVMe / Cold→CephFS) — Phase 3. Phase 2 only prefetches into the CPU primary tier.

## 11. Related

- [[Methodology/01 - Experiment Definition]] — Phase 2 objective, method, and exit criteria.
- [[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide]] — the Phase 1 implementation this builds on; the cross-turn benefit model; the five-counter telemetry; the global `_prefetched_keys` tracking.
- [[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]] — Phase 0 baseline data.
- [[AgentX Workload Definition]] — the agentic-replay workload family; the `061526` corpus.
- [[Experiment Methodology]] — run structure, acceptance gates, repetition requirements.
- [[Activity-Based KV Cache Offloading]] — concept note; the feature vector definition; the implementation-placement verdict.
- [[00 - Index]] — ABC project index.
- Workload dataset: [semianalysisai/cc-traces-weka-061526](https://huggingface.co/datasets/semianalysisai/cc-traces-weka-061526) (HuggingFace, public).