---
title: "Clean-prefetch v1 repeat — image-provenance invalidation of attempted v2 validation"
date: "2026-08-22"
type: "experiment-report"
experiment: "ABC"
status: "invalid-for-v2-valid-v1-repeat"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
runtime_image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-clean-prefetch-v1"
runtime_digest: "sha256:7c977defec51c588339290b8259b66b57413212c26810c0f09a7d10aff8b7a57"
control_run: "7f096342a54241ce99c6a98e53a87ca4"
treatment_run: "a6fe8407257c4c90b57771bce155a1f2"
workload: "AIPerf inferencex-agentx-mvp / Weka trace"
concurrency: 64
max_num_seqs: "vLLM default"
duration_seconds: 1800
---

# Clean-prefetch v1 repeat — image-provenance invalidation of attempted v2 validation

## Verdict

**This pair did not test the demand-cutoff and scheduler-ordering correction.** Both rendered manifests, both live pod descriptions, and both image IDs show the old `v0.27.0-clean-prefetch-v1` image at digest `sha256:7c977def…`. That is exactly the digest used by the earlier concurrency-64 v1 pair.

The pair is therefore **invalid for evaluating v1.1/v2**, but useful as a second v1 repeat. As a v1 repeat it strengthens the existing negative policy diagnosis: the prefetch footprint stayed saturated, admission-to-ready delay became extremely stale, and eviction regret exceeded useful prefetch outcomes. End-to-end performance remained mixed and small.

## Runs and actual configuration

- [Treatment — a6fe8407257c4c90b57771bce155a1f2](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/a6fe8407257c4c90b57771bce155a1f2?workspace=benchflow)
  - deployment profile: `clean-prefetch-cpu-kv-offload-nvme`
  - `admission_prefetch_chunks=64`
  - candidate/apply/pending/tracked limits: 1024/64/64/64
  - proactive load batch: 8 chunks
- [Control — 7f096342a54241ce99c6a98e53a87ca4](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/7f096342a54241ce99c6a98e53a87ca4?workspace=benchflow)
  - deployment profile: `multi-tier-offloading-nvme`
  - no server-side admission-prefetch setting

Both used TP=8, one replica, 256 GiB CPU KV, local NVMe with 64 read and 64 write threads, 128K model length, `--gpu-memory-utilization=0.8`, default `max_num_seqs`, concurrency 64, seed 20260707, and RHOAI 3.5.0. The benchmark request gate was enabled in both cells but was inert in control.

## End-to-end results

| Metric | Control | Treatment | Treatment change |
|---|---:|---:|---:|
| Completed requests | 1,246 | 1,256 | +0.80% |
| Request throughput | 0.67717 req/s | 0.68261 req/s | +0.80% |
| Total-token throughput | 43,276.27 tok/s | 43,522.85 tok/s | +0.57% |
| Output-token throughput | 451.50 tok/s | 458.14 tok/s | +1.47% |
| Mean TTFT | 7,957.18 ms | 8,007.91 ms | +0.64% worse |
| p50 TTFT | 5,683.65 ms | 5,670.18 ms | -0.24% better |
| p95 TTFT | 23,628.43 ms | 24,892.24 ms | +5.35% worse |
| p99 TTFT | 30,705.49 ms | 30,792.63 ms | +0.28% worse |
| Mean request latency | 43,318.75 ms | 43,129.43 ms | -0.44% better |
| p95 request latency | 132,135.12 ms | 135,515.12 ms | +2.56% worse |
| Mean ITL | 55.988 ms | 54.277 ms | -3.06% better |
| p95 ITL | 94.460 ms | 87.720 ms | -7.14% better |
| p99 ITL | 164.716 ms | 129.513 ms | -21.37% better |

Both completed without AIPerf errors or cancellation. The work was not identical: treatment processed ten more requests, 441,484 more input tokens, and 12,210 more output tokens; branch completion also differed slightly. The small aggregate deltas must therefore not be treated as a causal win.

