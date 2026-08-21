---
title: "V3 JIT demand-safe AgentX comparison"
date: "2026-08-21"
type: "experiment-report"
experiment: "ABC Version 3 JIT demand-safe speculative prefetch"
status: "mechanism-accepted-performance-inconclusive"
control_run: "19c4d1be0d0b4bbeb6358da05c32721f"
treatment_run: "5be11650e5a34043a3940c2e57dded74"
---

# V3 JIT demand-safe AgentX comparison

## Executive summary

The v6 treatment **successfully exercised the redesigned speculative-prefetch mechanism**: it promoted 1,024 chunks, classified 512 useful and 448 wasted, left 64 pending at the measurement boundary, preserved its physical reserve, and recorded neither reserve borrowing nor retention-lease reclamation. Useful yield improved from 4.71% in the v5 failure to 50.0% here. The mechanism should be retained for the next experiment.

The apparent performance result is large—treatment completed 151 requests versus 58 and had 39% lower mean TTFT—but **must not be accepted as a causal prefetch gain**. During matched warmup, before treatment recorded any speculative promotion, treatment was already 59% faster and its mean TTFT was 89% lower. The cells ran simultaneously on different nodes, neither profiling phase drained before timeout, and GPU/DCGM telemetry is absent. The correct verdict is:

> **Valid mechanism success; performance invalid/inconclusive for causal benefit.**

## Runs and fingerprint

- Control: [19c4d1be0d0b4bbeb6358da05c32721f](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/19c4d1be0d0b4bbeb6358da05c32721f?workspace=benchflow)
- Treatment: [5be11650e5a34043a3940c2e57dded74](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/5be11650e5a34043a3940c2e57dded74?workspace=benchflow)

Both used image `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v6` at digest `sha256:2d746cfe91ea5c47ffc635f2995d5696c066e94c0dc33185db16ccee8ad19033`, Nemotron Ultra 253B FP8, TP8, one H100 replica, `gpu_memory_utilization=0.8`, `max_model_len=131072`, `max_num_seqs=1`, AgentX concurrency 64 for 1,800 seconds, and seed `20260707`. Both used a 256 GiB CPU tier and a filesystem secondary tier with 64 read and 64 write threads.

The control ran on `diadochos-hqxzk-gpu-h100-gjfjh`; treatment ran concurrently on `diadochos-hqxzk-gpu-h100-mt46x`. There were no pod restarts or model-server request errors.

## Headline profiling metrics

These are **observed differences, not accepted causal effects**.

| Metric | Control | Treatment | Observed delta |
|---|---:|---:|---:|
| Completed requests | 58 | 151 | +160.3% |
| Cancelled at timeout | 51 | 51 | equal |
| Completed sessions | 2 | 7 | +250.0% |
| Request throughput | 0.03152 req/s | 0.08207 req/s | +160.3% |
| Output throughput | 27.954 tok/s | 59.176 tok/s | +111.7% |
| Mean TTFT | 727.761 s | 443.341 s | -39.1% |
| p95 TTFT | 1506.826 s | 751.417 s | -50.1% |
| Mean end-to-end latency | 756.486 s | 454.648 s | -39.9% |
| p95 end-to-end latency | 1521.730 s | 756.778 s | -50.3% |
| Mean inter-token latency | 31.465 ms | 15.449 ms | -50.9% |
| Average input | 65,599.5 tokens | 67,010.1 tokens | +2.2% |
| Average output | 886.8 tokens | 721.1 tokens | -18.7% |

