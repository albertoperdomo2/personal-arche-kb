---
title: "Phase 1 Naive Prefetch Implementation Guide — Part 2"
date: "2026-08-14"
type: "implementation-guide"
experiment: "ABC"
status: "historical-rejected"
part: 2
---

# Phase 1 — Naive Proactive Prefetching Implementation Guide — Part 2

Back to [[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide|the guide landing page]] or [[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide/01 - Reactive baseline through implementation|Part 1 — reactive baseline through implementation]].

## 4. Telemetry and measurement

Reuse the existing tiering telemetry — no new instrumentation is required to evaluate the toy, beyond the Step 4 counter.

### 4.0 Workload — Weka traces `cc-traces-weka-061526`

The Phase 1 sweep uses the SemiAnalysis Weka trace corpus `semianalysisai/cc-traces-weka-061526` (HuggingFace, public), loaded via the AIPerf loader plugin `--public-dataset semianalysis_cc_traces_weka_with_subagents`. This is the same agentic-replay workload family documented in [[AgentX Workload Definition]], built 2026-06-15 with tighter filtering than the earlier `060826` corpus.

Corpus statistics (from the dataset card):


| Field                                      | Value                                               |
| ------------------------------------------ | --------------------------------------------------- |
| Traces                                     | 233                                                 |
| Main turns                                 | 38,529                                              |
| Subagent groups                            | 787                                                 |
| Subagent inner requests                    | 22,467                                              |
| Total model requests                       | 60,996                                              |
| Total input tokens (hash-block count × 64) | 12.63 B                                             |
| Total output tokens                        | 59.76 M                                             |
| Block size                                 | 64 tokens                                           |
| ISL cap                                    | 990,016 tokens (drops KV-block overcount artifacts) |
| Min requests per session                   | 20                                                  |
| Peak concurrent subagent groups            | ≤ 10                                                |


Two structural properties drive the N selection:

1. **Bimodal request-size distribution.** The corpus mixes small subagent inner requests (a few hundred to a few thousand tokens = 6–40 blocks) with large main-turn requests (45k–52k tokens ≈ 700–815 blocks of prefix). The large requests are the ones that stress the secondary tier: their prefixes exceed the 64 GiB CPU tier's residency under 32-way concurrency, so a meaningful fraction is evicted to NVMe/CephFS between turns and must be promoted back on the next turn. The toy's benefit concentrates on these large requests.
2. **High prefix reuse (~93–94%).** Within a play, turns share prefix history; only the delta (new user message + prior assistant output) is novel per turn. So the *secondary fetch length* — the number of chunks a request needs to promote from secondary on a miss — is not the full prefix, but the prefix portion evicted from CPU but still resident in secondary. Under 32-way concurrency and 64 GiB CPU, this evicted fraction is the quantity N must cover.

> **The `in` field is hash-block count × 64, not a true tokenizer count.** The dataset card warns it overcounts the real prompt size by up to ~260k tokens in the heavy-cache-write tail. For KV-cache block accounting this is the *relevant* number (it is the count of 64-token prefix blocks), but it means the 12.63 B / 60,996 ≈ 207k mean `in` overstates the real mean prompt. The per-request prefix-block count from the trace is the right input for sizing N.

### 4.1 Primary signal (does latency improve?)

From the Nemotron report's metric set, compare across N:


| Metric                            | Source                                          | Expectation if prefetch helps                                                 |
| --------------------------------- | ----------------------------------------------- | ----------------------------------------------------------------------------- |
| Request latency P50/P90           | AIPerf profile                                  | lower as N grows, then rises when N overshoots                                |
| TTFT P50                          | AIPerf profile                                  | lower (fewer deferred prefill steps)                                          |
| Tiering lookup P50/P90/P99        | `kv_offload_tiering_lookup_async_delay_seconds` | **fewer lookup events per request**; per-event delay unchanged (same backend) |
| Blocked requests (avg concurrent) | tiering telemetry                               | lower (requests spend fewer steps stalled)                                    |
| External-token share              | prompt-token-source counters                    | unchanged or slightly higher (more secondary hits served)                     |
| **Prefetch hit rate**             | `PREFETCH_USEFUL / (PREFETCH_USEFUL + PREFETCH_WASTED)` | high at the sweet spot, drops at large N (eviction churn) |


The decisive comparison is **lookup-event count per request**, not per-event latency: the toy reduces the *number* of deferred steps, not the speed of each secondary read. Add a derived "lookup events per completed request" = `BLOCK_QUERIES / completed_requests` and plot vs N.