The prior v1 pair showed better TTFT but worse ITL; this repeat shows worse tail TTFT but better ITL. Throughput was slightly positive in both, mean request latency was effectively flat in both, and p95 request latency regressed in both. The sign reversals show that a single pair is dominated by run/workload variability at the observed effect size.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Treatment change across two identical-image v1 pairs",
  "width": 760,
  "height": 300,
  "data": {
    "values": [
      {"series": "Previous v1 pair", "metric": "Request throughput", "change_pct": 0.24},
      {"series": "New v1 repeat", "metric": "Request throughput", "change_pct": 0.80},
      {"series": "Previous v1 pair", "metric": "Mean TTFT", "change_pct": -2.27},
      {"series": "New v1 repeat", "metric": "Mean TTFT", "change_pct": 0.64},
      {"series": "Previous v1 pair", "metric": "p95 TTFT", "change_pct": -7.11},
      {"series": "New v1 repeat", "metric": "p95 TTFT", "change_pct": 5.35},
      {"series": "Previous v1 pair", "metric": "p95 request latency", "change_pct": 3.55},
      {"series": "New v1 repeat", "metric": "p95 request latency", "change_pct": 2.56},
      {"series": "Previous v1 pair", "metric": "Mean ITL", "change_pct": 2.28},
      {"series": "New v1 repeat", "metric": "Mean ITL", "change_pct": -3.06},
      {"series": "Previous v1 pair", "metric": "p99 ITL", "change_pct": 24.97},
      {"series": "New v1 repeat", "metric": "p99 ITL", "change_pct": -21.37}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "metric", "type": "nominal", "title": null, "sort": null, "axis": {"labelAngle": -30}},
    "y": {"field": "change_pct", "type": "quantitative", "title": "Treatment relative change (%)"},
    "color": {"field": "series", "type": "nominal", "title": "Run pair"},
    "column": {"field": "series", "type": "nominal", "header": {"labels": false, "title": null}},
    "tooltip": [
      {"field": "series", "type": "nominal"},
      {"field": "metric", "type": "nominal"},
      {"field": "change_pct", "type": "quantitative", "title": "Change (%)", "format": ".2f"}
    ]
  }
}
~~~

Negative is favorable for latency and positive is favorable for throughput. The chart is evidence of instability, not a performance claim.

## Mechanism evidence

Profiling-phase vLLM counter deltas for treatment:

| Status / measurement | Value |
|---|---:|
| Submitted | 1,025 chunks |
| Promoted | 1,076 chunks |
| Useful | 511 chunks |
| Wasted | 419 chunks |
| Late | 0 |
| Evicted for prefetch | 1,025 chunks |
| Eviction regret | 576 chunks |
| Source miss | 15 chunks |
| Tracked gauge average / p50 / p90 | 54.32 / 64 / 64 |
| Admission-to-ready mean | 209.55 s |
| Admission-to-ready p50 / p75 / p90 | 6.37 / 404.83 / 740.14 s |

Promotions can exceed in-window submissions because work submitted during warmup may complete after the profiling boundary. For terminal outcomes observed within profiling, useful was 54.95% and wasted 45.05%. More importantly, every new submission displaced an ordinary CPU entry and 56.20% of those victims were later demanded. Eviction regret was 1.13 times the useful count.

The ready-delay distribution is the strongest policy evidence. A median of 6.37 seconds can plausibly exploit queue lead time, but the p75/p90 values of roughly 405/740 seconds cannot represent useful admission-to-first-demand horizons for this workload. They show old work waiting behind the globally saturated 64-chunk footprint and being processed much later.

Treatment and control had nearly identical scheduler pressure:

| Gauge | Control | Treatment |
|---|---:|---:|
| Mean running requests | 26.79 | 26.74 |
| Mean waiting requests | 4.80 | 4.73 |
| Waiting p50 / p90 | 4 / 11 | 4 / 11 |

Reactive tier lookup was also similar: 4.080 seconds mean control versus 4.181 seconds treatment. This pair does not show that prefetch reduced exposed reactive lookup stalls.

## Proof that the corrected implementation was absent

The v2-specific lifecycle labels `cancelled_at_first_demand`, `expired_before_submit`, and `submitted_after_demand` do not exist in the treatment export. Combined with the identical old digest, this conclusively means the newly implemented cutoff/order path was not present.

The local BenchFlow experiment still points at:

- image: `quay.io/rh-ee-aperdomo/vllm:v0.27.0-clean-prefetch-v1`;
- revision tag: `clean-prefetch-v1`.

## Decision

Do not use this pair to accept or reject the surgical v2 correction. Do use it to close further tuning of the old FIFO implementation: the stale-ready tail and unfavorable eviction exchange reproduced and became worse.

Before rerunning:

1. point both cells to the newly built corrected image, preferably by immutable digest;
2. change the MLflow revision/comparison tags so the run is not confused with v1;
3. verify the rendered manifest before launch;
4. require the treatment export to contain the three new lifecycle labels;
5. require `submitted_after_demand=0`;
6. then repeat the same concurrency-64 matrix without changing N or adding `max_num_seqs`.

Only after the corrected mechanism removes post-demand submissions and collapses the hundreds-of-seconds ready tail should an N sweep be considered.

## Related evidence

- [[2026-08-22 - Clean-prefetch v1 AgentX concurrency 64 comparison|First identical-digest v1 concurrency-64 pair and surgical-fix rationale]]
- [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|Independent research audit and redirection]]