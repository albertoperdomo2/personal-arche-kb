---
title: "ABC Phase 1 — 256 GiB CPU offload prefetch validation"
date: "2026-08-14"
type: "research-report"
experiment: "Activity-Based KV Cache Tier Placement"
phase: "1 — naive proactive prefetching"
status: "invalid-inconclusive"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "unknown"
image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-p1"
vllm_version: "0.27.0 with Phase 1 prefetch overlay"
tensor_parallelism: 8
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: "unknown"
concurrency: 32
cpu_bytes: 274877906944
offload_spec: "TieringOffloadingSpec for CPU-offload runs"
secondary_tier: "none configured"
secondary_tier_threads: "N/A"
dev_shm: "300Gi"
workload: "semianalysisai/cc-traces-weka-062126"
random_seed: 20260707
duration_seconds: 1800
cache_cleaning_state: "not recorded"
configuration:
  no_offload:
    run_id: "c2c2e87883324898995c3ca1639db3b1"
  cpu_256g_control:
    run_id: "5f57165d7d464cee8514645215c526c7"
    prefetch_chunks: 0
  cpu_256g_prefetch:
    run_id: "d5bace21821648ec96bcb7f6efdb3077"
    prefetch_chunks: 100
---

# ABC Phase 1 — 256 GiB CPU offload prefetch validation

## Executive summary

This batch asked whether the Phase 1 fixed-$N$ hook reduced request-visible latency by proactively promoting upcoming KV chunks. It compared a no-offload reference, a 256 GiB CPU-offload control, and a nominally identical 256 GiB CPU-offload run with `prefetch_chunks=100`. All runs used one H100 node with TP=8, the same patched vLLM image, model, Weka workload, seed, concurrency 32, 1,800-second profile, GPU-memory utilization 0.8, and 131,072-token maximum context.

The hook was enabled and invoked, but **prefetch did not perform any promotion**. The prefetch run averaged 50.762 attempted chunks/s and exactly 50.762 skipped chunks/s across 154 native 15-second samples; promoted, useful, and wasted series were absent. The manifest configured `TieringOffloadingSpec` with a 256 GiB CPU primary tier but no `secondary_tiers`. This toy promotes only secondary→CPU, so every upcoming key was classified as absent from a secondary tier and skipped.

# Invalid / inconclusive

This batch is invalid for deciding whether prefetch improves latency or for tuning $N$. The mechanism under test never moved a block, there was only one nominal $N>0$ cell, and there were no repeated runs. The observed latency differences cannot be attributed to prefetch.

## Main takeaways

- **Measured:** the prefetch image and configuration loaded successfully. The server logged `Phase 1 naive prefetch enabled: prefetch_chunks=100, prefetch_track_capacity=8192`.
- **Measured:** attempted and skipped rates were identical at every one of 154 samples (15-second cadence), averaging 50.762 chunks/s and peaking at 92.285 chunks/s. Promoted/useful/wasted were absent, so the promotion rate and useful-hit sample count were zero.
- **Measured:** the 256 GiB control and prefetch profiles completed the same 838 requests. Prefetch-profile request latency was slightly lower, but TTFT was mixed: mean TTFT was 2.28% worse and P95 was 4.41% worse.
- **Inference:** the missing secondary tier fully explains the all-skipped accounting. In the implementation, `prefetch()` scans `secondary_tiers`; when the key is in none, it increments `PREFETCH_SKIPPED`.
- **Conclusion:** this is a configuration/plumbing rejection, not a negative result for proactive prefetching and not evidence that $N=100$ is good or bad.
- **Safety signals:** no allocation-failure or preemption series appeared. CPU-cache occupancy remained tiny (average 0.394%, maximum 1.5625%), consistent with no speculative promotions taking place.

## Headline metrics

The no-offload run is the system reference. For mechanism isolation, deltas are against the 256 GiB CPU-offload control because it differs from the prefetch profile only by the nominal `prefetch_chunks=100` setting.

| Configuration | Run ID | Completed requests | Request rate (req/s) | Output rate (tok/s) | Request latency mean / P50 / P95 (ms) | TTFT mean / P50 / P95 (ms) | ITL mean / P95 (ms) | Errors / cancelled | Delta vs CPU control |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| No offload reference | `c2c2e878…` | 797 | 0.4332 | 274.15 | 23,490.67 / 12,043.73 / 67,540.73 | 1,834.22 / 563.64 / 8,547.78 | 34.67 / 76.98 | 0 / no | N/A — different mechanism |
| 256 GiB CPU offload, $N=0$ | `5f57165d…` | 838 | 0.4554 | 314.72 | 21,432.24 / 9,889.84 / 55,995.61 | 1,291.28 / 551.16 / 5,793.48 | 31.13 / 55.52 | 0 / no | Declared mechanism baseline |
| 256 GiB CPU offload, nominal $N=100$ | `d5bace21…` | 838 | 0.4554 | 314.72 | 21,244.82 / 9,696.51 / 55,363.79 | 1,320.71 / 568.02 / 6,049.06 | 30.34 / 53.28 | 0 / no | E2E mean −0.87%; P50 −1.95%; P95 −1.13%; TTFT mean +2.28%; TTFT P95 +4.41% |

The two 256 GiB profiles contain the same 838 `conversation_id / turn / source-kind / depth` keys. A paired request-level check gives mean E2E difference −187.4 ms (normal-approximation 95% CI −345.4 to −29.4 ms), TTFT +29.4 ms (−55.8 to +114.6 ms), and ITL −0.789 ms (−1.622 to +0.043 ms). Requests inside a concurrent trajectory are correlated and this is only one run pair, so these intervals are descriptive—not acceptance evidence.