The **prefetch hit rate** is the second decisive signal: it tells you whether the latency improvement is *caused by the prefetch* (high hit rate at the sweet-spot N) or *despite the prefetch* (low hit rate, meaning the demand path is still doing the work and the latency improvement comes from something else). A high hit rate that drops at large N confirms the U-curve's right side is eviction churn — the prefetched blocks were promoted but evicted before the GPU could use them. Plot the hit rate alongside the latency U-curve; they should move together (high hit rate ↔ low latency) until the eviction threshold, where they diverge (hit rate drops, latency rises).

### 4.2 Negative-signal metrics (is prefetch hurting?)


| Metric                          | Why it matters                                                  |
| ------------------------------- | --------------------------------------------------------------- |
| `PROMOTION_ALLOCATION_FAILURES` | rising ⇒ prefetch is crowding out demand promotions             |
| `PRIMARY_WRITE_USAGE_PERC`      | sustained near 100% ⇒ primary tier saturated by read-ahead      |
| Total throughput P50            | dropping ⇒ prefetch overhead exceeds its benefit (N too large)  |
| Recompute share                 | rising ⇒ prefetched blocks evicted demand blocks (LRU pressure) |


### 4.3 The N sweep

#### Anchoring N to the workload

The read-ahead knob N is measured in **offload chunks** (one chunk = `blocks_per_chunk` GPU blocks = `blocks_per_chunk × 64` tokens). The token coverage of one read-ahead is:

$$
\text{coverage}(N) = N \times \text{blocks\_per\_chunk} \times 64 \text{ tokens}
$$

The sweep must span three regimes relative to **K**, the typical number of chunks a large request fetches from the secondary tier on a miss:


| Regime     | Condition     | Expected outcome                                                                                                                                                                      |
| ---------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Too small  | $N \ll K$     | read-ahead covers a tiny fraction of the secondary fetch; the request is still deferred $\lceil K/N \rceil$ times → little benefit over reactive                                      |
| Sweet spot | $N \approx K$ | one read-ahead covers the whole secondary fetch in 1–2 deferred steps → large latency reduction                                                                                       |
| Too large  | $N \gg K$     | read-ahead overshoots the fetch, spending CPU-tier capacity and transfer bandwidth on blocks the request will not reach before they are evicted under 32-way concurrency → regression |


**Estimating K from the workload.** A large main-turn request carries ~700–815 blocks of prefix (45–52k tokens). Under 32-way concurrency with a 64 GiB CPU tier and ~93% reuse, the fraction evicted from CPU but resident in secondary between turns is the fetch length. The Phase 0 baseline ([[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]]) measured a stall rate of ~587–591 events/s over 1,800 s (~1.06 M lookup events); dividing by the completed-request count yields the empirical K. **Measure K from the Phase 0 run** (`BLOCK_QUERIES` / completed requests, or more precisely, secondary-`HIT` promotions per request) and center the sweep on it. Until that measurement is in, the prefix-size distribution above gives the bracket: K is bounded above by the large-request prefix (~700–815 blocks) and below by the per-turn delta (~7% of prefix ≈ 50–60 blocks).

#### Proposed sweep

`prefetch_chunks ∈ {0, 4, 8, 16, 32, 64, 128, 256}`, with `0` as the within-batch control (reactive baseline). The values are chosen to span the three regimes for the Weka workload:


| N   | Token coverage (blocks_per_chunk=1) | Token coverage (blocks_per_chunk=8) | Expected regime                                                           |
| --- | ----------------------------------- | ----------------------------------- | ------------------------------------------------------------------------- |
| 0   | 0 (control)                         | 0 (control)                         | reactive baseline                                                         |
| 4   | 256 tok                             | 2,048 tok                           | too small — covers <10% of a large fetch                                  |
| 8   | 512 tok                             | 4,096 tok                           | too small — covers <15% of a large fetch                                  |
| 16  | 1,024 tok                           | 8,192 tok                           | transition — covers a small fetch or the per-turn delta                   |
| 32  | 2,048 tok                           | 16,384 tok                          | approaching sweet spot — covers a moderate fetch                          |
| 64  | 4,096 tok                           | 32,768 tok                          | expected sweet spot — covers a typical large fetch in 1–2 steps           |
| 128 | 8,192 tok                           | 65,536 tok                          | past sweet spot — overshoots, CPU-tier pressure builds                    |
| 256 | 16,384 tok                          | 131,072 tok                         | too large — 32 concurrent read-aheads compete for 64 GiB CPU → regression |


> **Confirm `blocks_per_chunk**` from the Nemotron run config before the sweep; the token-coverage column shifts with it but the chunk-count sweep is unchanged. If `blocks_per_chunk` is large (e.g. 8), the sweet spot shifts left (smaller N); if it is 1, the sweet spot shifts right. The sweep covers both cases.