Figure 1 compares like-for-like latency measures on a common seconds scale. Treatment is lower for every observed summary, but this is not a causal estimate because the same node divergence appears before prefetch runs.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Observed profiling latency",
  "width": 700,
  "height": 320,
  "data": {
    "values": [
      {"metric": "Mean TTFT", "cell": "Control", "seconds": 727.761},
      {"metric": "Mean TTFT", "cell": "Treatment", "seconds": 443.341},
      {"metric": "p95 TTFT", "cell": "Control", "seconds": 1506.826},
      {"metric": "p95 TTFT", "cell": "Treatment", "seconds": 751.417},
      {"metric": "Mean E2E", "cell": "Control", "seconds": 756.486},
      {"metric": "Mean E2E", "cell": "Treatment", "seconds": 454.648},
      {"metric": "p95 E2E", "cell": "Control", "seconds": 1521.730},
      {"metric": "p95 E2E", "cell": "Treatment", "seconds": 756.778}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "metric", "type": "nominal", "title": "Latency statistic", "sort": ["Mean TTFT", "p95 TTFT", "Mean E2E", "p95 E2E"]},
    "xOffset": {"field": "cell", "type": "nominal"},
    "y": {"field": "seconds", "type": "quantitative", "title": "Latency (s)", "scale": {"zero": true}},
    "color": {"field": "cell", "type": "nominal", "title": "Configuration", "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "metric", "type": "nominal", "title": "Metric"},
      {"field": "cell", "type": "nominal", "title": "Configuration"},
      {"field": "seconds", "type": "quantitative", "title": "Latency (s)", "format": ".3f"}
    ]
  }
}
```

Figure 2 isolates request throughput so incompatible units are not mixed on one axis. The observed treatment throughput is 2.60 times control, but the pair is confounded.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 2 — Observed profiling request throughput",
  "width": 500,
  "height": 300,
  "data": {
    "values": [
      {"cell": "Control", "requests_per_second": 0.0315217},
      {"cell": "Treatment", "requests_per_second": 0.0820651}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "cell", "type": "nominal", "title": "Configuration", "sort": ["Control", "Treatment"]},
    "y": {"field": "requests_per_second", "type": "quantitative", "title": "Request throughput (requests/s)", "scale": {"zero": true}},
    "color": {"field": "cell", "type": "nominal", "title": "Configuration", "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "cell", "type": "nominal", "title": "Configuration"},
      {"field": "requests_per_second", "type": "quantitative", "title": "Requests/s", "format": ".5f"}
    ]
  }
}
```

Figure 3 makes the timeout problem visible. Both cells cancelled 51 requests, so the benchmark did not produce two cleanly drained samples even though the model server reported zero request errors.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 3 — Profiling request outcomes",
  "width": 560,
  "height": 300,
  "data": {
    "values": [
      {"cell": "Control", "outcome": "Completed", "requests": 58},
      {"cell": "Control", "outcome": "Cancelled at timeout", "requests": 51},
      {"cell": "Treatment", "outcome": "Completed", "requests": 151},
      {"cell": "Treatment", "outcome": "Cancelled at timeout", "requests": 51}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "cell", "type": "nominal", "title": "Configuration", "sort": ["Control", "Treatment"]},
    "xOffset": {"field": "outcome", "type": "nominal"},
    "y": {"field": "requests", "type": "quantitative", "title": "Requests (count)", "scale": {"zero": true}},
    "color": {"field": "outcome", "type": "nominal", "title": "Outcome", "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "cell", "type": "nominal", "title": "Configuration"},
      {"field": "outcome", "type": "nominal", "title": "Outcome"},
      {"field": "requests", "type": "quantitative", "title": "Requests"}
    ]
  }
}
```

Both profiling phases timed out after the grace period. Control sent 109 requests, completed 58, and cancelled 51; treatment sent 202, completed 151, and cancelled 51. The benchmark also cleaned up stuck concurrency credits. Thus error count zero does not mean the samples completed cleanly.

## Warmup falsifies a simple causal reading

Warmup is a matched-work diagnostic: both cells processed 66 requests and emitted exactly 66 output tokens.

| Warmup metric | Control | Treatment | Observed delta |
|---|---:|---:|---:|
| Total input tokens | 4,623,854 | 4,623,891 | +0.0008% |
| Duration | 2021.786 s | 831.331 s | -58.9% |
| Throughput | 0.03264 req/s | 0.07939 req/s | +143.2% |
| Mean TTFT | 264.589 s | 29.516 s | -88.8% |
| p95 TTFT | 828.563 s | 114.989 s | -86.1% |
| GPU→CPU store bytes | 595.526 GB | 595.524 GB | effectively equal |
| Mean async lookup delay | 180.879 s | 0.857 s | -99.5% |

Figure 4 shows the matched-work warmup divergence. It occurs while treatment has zero speculative submissions or promotions, so it rules out speculative movement as the sole cause of the later performance difference.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 4 — Matched-work warmup duration and latency",
  "width": 650,
  "height": 320,
  "data": {
    "values": [
      {"metric": "Phase duration", "cell": "Control", "seconds": 2021.786},
      {"metric": "Phase duration", "cell": "Treatment", "seconds": 831.331},
      {"metric": "Mean TTFT", "cell": "Control", "seconds": 264.589},
      {"metric": "Mean TTFT", "cell": "Treatment", "seconds": 29.516},
      {"metric": "p95 TTFT", "cell": "Control", "seconds": 828.563},
      {"metric": "p95 TTFT", "cell": "Treatment", "seconds": 114.989}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "metric", "type": "nominal", "title": "Warmup statistic", "sort": ["Phase duration", "Mean TTFT", "p95 TTFT"]},
    "xOffset": {"field": "cell", "type": "nominal"},
    "y": {"field": "seconds", "type": "quantitative", "title": "Duration or latency (s)", "scale": {"zero": true}},
    "color": {"field": "cell", "type": "nominal", "title": "Configuration", "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "metric", "type": "nominal", "title": "Metric"},
      {"field": "cell", "type": "nominal", "title": "Configuration"},
      {"field": "seconds", "type": "quantitative", "title": "Seconds", "format": ".3f"}
    ]
  }
}
```