Figure 1 shows the metric-level deltas for the nominal prefetch profile. Lower latency is better; the mixed TTFT direction and lack of actual promotions make attribution invalid.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Prefetch-profile delta versus the 256 GiB CPU-offload control",
  "width": 640,
  "height": 300,
  "background": "white",
  "data": {
    "values": [
      {
        "metric": "Request latency mean",
        "delta_pct": -0.874
      },
      {
        "metric": "Request latency P50",
        "delta_pct": -1.955
      },
      {
        "metric": "Request latency P90",
        "delta_pct": -1.523
      },
      {
        "metric": "Request latency P95",
        "delta_pct": -1.128
      },
      {
        "metric": "TTFT mean",
        "delta_pct": 2.279
      },
      {
        "metric": "TTFT P50",
        "delta_pct": 3.059
      },
      {
        "metric": "TTFT P90",
        "delta_pct": -2.872
      },
      {
        "metric": "TTFT P95",
        "delta_pct": 4.412
      },
      {
        "metric": "ITL mean",
        "delta_pct": -2.538
      },
      {
        "metric": "ITL P95",
        "delta_pct": -4.042
      }
    ]
  },
  "layer": [
    {
      "mark": {
        "type": "bar",
        "color": "#1f77b4"
      },
      "encoding": {
        "y": {
          "field": "metric",
          "type": "nominal",
          "title": "Outcome metric",
          "sort": null
        },
        "x": {
          "field": "delta_pct",
          "type": "quantitative",
          "title": "Relative delta (%)"
        },
        "tooltip": [
          {
            "field": "metric",
            "type": "nominal"
          },
          {
            "field": "delta_pct",
            "type": "quantitative",
            "title": "Delta (%)",
            "format": ".3f"
          }
        ]
      }
    },
    {
      "mark": {
        "type": "rule",
        "color": "#444",
        "strokeWidth": 1
      },
      "encoding": {
        "x": {
          "datum": 0
        }
      }
    }
  ]
}
```

## Mechanism evidence

The MLflow raw artifacts provide 154 samples at the native 15-second cadence. Figure 2 plots the attempted and skipped Prometheus rates without downsampling or aggregation beyond the source query's 5-minute `rate()` definition. The two lines overlap exactly for the entire window.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 2 — Every Phase 1 prefetch attempt was skipped",
  "width": 700,
  "height": 320,
  "background": "white",
  "data": {
    "values": [
      {
        "elapsed_s": 135,
        "outcome": "Attempted",
        "rate_chunks_s": 1.045288888888889
      },
      {
        "elapsed_s": 135,
        "outcome": "Skipped",
        "rate_chunks_s": 1.045288888888889
      },
      {
        "elapsed_s": 150,
        "outcome": "Attempted",
        "rate_chunks_s": 1.9311333333333334
      },
      {
        "elapsed_s": 150,
        "outcome": "Skipped",
        "rate_chunks_s": 1.9311333333333334
      },
      {
        "elapsed_s": 165,
        "outcome": "Attempted",
        "rate_chunks_s": 2.0031444444444446
      },
      {
        "elapsed_s": 165,
        "outcome": "Skipped",
        "rate_chunks_s": 2.0031444444444446
      },
      {
        "elapsed_s": 180,
        "outcome": "Attempted",
        "rate_chunks_s": 2.8627333333333334
      },
      {
        "elapsed_s": 180,
        "outcome": "Skipped",
        "rate_chunks_s": 2.8627333333333334
      },
      {
        "elapsed_s": 195,
        "outcome": "Attempted",
        "rate_chunks_s": 3.5609518518518515
      },
      {
        "elapsed_s": 195,
        "outcome": "Skipped",
        "rate_chunks_s": 3.5609518518518515
      },
      {
        "elapsed_s": 210,
        "outcome": "Attempted",
        "rate_chunks_s": 3.885533333333333
      },
      {
        "elapsed_s": 210,
        "outcome": "Skipped",
        "rate_chunks_s": 3.885533333333333
      },
      {
        "elapsed_s": 225,
        "outcome": "Attempted",
        "rate_chunks_s": 4.131759259259259
      },
      {
        "elapsed_s": 225,
        "outcome": "Skipped",
        "rate_chunks_s": 4.131759259259259
      },
      {
        "elapsed_s": 240,
        "outcome": "Attempted",
        "rate_chunks_s": 4.862733333333333
      },
      {
        "elapsed_s": 240,
        "outcome": "Skipped",
        "rate_chunks_s": 4.862733333333333
      },
      {
        "elapsed_s": 255,
        "outcome": "Attempted",
        "rate_chunks_s": 5.5483311111111115
      },
      {
        "elapsed_s": 255,
        "outcome": "Skipped",
        "rate_chunks_s": 5.5483311111111115
      },
      {
        "elapsed_s": 270,
        "outcome": "Attempted",
        "rate_chunks_s": 6.253582222222223
      },
      {
        "elapsed_s": 270,
        "outcome": "Skipped",
        "rate_chunks_s": 6.253582222222223
      },
      {
        "elapsed_s": 285,
        "outcome": "Attempted",
        "rate_chunks_s": 6.93286962962963
      },
      {
        "elapsed_s": 285,
        "outcome": "Skipped",
        "rate_chunks_s": 6.93286962962963
      },
      {
        "elapsed_s": 300,
        "outcome": "Attempted",
        "rate_chunks_s": 7.255396296296297
      },
      {
        "elapsed_s": 300,
        "outcome": "Skipped",
        "rate_chunks_s": 7.255396296296297
      },
      {
        "elapsed_s": 315,
        "outcome": "Attempted",
        "rate_chunks_s": 7.542922222222223
      },
      {
        "elapsed_s": 315,
        "outcome": "Skipped",
        "rate_chunks_s": 7.542922222222223
      },
      {
        "elapsed_s": 330,
        "outcome": "Attempted",
        "rate_chunks_s": 8.237149206349208
      },
      {
        "elapsed_s": 330,
        "outcome": "Skipped",
        "rate_chunks_s": 8.237149206349208
      },
      {
        "elapsed_s": 345,
        "outcome": "Attempted",
        "rate_chunks_s": 9.270954761904761
      },
      {
        "elapsed_s": 345,
        "outcome": "Skipped",
        "rate_chunks_s": 9.270954761904761
      },
      {
        "elapsed_s": 360,
        "outcome": "Attempted",
        "rate_chunks_s": 10.339855555555555
      },
      {
        "elapsed_s": 360,
        "outcome": "Skipped",
        "rate_chunks_s": 10.339855555555555
      },
      {
        "elapsed_s": 375,
        "outcome": "Attempted",
        "rate_chunks_s": 11.048817901234568
      },
      {
        "elapsed_s": 375,
        "outcome": "Skipped",
        "rate_chunks_s": 11.048817901234568
      },
      {
        "elapsed_s": 390,
        "outcome": "Attempted",
        "rate_chunks_s": 12.222222222222223
      },
      {
        "elapsed_s": 390,
        "outcome": "Skipped",
        "rate_chunks_s": 12.222222222222223
      },
      {
        "elapsed_s": 405,
        "outcome": "Attempted",
        "rate_chunks_s": 12.592592592592593
      },
      {
        "elapsed_s": 405,
        "outcome": "Skipped",
        "rate_chunks_s": 12.592592592592593
      },
      {
        "elapsed_s": 420,
        "outcome": "Attempted",
        "rate_chunks_s": 12.962962962962964
      },
      {
        "elapsed_s": 420,
        "outcome": "Skipped",
        "rate_chunks_s": 12.962962962962964
      },
      {
        "elapsed_s": 435,
        "outcome": "Attempted",
        "rate_chunks_s": 13.703703703703704
      },
      {
        "elapsed_s": 435,
        "outcome": "Skipped",
        "rate_chunks_s": 13.703703703703704
      },
      {
        "elapsed_s": 450,
        "outcome": "Attempted",
        "rate_chunks_s": 14.074074074074074
      },
      {
        "elapsed_s": 450,
        "outcome": "Skipped",
        "rate_chunks_s": 14.074074074074074
      },
      {
        "elapsed_s": 465,
        "outcome": "Attempted",
        "rate_chunks_s": 14.074074074074074
      },
      {
        "elapsed_s": 465,
        "outcome": "Skipped",
        "rate_chunks_s": 14.074074074074074
      },
      {
        "elapsed_s": 480,
        "outcome": "Attempted",
        "rate_chunks_s": 14.444444444444445
      },
      {
        "elapsed_s": 480,
        "outcome": "Skipped",
        "rate_chunks_s": 14.444444444444445
      },
      {
        "elapsed_s": 495,
        "outcome": "Attempted",
        "rate_chunks_s": 14.814814814814815
      },
      {
        "elapsed_s": 495,
        "outcome": "Skipped",
        "rate_chunks_s": 14.814814814814815
      },
      {
        "elapsed_s": 510,
        "outcome": "Attempted",
        "rate_chunks_s": 15.185185185185185
      },
      {
        "elapsed_s": 510,
        "outcome": "Skipped",
        "rate_chunks_s": 15.185185185185185
      },
      {
        "elapsed_s": 525,
        "outcome": "Attempted",
        "rate_chunks_s": 15.925925925925927
      },
      {
        "elapsed_s": 525,
        "outcome": "Skipped",
        "rate_chunks_s": 15.925925925925927
      },
      {
        "elapsed_s": 540,
        "outcome": "Attempted",
        "rate_chunks_s": 16.296296296296298
      },
      {
        "elapsed_s": 540,
        "outcome": "Skipped",
        "rate_chunks_s": 16.296296296296298
      },
      {
        "elapsed_s": 555,
        "outcome": "Attempted",
        "rate_chunks_s": 16.666666666666668
      },
      {
        "elapsed_s": 555,
        "outcome": "Skipped",
        "rate_chunks_s": 16.666666666666668
      },
      {
        "elapsed_s": 570,
        "outcome": "Attempted",
        "rate_chunks_s": 18.14814814814815
      },
      {
        "elapsed_s": 570,
        "outcome": "Skipped",
        "rate_chunks_s": 18.14814814814815
      },
      {
        "elapsed_s": 585,
        "outcome": "Attempted",
        "rate_chunks_s": 22.096296296296295
      },
      {
        "elapsed_s": 585,
        "outcome": "Skipped",
        "rate_chunks_s": 22.096296296296295
      },
      {
        "elapsed_s": 600,
        "outcome": "Attempted",
        "rate_chunks_s": 26.185185185185183
      },
      {
        "elapsed_s": 600,
        "outcome": "Skipped",
        "rate_chunks_s": 26.185185185185183
      },
      {
        "elapsed_s": 615,
        "outcome": "Attempted",
        "rate_chunks_s": 28.488888888888887
      },
      {
        "elapsed_s": 615,
        "outcome": "Skipped",
        "rate_chunks_s": 28.488888888888887
      },
      {
        "elapsed_s": 630,
        "outcome": "Attempted",
        "rate_chunks_s": 32.02962962962963
      },
      {
        "elapsed_s": 630,
        "outcome": "Skipped",
        "rate_chunks_s": 32.02962962962963
      },
      {
        "elapsed_s": 645,
        "outcome": "Attempted",
        "rate_chunks_s": 36.35925925925926
      },
      {
        "elapsed_s": 645,
        "outcome": "Skipped",
        "rate_chunks_s": 36.35925925925926
      },
      {
        "elapsed_s": 660,
        "outcome": "Attempted",
        "rate_chunks_s": 40.65555555555556
      },
      {
        "elapsed_s": 660,
        "outcome": "Skipped",
        "rate_chunks_s": 40.65555555555556
      },
      {
        "elapsed_s": 675,
        "outcome": "Attempted",
        "rate_chunks_s": 46.11481481481482
      },
      {
        "elapsed_s": 675,
        "outcome": "Skipped",
        "rate_chunks_s": 46.11481481481482
      },
      {
        "elapsed_s": 690,
        "outcome": "Attempted",
        "rate_chunks_s": 50.36296296296297
      },
      {
        "elapsed_s": 690,
        "outcome": "Skipped",
        "rate_chunks_s": 50.36296296296297
      },
      {
        "elapsed_s": 705,
        "outcome": "Attempted",
        "rate_chunks_s": 53.46296296296296
      },
      {
        "elapsed_s": 705,
        "outcome": "Skipped",
        "rate_chunks_s": 53.46296296296296
      },
      {
        "elapsed_s": 720,
        "outcome": "Attempted",
        "rate_chunks_s": 58.05555555555556
      },
      {
        "elapsed_s": 720,
        "outcome": "Skipped",
        "rate_chunks_s": 58.05555555555556
      },
      {
        "elapsed_s": 735,
        "outcome": "Attempted",
        "rate_chunks_s": 62.70370370370371
      },
      {
        "elapsed_s": 735,
        "outcome": "Skipped",
        "rate_chunks_s": 62.70370370370371
      },
      {
        "elapsed_s": 750,
        "outcome": "Attempted",
        "rate_chunks_s": 65.43703703703704
      },
      {
        "elapsed_s": 750,
        "outcome": "Skipped",
        "rate_chunks_s": 65.43703703703704
      },
      {
        "elapsed_s": 765,
        "outcome": "Attempted",
        "rate_chunks_s": 67.52222222222223
      },
      {
        "elapsed_s": 765,
        "outcome": "Skipped",
        "rate_chunks_s": 67.52222222222223
      },
      {
        "elapsed_s": 780,
        "outcome": "Attempted",
        "rate_chunks_s": 68.63333333333333
      },
      {
        "elapsed_s": 780,
        "outcome": "Skipped",
        "rate_chunks_s": 68.63333333333333
      },
      {
        "elapsed_s": 795,
        "outcome": "Attempted",
        "rate_chunks_s": 70.97407407407408
      },
      {
        "elapsed_s": 795,
        "outcome": "Skipped",
        "rate_chunks_s": 70.97407407407408
      },
      {
        "elapsed_s": 810,
        "outcome": "Attempted",
        "rate_chunks_s": 75.6962962962963
      },
      {
        "elapsed_s": 810,
        "outcome": "Skipped",
        "rate_chunks_s": 75.6962962962963
      },
      {
        "elapsed_s": 825,
        "outcome": "Attempted",
        "rate_chunks_s": 81.48148148148148
      },
      {
        "elapsed_s": 825,
        "outcome": "Skipped",
        "rate_chunks_s": 81.48148148148148
      },
      {
        "elapsed_s": 840,
        "outcome": "Attempted",
        "rate_chunks_s": 86.4888888888889
      },
      {
        "elapsed_s": 840,
        "outcome": "Skipped",
        "rate_chunks_s": 86.4888888888889
      },
      {
        "elapsed_s": 855,
        "outcome": "Attempted",
        "rate_chunks_s": 86.78148148148148
      },
      {
        "elapsed_s": 855,
        "outcome": "Skipped",
        "rate_chunks_s": 86.78148148148148
      },
      {
        "elapsed_s": 870,
        "outcome": "Attempted",
        "rate_chunks_s": 86.43703703703704
      },
      {
        "elapsed_s": 870,
        "outcome": "Skipped",
        "rate_chunks_s": 86.43703703703704
      },
      {
        "elapsed_s": 885,
        "outcome": "Attempted",
        "rate_chunks_s": 88.64444444444445
      },
      {
        "elapsed_s": 885,
        "outcome": "Skipped",
        "rate_chunks_s": 88.64444444444445
      },
      {
        "elapsed_s": 900,
        "outcome": "Attempted",
        "rate_chunks_s": 89.82592592592593
      },
      {
        "elapsed_s": 900,
        "outcome": "Skipped",
        "rate_chunks_s": 89.82592592592593
      },
      {
        "elapsed_s": 915,
        "outcome": "Attempted",
        "rate_chunks_s": 89.53703703703704
      },
      {
        "elapsed_s": 915,
        "outcome": "Skipped",
        "rate_chunks_s": 89.53703703703704
      },
      {
        "elapsed_s": 930,
        "outcome": "Attempted",
        "rate_chunks_s": 89.01111111111112
      },
      {
        "elapsed_s": 930,
        "outcome": "Skipped",
        "rate_chunks_s": 89.01111111111112
      },
      {
        "elapsed_s": 945,
        "outcome": "Attempted",
        "rate_chunks_s": 88.11851851851853
      },
      {
        "elapsed_s": 945,
        "outcome": "Skipped",
        "rate_chunks_s": 88.11851851851853
      },
      {
        "elapsed_s": 960,
        "outcome": "Attempted",
        "rate_chunks_s": 87.8
      },
      {
        "elapsed_s": 960,
        "outcome": "Skipped",
        "rate_chunks_s": 87.8
      },
      {
        "elapsed_s": 975,
        "outcome": "Attempted",
        "rate_chunks_s": 86.8111111111111
      },
      {
        "elapsed_s": 975,
        "outcome": "Skipped",
        "rate_chunks_s": 86.8111111111111
      },
      {
        "elapsed_s": 990,
        "outcome": "Attempted",
        "rate_chunks_s": 83.93703703703704
      },
      {
        "elapsed_s": 990,
        "outcome": "Skipped",
        "rate_chunks_s": 83.93703703703704
      },
      {
        "elapsed_s": 1005,
        "outcome": "Attempted",
        "rate_chunks_s": 83.44444444444444
      },
      {
        "elapsed_s": 1005,
        "outcome": "Skipped",
        "rate_chunks_s": 83.44444444444444
      },
      {
        "elapsed_s": 1020,
        "outcome": "Attempted",
        "rate_chunks_s": 86.50740740740741
      },
      {
        "elapsed_s": 1020,
        "outcome": "Skipped",
        "rate_chunks_s": 86.50740740740741
      },
      {
        "elapsed_s": 1035,
        "outcome": "Attempted",
        "rate_chunks_s": 88.10740740740741
      },
      {
        "elapsed_s": 1035,
        "outcome": "Skipped",
        "rate_chunks_s": 88.10740740740741
      },
      {
        "elapsed_s": 1050,
        "outcome": "Attempted",
        "rate_chunks_s": 89.44074074074075
      },
      {
        "elapsed_s": 1050,
        "outcome": "Skipped",
        "rate_chunks_s": 89.44074074074075
      },
      {
        "elapsed_s": 1065,
        "outcome": "Attempted",
        "rate_chunks_s": 92.28518518518518
      },
      {
        "elapsed_s": 1065,
        "outcome": "Skipped",
        "rate_chunks_s": 92.28518518518518
      },
      {
        "elapsed_s": 1080,
        "outcome": "Attempted",
        "rate_chunks_s": 92.11111111111111
      },
      {
        "elapsed_s": 1080,
        "outcome": "Skipped",
        "rate_chunks_s": 92.11111111111111
      },
      {
        "elapsed_s": 1095,
        "outcome": "Attempted",
        "rate_chunks_s": 88.97037037037038
      },
      {
        "elapsed_s": 1095,
        "outcome": "Skipped",
        "rate_chunks_s": 88.97037037037038
      },
      {
        "elapsed_s": 1110,
        "outcome": "Attempted",
        "rate_chunks_s": 87.02592592592592
      },
      {
        "elapsed_s": 1110,
        "outcome": "Skipped",
        "rate_chunks_s": 87.02592592592592
      },
      {
        "elapsed_s": 1125,
        "outcome": "Attempted",
        "rate_chunks_s": 87.18148148148148
      },
      {
        "elapsed_s": 1125,
        "outcome": "Skipped",
        "rate_chunks_s": 87.18148148148148
      },
      {
        "elapsed_s": 1140,
        "outcome": "Attempted",
        "rate_chunks_s": 86.4962962962963
      },
      {
        "elapsed_s": 1140,
        "outcome": "Skipped",
        "rate_chunks_s": 86.4962962962963
      },
      {
        "elapsed_s": 1155,
        "outcome": "Attempted",
        "rate_chunks_s": 84.5962962962963
      },
      {
        "elapsed_s": 1155,
        "outcome": "Skipped",
        "rate_chunks_s": 84.5962962962963
      },
      {
        "elapsed_s": 1170,
        "outcome": "Attempted",
        "rate_chunks_s": 82.74814814814815
      },
      {
        "elapsed_s": 1170,
        "outcome": "Skipped",
        "rate_chunks_s": 82.74814814814815
      },
      {
        "elapsed_s": 1185,
        "outcome": "Attempted",
        "rate_chunks_s": 80.71851851851852
      },
      {
        "elapsed_s": 1185,
        "outcome": "Skipped",
        "rate_chunks_s": 80.71851851851852
      },
      {
        "elapsed_s": 1200,
        "outcome": "Attempted",
        "rate_chunks_s": 78.44444444444446
      },
      {
        "elapsed_s": 1200,
        "outcome": "Skipped",
        "rate_chunks_s": 78.44444444444446
      },
      {
        "elapsed_s": 1215,
        "outcome": "Attempted",
        "rate_chunks_s": 76.69629629629631
      },
      {
        "elapsed_s": 1215,
        "outcome": "Skipped",
        "rate_chunks_s": 76.69629629629631
      },
      {
        "elapsed_s": 1230,
        "outcome": "Attempted",
        "rate_chunks_s": 76.21851851851852
      },
      {
        "elapsed_s": 1230,
        "outcome": "Skipped",
        "rate_chunks_s": 76.21851851851852
      },
      {
        "elapsed_s": 1245,
        "outcome": "Attempted",
        "rate_chunks_s": 77.46666666666667
      },
      {
        "elapsed_s": 1245,
        "outcome": "Skipped",
        "rate_chunks_s": 77.46666666666667
      },
      {
        "elapsed_s": 1260,
        "outcome": "Attempted",
        "rate_chunks_s": 79.1037037037037
      },
      {
        "elapsed_s": 1260,
        "outcome": "Skipped",
        "rate_chunks_s": 79.1037037037037
      },
      {
        "elapsed_s": 1275,
        "outcome": "Attempted",
        "rate_chunks_s": 77.54814814814816
      },
      {
        "elapsed_s": 1275,
        "outcome": "Skipped",
        "rate_chunks_s": 77.54814814814816
      },
      {
        "elapsed_s": 1290,
        "outcome": "Attempted",
        "rate_chunks_s": 73.82222222222222
      },
      {
        "elapsed_s": 1290,
        "outcome": "Skipped",
        "rate_chunks_s": 73.82222222222222
      },
      {
        "elapsed_s": 1305,
        "outcome": "Attempted",
        "rate_chunks_s": 73.14814814814815
      },
      {
        "elapsed_s": 1305,
        "outcome": "Skipped",
        "rate_chunks_s": 73.14814814814815
      },
      {
        "elapsed_s": 1320,
        "outcome": "Attempted",
        "rate_chunks_s": 73.7
      },
      {
        "elapsed_s": 1320,
        "outcome": "Skipped",
        "rate_chunks_s": 73.7
      },
      {
        "elapsed_s": 1335,
        "outcome": "Attempted",
        "rate_chunks_s": 70.81851851851852
      },
      {
        "elapsed_s": 1335,
        "outcome": "Skipped",
        "rate_chunks_s": 70.81851851851852
      },
      {
        "elapsed_s": 1350,
        "outcome": "Attempted",
        "rate_chunks_s": 68.8962962962963
      },
      {
        "elapsed_s": 1350,
        "outcome": "Skipped",
        "rate_chunks_s": 68.8962962962963
      },
      {
        "elapsed_s": 1365,
        "outcome": "Attempted",
        "rate_chunks_s": 68.23703703703703
      },
      {
        "elapsed_s": 1365,
        "outcome": "Skipped",
        "rate_chunks_s": 68.23703703703703
      },
      {
        "elapsed_s": 1380,
        "outcome": "Attempted",
        "rate_chunks_s": 66.2
      },
      {
        "elapsed_s": 1380,
        "outcome": "Skipped",
        "rate_chunks_s": 66.2
      },
      {
        "elapsed_s": 1395,
        "outcome": "Attempted",
        "rate_chunks_s": 65.11481481481482
      },
      {
        "elapsed_s": 1395,
        "outcome": "Skipped",
        "rate_chunks_s": 65.11481481481482
      },
      {
        "elapsed_s": 1410,
        "outcome": "Attempted",
        "rate_chunks_s": 63.829629629629636
      },
      {
        "elapsed_s": 1410,
        "outcome": "Skipped",
        "rate_chunks_s": 63.829629629629636
      },
      {
        "elapsed_s": 1425,
        "outcome": "Attempted",
        "rate_chunks_s": 61.611111111111114
      },
      {
        "elapsed_s": 1425,
        "outcome": "Skipped",
        "rate_chunks_s": 61.611111111111114
      },
      {
        "elapsed_s": 1440,
        "outcome": "Attempted",
        "rate_chunks_s": 59.848148148148155
      },
      {
        "elapsed_s": 1440,
        "outcome": "Skipped",
        "rate_chunks_s": 59.848148148148155
      },
      {
        "elapsed_s": 1455,
        "outcome": "Attempted",
        "rate_chunks_s": 59.318518518518516
      },
      {
        "elapsed_s": 1455,
        "outcome": "Skipped",
        "rate_chunks_s": 59.318518518518516
      },
      {
        "elapsed_s": 1470,
        "outcome": "Attempted",
        "rate_chunks_s": 58.88148148148149
      },
      {
        "elapsed_s": 1470,
        "outcome": "Skipped",
        "rate_chunks_s": 58.88148148148149
      },
      {
        "elapsed_s": 1485,
        "outcome": "Attempted",
        "rate_chunks_s": 57.144444444444446
      },
      {
        "elapsed_s": 1485,
        "outcome": "Skipped",
        "rate_chunks_s": 57.144444444444446
      },
      {
        "elapsed_s": 1500,
        "outcome": "Attempted",
        "rate_chunks_s": 56.22222222222222
      },
      {
        "elapsed_s": 1500,
        "outcome": "Skipped",
        "rate_chunks_s": 56.22222222222222
      },
      {
        "elapsed_s": 1515,
        "outcome": "Attempted",
        "rate_chunks_s": 54.31111111111111
      },
      {
        "elapsed_s": 1515,
        "outcome": "Skipped",
        "rate_chunks_s": 54.31111111111111
      },
      {
        "elapsed_s": 1530,
        "outcome": "Attempted",
        "rate_chunks_s": 52.522222222222226
      },
      {
        "elapsed_s": 1530,
        "outcome": "Skipped",
        "rate_chunks_s": 52.522222222222226
      },
      {
        "elapsed_s": 1545,
        "outcome": "Attempted",
        "rate_chunks_s": 53.52592592592593
      },
      {
        "elapsed_s": 1545,
        "outcome": "Skipped",
        "rate_chunks_s": 53.52592592592593
      },
      {
        "elapsed_s": 1560,
        "outcome": "Attempted",
        "rate_chunks_s": 54.488888888888894
      },
      {
        "elapsed_s": 1560,
        "outcome": "Skipped",
        "rate_chunks_s": 54.488888888888894
      },
      {
        "elapsed_s": 1575,
        "outcome": "Attempted",
        "rate_chunks_s": 55.529629629629625
      },
      {
        "elapsed_s": 1575,
        "outcome": "Skipped",
        "rate_chunks_s": 55.529629629629625
      },
      {
        "elapsed_s": 1590,
        "outcome": "Attempted",
        "rate_chunks_s": 56.385185185185186
      },
      {
        "elapsed_s": 1590,
        "outcome": "Skipped",
        "rate_chunks_s": 56.385185185185186
      },
      {
        "elapsed_s": 1605,
        "outcome": "Attempted",
        "rate_chunks_s": 55.70370370370371
      },
      {
        "elapsed_s": 1605,
        "outcome": "Skipped",
        "rate_chunks_s": 55.70370370370371
      },
      {
        "elapsed_s": 1620,
        "outcome": "Attempted",
        "rate_chunks_s": 54.37407407407407
      },
      {
        "elapsed_s": 1620,
        "outcome": "Skipped",
        "rate_chunks_s": 54.37407407407407
      },
      {
        "elapsed_s": 1635,
        "outcome": "Attempted",
        "rate_chunks_s": 54.300000000000004
      },
      {
        "elapsed_s": 1635,
        "outcome": "Skipped",
        "rate_chunks_s": 54.300000000000004
      },
      {
        "elapsed_s": 1650,
        "outcome": "Attempted",
        "rate_chunks_s": 54.91481481481482
      },
      {
        "elapsed_s": 1650,
        "outcome": "Skipped",
        "rate_chunks_s": 54.91481481481482
      },
      {
        "elapsed_s": 1665,
        "outcome": "Attempted",
        "rate_chunks_s": 54.14074074074074
      },
      {
        "elapsed_s": 1665,
        "outcome": "Skipped",
        "rate_chunks_s": 54.14074074074074
      },
      {
        "elapsed_s": 1680,
        "outcome": "Attempted",
        "rate_chunks_s": 54.418518518518525
      },
      {
        "elapsed_s": 1680,
        "outcome": "Skipped",
        "rate_chunks_s": 54.418518518518525
      },
      {
        "elapsed_s": 1695,
        "outcome": "Attempted",
        "rate_chunks_s": 55
      },
      {
        "elapsed_s": 1695,
        "outcome": "Skipped",
        "rate_chunks_s": 55
      },
      {
        "elapsed_s": 1710,
        "outcome": "Attempted",
        "rate_chunks_s": 55.63333333333334
      },
      {
        "elapsed_s": 1710,
        "outcome": "Skipped",
        "rate_chunks_s": 55.63333333333334
      },
      {
        "elapsed_s": 1725,
        "outcome": "Attempted",
        "rate_chunks_s": 57.87407407407407
      },
      {
        "elapsed_s": 1725,
        "outcome": "Skipped",
        "rate_chunks_s": 57.87407407407407
      },
      {
        "elapsed_s": 1740,
        "outcome": "Attempted",
        "rate_chunks_s": 59.72222222222223
      },
      {
        "elapsed_s": 1740,
        "outcome": "Skipped",
        "rate_chunks_s": 59.72222222222223
      },
      {
        "elapsed_s": 1755,
        "outcome": "Attempted",
        "rate_chunks_s": 61.34444444444445
      },
      {
        "elapsed_s": 1755,
        "outcome": "Skipped",
        "rate_chunks_s": 61.34444444444445
      },
      {
        "elapsed_s": 1770,
        "outcome": "Attempted",
        "rate_chunks_s": 61.10370370370371
      },
      {
        "elapsed_s": 1770,
        "outcome": "Skipped",
        "rate_chunks_s": 61.10370370370371
      },
      {
        "elapsed_s": 1785,
        "outcome": "Attempted",
        "rate_chunks_s": 61.54444444444445
      },
      {
        "elapsed_s": 1785,
        "outcome": "Skipped",
        "rate_chunks_s": 61.54444444444445
      },
      {
        "elapsed_s": 1800,
        "outcome": "Attempted",
        "rate_chunks_s": 61.925925925925924
      },
      {
        "elapsed_s": 1800,
        "outcome": "Skipped",
        "rate_chunks_s": 61.925925925925924
      },
      {
        "elapsed_s": 1815,
        "outcome": "Attempted",
        "rate_chunks_s": 60.733333333333334
      },
      {
        "elapsed_s": 1815,
        "outcome": "Skipped",
        "rate_chunks_s": 60.733333333333334
      },
      {
        "elapsed_s": 1830,
        "outcome": "Attempted",
        "rate_chunks_s": 60.84444444444445
      },
      {
        "elapsed_s": 1830,
        "outcome": "Skipped",
        "rate_chunks_s": 60.84444444444445
      },
      {
        "elapsed_s": 1845,
        "outcome": "Attempted",
        "rate_chunks_s": 59.3
      },
      {
        "elapsed_s": 1845,
        "outcome": "Skipped",
        "rate_chunks_s": 59.3
      },
      {
        "elapsed_s": 1860,
        "outcome": "Attempted",
        "rate_chunks_s": 57.88888888888889
      },
      {
        "elapsed_s": 1860,
        "outcome": "Skipped",
        "rate_chunks_s": 57.88888888888889
      },
      {
        "elapsed_s": 1875,
        "outcome": "Attempted",
        "rate_chunks_s": 58
      },
      {
        "elapsed_s": 1875,
        "outcome": "Skipped",
        "rate_chunks_s": 58
      },
      {
        "elapsed_s": 1890,
        "outcome": "Attempted",
        "rate_chunks_s": 57.46296296296296
      },
      {
        "elapsed_s": 1890,
        "outcome": "Skipped",
        "rate_chunks_s": 57.46296296296296
      },
      {
        "elapsed_s": 1905,
        "outcome": "Attempted",
        "rate_chunks_s": 55.662962962962965
      },
      {
        "elapsed_s": 1905,
        "outcome": "Skipped",
        "rate_chunks_s": 55.662962962962965
      },
      {
        "elapsed_s": 1920,
        "outcome": "Attempted",
        "rate_chunks_s": 53.196296296296296
      },
      {
        "elapsed_s": 1920,
        "outcome": "Skipped",
        "rate_chunks_s": 53.196296296296296
      },
      {
        "elapsed_s": 1935,
        "outcome": "Attempted",
        "rate_chunks_s": 51.69259259259259
      },
      {
        "elapsed_s": 1935,
        "outcome": "Skipped",
        "rate_chunks_s": 51.69259259259259
      },
      {
        "elapsed_s": 1950,
        "outcome": "Attempted",
        "rate_chunks_s": 50.99629629629629
      },
      {
        "elapsed_s": 1950,
        "outcome": "Skipped",
        "rate_chunks_s": 50.99629629629629
      },
      {
        "elapsed_s": 1965,
        "outcome": "Attempted",
        "rate_chunks_s": 50.96296296296296
      },
      {
        "elapsed_s": 1965,
        "outcome": "Skipped",
        "rate_chunks_s": 50.96296296296296
      },
      {
        "elapsed_s": 1980,
        "outcome": "Attempted",
        "rate_chunks_s": 50.88518518518519
      },
      {
        "elapsed_s": 1980,
        "outcome": "Skipped",
        "rate_chunks_s": 50.88518518518519
      },
      {
        "elapsed_s": 1995,
        "outcome": "Attempted",
        "rate_chunks_s": 49.50370370370371
      },
      {
        "elapsed_s": 1995,
        "outcome": "Skipped",
        "rate_chunks_s": 49.50370370370371
      },
      {
        "elapsed_s": 2010,
        "outcome": "Attempted",
        "rate_chunks_s": 48.14074074074074
      },
      {
        "elapsed_s": 2010,
        "outcome": "Skipped",
        "rate_chunks_s": 48.14074074074074
      },
      {
        "elapsed_s": 2025,
        "outcome": "Attempted",
        "rate_chunks_s": 48.833333333333336
      },
      {
        "elapsed_s": 2025,
        "outcome": "Skipped",
        "rate_chunks_s": 48.833333333333336
      },
      {
        "elapsed_s": 2040,
        "outcome": "Attempted",
        "rate_chunks_s": 50.15925925925926
      },
      {
        "elapsed_s": 2040,
        "outcome": "Skipped",
        "rate_chunks_s": 50.15925925925926
      },
      {
        "elapsed_s": 2055,
        "outcome": "Attempted",
        "rate_chunks_s": 50.355555555555554
      },
      {
        "elapsed_s": 2055,
        "outcome": "Skipped",
        "rate_chunks_s": 50.355555555555554
      },
      {
        "elapsed_s": 2070,
        "outcome": "Attempted",
        "rate_chunks_s": 49.86296296296297
      },
      {
        "elapsed_s": 2070,
        "outcome": "Skipped",
        "rate_chunks_s": 49.86296296296297
      },
      {
        "elapsed_s": 2085,
        "outcome": "Attempted",
        "rate_chunks_s": 49.02962962962963
      },
      {
        "elapsed_s": 2085,
        "outcome": "Skipped",
        "rate_chunks_s": 49.02962962962963
      },
      {
        "elapsed_s": 2100,
        "outcome": "Attempted",
        "rate_chunks_s": 47.63703703703704
      },
      {
        "elapsed_s": 2100,
        "outcome": "Skipped",
        "rate_chunks_s": 47.63703703703704
      },
      {
        "elapsed_s": 2115,
        "outcome": "Attempted",
        "rate_chunks_s": 46.56666666666666
      },
      {
        "elapsed_s": 2115,
        "outcome": "Skipped",
        "rate_chunks_s": 46.56666666666666
      },
      {
        "elapsed_s": 2130,
        "outcome": "Attempted",
        "rate_chunks_s": 45.34074074074074
      },
      {
        "elapsed_s": 2130,
        "outcome": "Skipped",
        "rate_chunks_s": 45.34074074074074
      },
      {
        "elapsed_s": 2145,
        "outcome": "Attempted",
        "rate_chunks_s": 43.53703703703704
      },
      {
        "elapsed_s": 2145,
        "outcome": "Skipped",
        "rate_chunks_s": 43.53703703703704
      },
      {
        "elapsed_s": 2160,
        "outcome": "Attempted",
        "rate_chunks_s": 42.992592592592594
      },
      {
        "elapsed_s": 2160,
        "outcome": "Skipped",
        "rate_chunks_s": 42.992592592592594
      },
      {
        "elapsed_s": 2175,
        "outcome": "Attempted",
        "rate_chunks_s": 43.281481481481485
      },
      {
        "elapsed_s": 2175,
        "outcome": "Skipped",
        "rate_chunks_s": 43.281481481481485
      },
      {
        "elapsed_s": 2190,
        "outcome": "Attempted",
        "rate_chunks_s": 43.82222222222222
      },
      {
        "elapsed_s": 2190,
        "outcome": "Skipped",
        "rate_chunks_s": 43.82222222222222
      },
      {
        "elapsed_s": 2205,
        "outcome": "Attempted",
        "rate_chunks_s": 44.36296296296297
      },
      {
        "elapsed_s": 2205,
        "outcome": "Skipped",
        "rate_chunks_s": 44.36296296296297
      },
      {
        "elapsed_s": 2220,
        "outcome": "Attempted",
        "rate_chunks_s": 44.12222222222222
      },
      {
        "elapsed_s": 2220,
        "outcome": "Skipped",
        "rate_chunks_s": 44.12222222222222
      },
      {
        "elapsed_s": 2235,
        "outcome": "Attempted",
        "rate_chunks_s": 45.12592592592593
      },
      {
        "elapsed_s": 2235,
        "outcome": "Skipped",
        "rate_chunks_s": 45.12592592592593
      },
      {
        "elapsed_s": 2250,
        "outcome": "Attempted",
        "rate_chunks_s": 46.3962962962963
      },
      {
        "elapsed_s": 2250,
        "outcome": "Skipped",
        "rate_chunks_s": 46.3962962962963
      },
      {
        "elapsed_s": 2265,
        "outcome": "Attempted",
        "rate_chunks_s": 46.35925925925926
      },
      {
        "elapsed_s": 2265,
        "outcome": "Skipped",
        "rate_chunks_s": 46.35925925925926
      },
      {
        "elapsed_s": 2280,
        "outcome": "Attempted",
        "rate_chunks_s": 46.21111111111111
      },
      {
        "elapsed_s": 2280,
        "outcome": "Skipped",
        "rate_chunks_s": 46.21111111111111
      },
      {
        "elapsed_s": 2295,
        "outcome": "Attempted",
        "rate_chunks_s": 43.75925925925926
      },
      {
        "elapsed_s": 2295,
        "outcome": "Skipped",
        "rate_chunks_s": 43.75925925925926
      },
      {
        "elapsed_s": 2310,
        "outcome": "Attempted",
        "rate_chunks_s": 41.03703703703704
      },
      {
        "elapsed_s": 2310,
        "outcome": "Skipped",
        "rate_chunks_s": 41.03703703703704
      },
      {
        "elapsed_s": 2325,
        "outcome": "Attempted",
        "rate_chunks_s": 40.214814814814815
      },
      {
        "elapsed_s": 2325,
        "outcome": "Skipped",
        "rate_chunks_s": 40.214814814814815
      },
      {
        "elapsed_s": 2340,
        "outcome": "Attempted",
        "rate_chunks_s": 40.56666666666666
      },
      {
        "elapsed_s": 2340,
        "outcome": "Skipped",
        "rate_chunks_s": 40.56666666666666
      },
      {
        "elapsed_s": 2355,
        "outcome": "Attempted",
        "rate_chunks_s": 39.988888888888894
      },
      {
        "elapsed_s": 2355,
        "outcome": "Skipped",
        "rate_chunks_s": 39.988888888888894
      },
      {
        "elapsed_s": 2370,
        "outcome": "Attempted",
        "rate_chunks_s": 38.3962962962963
      },
      {
        "elapsed_s": 2370,
        "outcome": "Skipped",
        "rate_chunks_s": 38.3962962962963
      },
      {
        "elapsed_s": 2385,
        "outcome": "Attempted",
        "rate_chunks_s": 36.733333333333334
      },
      {
        "elapsed_s": 2385,
        "outcome": "Skipped",
        "rate_chunks_s": 36.733333333333334
      },
      {
        "elapsed_s": 2400,
        "outcome": "Attempted",
        "rate_chunks_s": 34.477777777777774
      },
      {
        "elapsed_s": 2400,
        "outcome": "Skipped",
        "rate_chunks_s": 34.477777777777774
      },
      {
        "elapsed_s": 2415,
        "outcome": "Attempted",
        "rate_chunks_s": 33.06666666666666
      },
      {
        "elapsed_s": 2415,
        "outcome": "Skipped",
        "rate_chunks_s": 33.06666666666666
      },
      {
        "elapsed_s": 2430,
        "outcome": "Attempted",
        "rate_chunks_s": 31.74074074074074
      },
      {
        "elapsed_s": 2430,
        "outcome": "Skipped",
        "rate_chunks_s": 31.74074074074074
      }
    ]
  },
  "mark": {
    "type": "line",
    "strokeWidth": 2
  },
  "encoding": {
    "x": {
      "field": "elapsed_s",
      "type": "quantitative",
      "title": "Elapsed time from metric window start (s)"
    },
    "y": {
      "field": "rate_chunks_s",
      "type": "quantitative",
      "title": "Prefetch outcome rate (chunks/s)",
      "scale": {
        "zero": true
      }
    },
    "color": {
      "field": "outcome",
      "type": "nominal",
      "title": "Counter",
      "scale": {
        "domain": [
          "Attempted",
          "Skipped"
        ],
        "scheme": "category10"
      }
    },
    "strokeDash": {
      "field": "outcome",
      "type": "nominal",
      "title": "Counter"
    },
    "tooltip": [
      {
        "field": "elapsed_s",
        "type": "quantitative",
        "title": "Elapsed (s)"
      },
      {
        "field": "outcome",
        "type": "nominal"
      },
      {
        "field": "rate_chunks_s",
        "type": "quantitative",
        "title": "Rate (chunks/s)",
        "format": ".3f"
      }
    ]
  }
}
```