#### Expected U-curve

The figure below shows the expected shape: latency (or lookup-events-per-request) as a function of N. The left flat region is the reactive cost surviving; the dip is the sweet spot where read-ahead amortizes the secondary fetch; the right rise is CPU-tier eviction churn and wasted bandwidth.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 2 — Expected U-curve: latency vs. prefetch_chunks (N)",
  "width": 600,
  "height": 300,
  "background": "white",
  "data": {
    "values": [
      {"N": 4,   "latency": 98, "regime": "too small"},
      {"N": 8,   "latency": 92, "regime": "too small"},
      {"N": 16,  "latency": 75, "regime": "transition"},
      {"N": 32,  "latency": 52, "regime": "sweet spot"},
      {"N": 64,  "latency": 45, "regime": "sweet spot"},
      {"N": 128, "latency": 68, "regime": "too large"},
      {"N": 256, "latency": 95, "regime": "too large"}
    ]
  },
  "layer": [
    {
      "mark": {"type": "line", "point": false, "strokeWidth": 2, "color": "#1f77b4"},
      "encoding": {
        "x": {"field": "N", "type": "quantitative", "title": "prefetch_chunks (N)", "scale": {"type": "log", "domain": [1, 256]}},
        "y": {"field": "latency", "type": "quantitative", "title": "Request latency P50 (normalized, %)", "scale": {"domain": [0, 110]}}
      }
    },
    {
      "mark": {"type": "circle", "size": 100, "stroke": "white", "strokeWidth": 1},
      "encoding": {
        "x": {"field": "N", "type": "quantitative", "scale": {"type": "log", "domain": [1, 256]}},
        "y": {"field": "latency", "type": "quantitative", "scale": {"domain": [0, 110]}},
        "color": {"field": "regime", "type": "nominal", "legend": {"title": "Regime"}}
      }
    },
    {
      "mark": {"type": "rule", "strokeDash": [4, 4], "color": "gray", "size": 1},
      "encoding": {"y": {"datum": 100}}
    },
    {
      "mark": {"type": "text", "dx": 4, "dy": -8, "color": "gray", "fontSize": 10},
      "encoding": {
        "x": {"datum": 256},
        "y": {"datum": 100},
        "text": {"datum": "N = 0 reactive baseline"}
      }
    }
  ]
}