Treatment warmup recorded **zero submitted, promoted, useful, or wasted speculative chunks**. Its policy outcomes were 22 absent bundles, 41 late bundles, 2,560 gate-rejected keys, and 1,087 primary-redundant keys. Because the large latency and duration difference predates the tested speculative data movement, speculative prefetch alone cannot explain the profiling difference.

The 211-fold async-lookup-delay difference is an unresolved systems confound. Missing DCGM GPU utilization, clocks, memory, and PCIe series prevent testing whether node runtime state or GPU behavior caused it. Concurrent execution avoids time drift but does not remove node effects.

## Performance diagnostics

Figure 5 compares the queue and tiering delays that most directly determine TTFT in this saturated, one-sequence test. Treatment shows lower queue percentiles and lower mean async lookup latency, but warmup proves that at least part of this difference is node- or runtime-dependent rather than a demonstrated prefetch effect.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 5 — Profiling queue and tiering delay",
  "width": 650,
  "height": 320,
  "data": {
    "values": [
      {"metric": "Queue p50", "cell": "Control", "seconds": 449.2},
      {"metric": "Queue p50", "cell": "Treatment", "seconds": 277.3},
      {"metric": "Queue p99", "cell": "Control", "seconds": 711.9},
      {"metric": "Queue p99", "cell": "Treatment", "seconds": 491.9},
      {"metric": "Async lookup mean", "cell": "Control", "seconds": 422.178},
      {"metric": "Async lookup mean", "cell": "Treatment", "seconds": 214.976}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "metric", "type": "nominal", "title": "Delay statistic", "sort": ["Queue p50", "Queue p99", "Async lookup mean"]},
    "xOffset": {"field": "cell", "type": "nominal"},
    "y": {"field": "seconds", "type": "quantitative", "title": "Delay (s)", "scale": {"zero": true}},
    "color": {"field": "cell", "type": "nominal", "title": "Configuration", "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "metric", "type": "nominal", "title": "Metric"},
      {"field": "cell", "type": "nominal", "title": "Configuration"},
      {"field": "seconds", "type": "quantitative", "title": "Delay (s)", "format": ".3f"}
    ]
  }
}
```

Figure 6 shows the external-KV reuse indicators. Treatment sourced a larger share of prompt tokens externally and had a higher external prefix hit rate. This is consistent with more external reuse but is not sufficient to attribute the latency difference to speculative prefetch.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 6 — External KV reuse indicators",
  "width": 560,
  "height": 300,
  "data": {
    "values": [
      {"metric": "External prompt-token share", "cell": "Control", "percent": 24.3},
      {"metric": "External prompt-token share", "cell": "Treatment", "percent": 40.6},
      {"metric": "External prefix hit rate", "cell": "Control", "percent": 33.1},
      {"metric": "External prefix hit rate", "cell": "Treatment", "percent": 51.8}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "metric", "type": "nominal", "title": "Reuse metric", "sort": ["External prompt-token share", "External prefix hit rate"]},
    "xOffset": {"field": "cell", "type": "nominal"},
    "y": {"field": "percent", "type": "quantitative", "title": "Share or hit rate (%)", "scale": {"zero": true, "domain": [0, 100]}},
    "color": {"field": "cell", "type": "nominal", "title": "Configuration", "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "metric", "type": "nominal", "title": "Metric"},
      {"field": "cell", "type": "nominal", "title": "Configuration"},
      {"field": "percent", "type": "quantitative", "title": "Percent", "format": ".1f"}
    ]
  }
}
```