The average and range were identical for both outcomes: 50.762 chunks/s average, 1.045 minimum, and 92.285 maximum. No `prefetch_promoted`, `prefetch_useful`, `prefetch_wasted`, `prefetch_redundant`, or `prefetch_untracked` series were present. Therefore:

$$
\text{skip fraction} =
\frac{\text{skipped rate}}{\text{attempted rate}} = 1.0
$$

at every exported sample, and

$$
\text{promotion fraction} =
\frac{\text{promoted rate}}{\text{attempted rate}} = 0.
$$

### Root cause

The deployed prefetch profile contains:

```text
TieringOffloadingSpec
cpu_bytes_to_use = 274877906944
prefetch_chunks = 100
secondary_tiers = []
```

The code path is secondary→CPU read-ahead. It first checks CPU residency, then searches configured secondary tiers; only a secondary hit can initiate a promotion. With an empty secondary-tier list, non-resident upcoming keys necessarily take the “not in any secondary tier” branch and increment `PREFETCH_SKIPPED`.

The unrelated startup message about safetensors filesystem prefetch concerns model-weight loading, not Phase 1 KV prefetch, and does not explain this result.

## Other mechanism and validity signals

| Signal | 256 GiB control | Nominal prefetch $N=100$ | Interpretation |
|---|---:|---:|---|
| External KV prompt-token share, average | 2.504% | 2.404% | Nearly unchanged; no extra secondary→CPU source existed |
| External prefix-cache hit rate, average | 18.990% | 18.638% | Nearly unchanged |
| CPU cache occupancy, average / max | 0.375% / 1.5625% | 0.394% / 1.5625% | No speculative residency pressure |
| CPU read-pinned occupancy, average / max | 0% / 0% | 0.034% / 1.541% | Small demand activity, not evidence of prefetch promotion |
| Transfer byte rate, average across exported series | 517.15 MB/s | 513.49 MB/s | Comparable demand offload traffic |
| Waiting requests, average / max | 0.270 / 10 | 0.307 / 10 | No clear queueing improvement |
| Preemptions | 0 | 0 | Clean |
| Allocation failures | No series | No series | No observed failure, but absence is not a numeric zero |