```

The dashed line at 100% is the reactive baseline (N = 0). The curve is illustrative — the actual sweet-spot N and depth depend on K, `blocks_per_chunk`, and the CPU-tier eviction dynamics under this workload. The Phase 1 exit criterion is a **repeatable, attributable** latency improvement at some N > 0 over the N = 0 control, with the U-shape visible across the sweep.

#### Measurement protocol

Minimum 3 paired repetitions per cell; report mean ± CI and paired-request tail analysis (pair by `conversation_id` / turn / depth, per [[Experiment Methodology]]). Plot three curves against N:

1. **Latency** (request P50/P90) — the U-curve (Figure 2 shape).
2. **Lookup-events-per-request** (`BLOCK_QUERIES` / completed requests) — should show the same U-shape but inverted (high at small N, dips at the sweet spot, rises again at large N due to evicted-and-refetched churn).
3. **Prefetch hit rate** (`PREFETCH_USEFUL / (PREFETCH_USEFUL + PREFETCH_WASTED)`) — should be high at the sweet spot (the prefetch is useful) and drop at large N (prefetched blocks evicted before use). This is the signal that confirms the latency improvement is *caused by the prefetch*, not despite it.

The hit rate and latency curves should move together until the eviction threshold: high hit rate ↔ low latency. Where they diverge (hit rate drops, latency rises) is the onset of the U-curve's right side — the point where N is too large.

## 5. Run plan and exit criteria

This maps directly to Phase 1 of [[Methodology/01 - Experiment Definition]].

1. **Implement** Steps 1–4 on a vLLM fork/branch; keep `N = 0` path identical to `main`.
2. **Unit-test** `_try_promote` and `prefetch` with a mock primary tier and a mock `SecondaryTierManager`: assert no double-promotion, primary-full tolerance, filter honored, `N = 0` no-op.
3. **Measure K from Phase 0.** From the existing Nemotron baseline run ([[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]]), compute the empirical secondary-fetch length per large request: secondary-`HIT` promotions per completed request, or `BLOCK_QUERIES` / completed requests. This anchors the sweep's sweet-spot expectation (Section 4.3).
4. **Deploy** the branch image to the PSAP cluster; record the full image digest (per [[Experiment Methodology]]).
5. **Sweep** `prefetch_chunks ∈ {0, 4, 8, 16, 32, 64, 128, 256}`, 3 repetitions each, same batch, same node class, on the `cc-traces-weka-061526` workload (Section 4.0). `0` is the within-batch control.
6. **Report** as a dated note under `Research/ABC/`: latency curves vs N, lookup-events-per-request vs N, prefetch hit rate vs N, negative-signal metrics, the measured K, and a **go/no-go decision**. The report should show the U-curve (Figure 2 shape), the hit rate curve alongside it, and identify the sweet-spot N where hit rate is high and latency is low.
7. **Exit gate** (from the experiment definition): a measurable, repeatable latency change for at least three values of N, reported as mean ± CI with paired-request analysis, plus a recorded decision on whether to proceed to Phase 2.

## 6. Risks and unknowns

- **Primary-tier eviction churn (the right side of the U-curve).** Read-ahead fills the CPU tier with blocks the request *will* need soon, but under concurrency 32 several large requests' read-aheads compete for the same 64 GiB CPU tier. At N = 128–256, 32 concurrent read-aheads of 8k–16k blocks each can demand far more CPU residency than the tier holds, so prefetched blocks evict each other (or evict demand blocks), and latency regresses. The `PRIMARY_WRITE_USAGE_PERC` and recompute-share metrics catch this; the U-curve's right side quantifies it. The bimodal Weka workload makes this acute: the large main-turn requests are exactly the ones that read-ahead, and they arrive in bursts when subagents rejoin.
- **Interaction with `offload_prompt_only`.** `OffloadingSpec.offload_prompt_only` defaults to `True` (decode blocks are not offloaded). Read-ahead only touches prompt-prefix chunks, so this is consistent — but confirm the prefetched keys are all prompt chunks during the run. The Weka workload's ~93% reuse means most prefetched prefix chunks are genuinely reused within the play, which is what makes read-ahead viable here; a low-reuse workload would waste more.
- **Censored P99.** The Nemotron report notes overflow buckets are unavailable, so P99 at 10 s is a lower bound. Use P50/P90 and lookup-event-count as the primary signals; treat P99 as directional only until overflow is exported (a Phase 0 telemetry-gap item).
- **Multi-group models.** `offload_keys` are per KV group; `prefetch` is called within one group's scan. For multi-group models the toy is applied per full-attention group independently. This is fine for Phase 1 but must be revisited if groups have very different chunk sizes.
- **K estimation uncertainty.** The sweet-spot N depends on K (the secondary-fetch length per large request), which is not yet measured directly. The sweep spans a wide range (4–256) precisely to bracket the uncertainty; if the measured K falls outside this range, extend the sweep before concluding.

## 7. Out of scope for Phase 1

Explicitly **not** in this toy (deferred to later phases per [[Methodology/01 - Experiment Definition]]):

- Any prediction model (XGBoost or otherwise) — Phase 3.
- The cost-benefit migration gate $\text{Benefit} > N \times \text{Cost}$ — Phase 3.
- Access-frequency / recency feature collection — Phase 2 (the toy uses only sequence order, not access history).
- Adaptive N based on prefetch hit rate — Phase 2. The hit rate measured in Phase 1 (`PREFETCH_USEFUL / (PREFETCH_USEFUL + PREFETCH_WASTED)`) is the input signal for an adaptive read-ahead policy: ~100% hit rate → increase N (2→4→8→16, capped); ~0% → stop prefetching; ~50% → hold or reduce N. Phase 1 uses static N; the sweep data informs the adaptive thresholds.
- Session-aware prefetching on conversation migration — Phase 3.
- Sliding-window-group read-ahead — Phase 2.
- Temperature export to the llm-d endpoint picker — Phase 3.

## 8. Related

- [[Methodology/01 - Experiment Definition]] — Phase 1 objective, method, and exit criteria.
- [[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]] — Phase 0 baseline data and the lookup-delay numbers this toy targets.
- [[AgentX Workload Definition]] — the agentic-replay workload family; the `061526` corpus used here is a tighter-filtered build of the same family.
- [[Experiment Methodology]] — run structure, acceptance gates, repetition requirements.
- [[Activity-Based KV Cache Offloading]] — concept note; the implementation-placement verdict (prediction + placement live in core vLLM `vllm/v1/kv_offload`; this guide follows that verdict).
- [[00 - Index]] — ABC project index.
- Workload dataset: [semianalysisai/cc-traces-weka-061526](https://huggingface.co/datasets/semianalysisai/cc-traces-weka-061526) (HuggingFace, public).