## Mechanism result

Treatment profiling produced an exact terminal accounting snapshot:

| Outcome | Chunks | Share |
|---|---:|---:|
| Considered | 199,600 | 100% of considered |
| Primary redundant | 155,583 | 77.95% of considered |
| Bundle trimmed | 41,416 | 20.75% of considered |
| Gate rejected | 641 | 0.32% of considered |
| Secondary absent | 10 | 0.005% of considered |
| Submitted / attempted / promoted | 1,024 | 0.513% of considered |
| Useful | 512 | 50.0% of promoted |
| Wasted | 448 | 43.75% of promoted |
| Pending at measurement end | 64 | 6.25% of promoted |
| Evicted before demand | 384 | 37.5% of promoted; subset of wasted |

The promoted partition balances: $512 + 448 + 64 = 1{,}024$. There were no load failures, allocation refusals, skipped chunks, reserve borrowing, or lease reclamation. All promoted bundles were 64 chunks. Bundle state ended at 16 ready, 10 late, and 10 absent. Mean active-owner gauge was approximately 0.986.

Figure 7 compares terminal promotion quality with the failed v5 lease run. The useful share rises from 4.71% to 50.0%, while waste falls from 93.29% to 43.75%.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 7 — Promoted-chunk outcomes in v5 and v6",
  "width": 650,
  "height": 240,
  "data": {
    "values": [
      {"version": "v5 failed lease run", "outcome": "Useful", "percent": 4.71, "order": 1},
      {"version": "v5 failed lease run", "outcome": "Wasted", "percent": 93.29, "order": 2},
      {"version": "v5 failed lease run", "outcome": "Other or pending", "percent": 2.00, "order": 3},
      {"version": "v6 JIT run", "outcome": "Useful", "percent": 50.00, "order": 1},
      {"version": "v6 JIT run", "outcome": "Wasted", "percent": 43.75, "order": 2},
      {"version": "v6 JIT run", "outcome": "Other or pending", "percent": 6.25, "order": 3}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "y": {"field": "version", "type": "nominal", "title": "Implementation", "sort": ["v5 failed lease run", "v6 JIT run"]},
    "x": {"field": "percent", "type": "quantitative", "title": "Share of promoted chunks (%)", "stack": "zero", "scale": {"zero": true, "domain": [0, 100]}},
    "color": {"field": "outcome", "type": "nominal", "title": "Terminal outcome", "scale": {"scheme": "category10"}},
    "order": {"field": "order", "type": "ordinal"},
    "tooltip": [
      {"field": "version", "type": "nominal", "title": "Implementation"},
      {"field": "outcome", "type": "nominal", "title": "Outcome"},
      {"field": "percent", "type": "quantitative", "title": "Promoted chunks (%)", "format": ".2f"}
    ]
  }
}
```

The reserve was configured at 512 blocks. Free reserve never fell below 128; speculative occupancy peaked at 384 blocks; leased occupancy peaked at 64. Therefore this run validates reserve preservation and owner/lease behavior, but does not locate the minimum safe reserve.

Compared with the v5 failure—4.71% useful, 93.29% wasted, and lease reclamation averaging 3.888 blocks/s—the v6 mechanism is a material retention correction. This is sufficient evidence to continue with v6 rather than revert to broad multi-owner admission prefetch.

## Supporting profiling telemetry

| Metric | Control | Treatment |
|---|---:|---:|
| Mean waiting requests | 25.27 | 29.51 |
| Mean queue p50 | 449.2 s | 277.3 s |
| Mean queue p99 | 711.9 s | 491.9 s |
| External prompt-token share | 24.3% | 40.6% |
| External prefix hit rate | 33.1% | 51.8% |
| Exact-histogram mean async lookup | 422.178 s | 214.976 s |
| GPU→CPU transfer | 645.824 GB | 738.718 GB |
| CPU→GPU transfer | 252.801 GB | 935.676 GB |

Raw transfer totals are not normalized: treatment completed 2.6 times as many requests. They should not be interpreted as per-request efficiency.

Prometheus counters/gauges were sampled at approximately 15-second cadence. Dashboard rate series use five-minute windows where applicable, so short-lived transitions are smoothed and boundary values can include work immediately outside a phase. Exact counters and histogram deltas are preferred for terminal accounting.

## Validity assessment

### What is valid

- Same workload profile, model, seed, image digest, TP degree, memory setting, CPU-tier size, and filesystem configuration.
- Treatment produced real speculative submissions and promotions.
- Promoted terminal accounting balances exactly at the measurement boundary.
- Useful yield and retention improved sharply versus v5.
- No observed demand reserve borrowing, lease breaking, allocation refusal, or load failure.
- No server errors or restarts.

### What prevents a performance claim

1. **Pre-treatment warmup divergence:** treatment was much faster before any speculative promotion.
2. **Different nodes:** the pair was concurrent but not a within-node cross-over.
3. **Incomplete profiling phases:** both ended with 51 cancellations and stuck-credit cleanup.
4. **One pair only:** there is no estimate of run-to-run or node-to-node variance.
5. **Missing GPU telemetry:** no DCGM utilization, clocks, memory, or PCIe evidence.
6. **Storage isolation uncertainty:** aggregate NVMe/Weka telemetry may not cleanly isolate each model node/device.
7. **Mechanism-test scheduler setting:** `max_num_seqs=1` is intentional for attribution and pressure but is not representative of a tuned production deployment.

## Decision

**Keep the v6 direction. Do not make a 180-degree design change.** The implementation repaired the specific v5 pathology—promotions expiring before demand—and did so with bounded ownership, demand priority, and exact safety accounting. The performance hypothesis remains open, not supported or falsified, because this pair is confounded.

Do not tune many knobs from this pair. In particular, the 512-block reserve was non-binding, so no performance conclusion can be attached to that value. A 64-block reserve is a reasonable next mechanism probe because one promoted bundle is 64 chunks, but it must be tested rather than assumed.

## Next experiment

1. Run a randomized node cross-over: control and treatment swap `gjfjh` and `mt46x`.
2. Collect at least three completed pairs, balancing cell order and node assignment.
3. Extend grace time or reduce offered pressure until every admitted profiling request reaches a terminal benchmark result.
4. Preserve `max_num_seqs=1` for the immediate mechanism/causality test; after confirmation, repeat at a realistic scheduler concurrency.
5. Capture per-node DCGM utilization, SM/memory clocks, GPU memory, PCIe/NVLink traffic, CPU pressure, and device-specific NVMe latency/throughput.
6. Keep the current JIT, single-owner, demand-idle policy fixed. Test reserve 64 against 512 only after a clean baseline pair; avoid a broad parameter sweep.
7. Acceptance gate: terminal accounting must balance, promotions must be nonzero, no load failures may occur, and a performance effect must repeat across node swaps without a pre-promotion warmup divergence.

## Conclusion

The experiment makes sense as a **mechanism validation** and supports continuing Version3. It does not make sense as evidence of a 39–50% latency improvement or 160% throughput gain. The next work is experimental control and replication, not another speculative-prefetch concept.