GPU core/SM and PCIe telemetry were unavailable in these artifacts; they are not inferred from KV-cache metrics.

## Recommendation for the next test

Use **`prefetch_chunks=100` again for the next plumbing-validation test**, after adding a real, populated secondary tier. Keeping $N=100$ holds the nominal knob constant and isolates the configuration fix. If the deployment wrapper names the knob `prefetch_blocks`, set `prefetch_blocks=100` only if one wrapper block maps to one offload chunk; the patched image itself consumes `prefetch_chunks`. The exported GPU KV block size is 16 tokens, and the manifest did not override offload block size.

This is a validation value, **not an estimated optimum**. Do not tune $N$ from this rejected run.

The next test must pass these gates before a performance comparison is accepted:

1. The rendered manifest contains at least one intended `secondary_tiers` entry and the tier is populated with keys the replay will request.
2. `attempted > 0`, `promoted > 0`, and skipped/attempted is below 100%.
3. After enough demand lookups, `useful + wasted > 0`; `untracked/promoted` remains acceptably low.
4. Secondary read/transfer activity aligns with promotions.
5. The $N=0$ and $N=100$ cells are repeated at least three times under the same cache-cleaning protocol.

After the plumbing gate passes, run the actual fixed-$N$ sweep, for example `{0, 32, 64, 100, 128, 256}`, and select the smallest $N$ near the minimum paired latency/lookup-event curve with a high useful ratio and no allocation/occupancy pressure.

## Run registry

- [No-offload reference — c2c2e87883324898995c3ca1639db3b1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/c2c2e87883324898995c3ca1639db3b1?workspace=benchflow) — finished; accepted as system reference, not as the mechanism baseline.
- [256 GiB CPU-offload control — 5f57165d7d464cee8514645215c526c7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5f57165d7d464cee8514645215c526c7?workspace=benchflow) — finished; accepted as the within-pair control.
- [Nominal prefetch $N=100$ — d5bace21821648ec96bcb7f6efdb3077](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d5bace21821648ec96bcb7f6efdb3077?workspace=benchflow) — finished; rejected for prefetch-effect/tuning conclusions because every attempt was skipped.

## Conclusion

Phase 1 remains open. The batch confirms that the scheduler hook and counters execute, but it does **not** confirm working KV prefetch. The smallest decisive next experiment is a corrected secondary-tier deployment with the same $N=100$ validation value, followed only then by a repeated $N$ sweep.