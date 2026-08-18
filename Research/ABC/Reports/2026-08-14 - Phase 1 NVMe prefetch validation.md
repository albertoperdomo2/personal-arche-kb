---
title: "ABC Phase 1 — NVMe prefetch validation"
date: "2026-08-14"
type: "research-report"
experiment: "Activity-Based KV Cache Tier Placement"
phase: "1 — naive proactive prefetching"
status: "invalid-inconclusive"
root_cause_status: "confirmed from code and telemetry"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "unknown"
image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-p1"
tensor_parallelism: 8
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 131072
concurrency: 32
cpu_bytes: 274877906944
secondary_tier: "filesystem on /mnt/nvme-kv-cache"
secondary_tier_threads: "64 read / 64 write"
workload: "semianalysisai/cc-traces-weka-062126"
random_seed: 20260707
duration_seconds: 1800
configuration:
  nvme_control:
    run_id: "988f03995bb745659749110472019c6b"
    prefetch_chunks: 0
  nvme_prefetch:
    run_id: "96d01b33a71f4f1bbb2d55a53a8aaacd"
    prefetch_chunks: 100
---

# ABC Phase 1 — NVMe prefetch validation

## Executive summary

This controlled NVMe pair answers the configuration question left open by the CPU-only batch. Both runs used a 256 GiB CPU primary tier and the same filesystem secondary tier at `/mnt/nvme-kv-cache`; the prefetch cell added only `prefetch_chunks=100`.

The NVMe tier was active for ordinary reactive KV lookup, but **Phase 1 prefetch still performed zero promotions**. Across 153 native 15-second samples, the prefetch run averaged 119.103 attempted chunks/s and exactly 119.103 skipped chunks/s. Attempted and skipped were identical sample-by-sample; promoted, useful, wasted, redundant, and untracked series were absent.

# Invalid / inconclusive for performance and N selection

This pair is valid for diagnosing the mechanism but invalid for attributing latency changes to prefetch or tuning $N$. The mechanism under test never moved a block, and there is only one run per cell.

## What the pair establishes

- **Measured:** both manifests contain the same filesystem secondary tier, 64 read threads, 64 write threads, a 256 GiB CPU primary tier, TP=8, and otherwise matching benchmark settings.
- **Measured:** reactive NVMe lookup was active. External-prefix hits were about 27%, async tiering stalls occurred at about 0.74 events/s, and the active NVMe device sustained reads and writes in both cells.
- **Measured:** `prefetch_chunks=100` loaded and the scheduler hook fired, but every selected candidate was skipped. No promoted/useful/wasted telemetry appeared.
- **Code-confirmed root cause:** the hook runs only on the first terminal `MISS` and submits `keys[local_idx + 1 : local_idx + 1 + N]`. vLLM block hashes are chained through the entire preceding prefix, while the filesystem tier persistently stores every successfully cascaded chunk. Consequently, if a later key had been stored, its predecessor must also have been stored. A genuine miss at chunk $i$ therefore implies that chunks $i+1, \ldots, i+N$ are unavailable under normal operation.
- **Conclusion:** changing $N$ alone cannot repair the mechanism. The trigger/candidate construction must change before an $N$ sweep is meaningful.

## Headline outcomes

Both runs completed exactly 863 profiling requests with zero errors or cancellations.

| Metric | NVMe control, $N=0$ | NVMe nominal prefetch, $N=100$ | Relative delta |
|---|---:|---:|---:|
| Request throughput (req/s) | 0.469021 | 0.469021 | approximately 0.000% |
| Output throughput (tok/s) | 350.383 | 350.383 | approximately 0.000% |
| Request latency mean (ms) | 20,866.36 | 20,876.14 | +0.047% |
| Request latency P50 (ms) | 9,041.57 | 9,048.85 | +0.080% |
| Request latency P90 (ms) | 37,934.90 | 38,503.88 | +1.500% |
| Request latency P95 (ms) | 51,930.81 | 52,541.77 | +1.176% |
| TTFT mean (ms) | 1,124.09 | 1,175.30 | +4.555% |
| TTFT P50 (ms) | 583.13 | 599.81 | +2.861% |
| TTFT P90 (ms) | 2,464.41 | 2,832.07 | +14.920% |
| TTFT P95 (ms) | 4,614.44 | 4,686.07 | +1.552% |
| ITL mean (ms) | 27.575 | 26.729 | −3.069% |
| ITL P95 (ms) | 46.258 | 44.106 | −4.652% |

The mixed, small differences are ordinary between-run variation for purposes of this mechanism test. Because the prefetch promotion count was zero, none can be attributed to prefetch.

## Mechanism telemetry

The source query exported 153 samples at a 15-second cadence. Attempted and skipped values are exactly equal in the raw JSON, including the largest burst.

- Mean: 119.103 chunks/s for each counter
- Minimum: 2.202 chunks/s for each counter
- Maximum: 522.156 chunks/s for each counter
- Skip fraction: 100% at every sample
- Promotion fraction: 0%
- Promoted/useful/wasted/redundant/untracked: no series

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — NVMe Phase 1 prefetch attempts and skips overlap exactly",
  "width": 700,
  "height": 320,
  "background": "white",
  "data": {
    "values": [
      {
        "elapsed_s": 0,
        "outcome": "Attempted",
        "rate_chunks_s": 2.201533333333333
      },
      {
        "elapsed_s": 15,
        "outcome": "Attempted",
        "rate_chunks_s": 2.8682
      },
      {
        "elapsed_s": 30,
        "outcome": "Attempted",
        "rate_chunks_s": 3.0866888888888893
      },
      {
        "elapsed_s": 45,
        "outcome": "Attempted",
        "rate_chunks_s": 3.6700222222222223
      },
      {
        "elapsed_s": 60,
        "outcome": "Attempted",
        "rate_chunks_s": 4.0568333333333335
      },
      {
        "elapsed_s": 75,
        "outcome": "Attempted",
        "rate_chunks_s": 4.612388888888889
      },
      {
        "elapsed_s": 90,
        "outcome": "Attempted",
        "rate_chunks_s": 4.6511499999999995
      },
      {
        "elapsed_s": 105,
        "outcome": "Attempted",
        "rate_chunks_s": 5.151149999999999
      },
      {
        "elapsed_s": 120,
        "outcome": "Attempted",
        "rate_chunks_s": 6.027893333333333
      },
      {
        "elapsed_s": 135,
        "outcome": "Attempted",
        "rate_chunks_s": 6.561226666666666
      },
      {
        "elapsed_s": 150,
        "outcome": "Attempted",
        "rate_chunks_s": 7.3901666666666666
      },
      {
        "elapsed_s": 165,
        "outcome": "Attempted",
        "rate_chunks_s": 7.945722222222223
      },
      {
        "elapsed_s": 180,
        "outcome": "Attempted",
        "rate_chunks_s": 8.015490476190475
      },
      {
        "elapsed_s": 195,
        "outcome": "Attempted",
        "rate_chunks_s": 8.5393
      },
      {
        "elapsed_s": 210,
        "outcome": "Attempted",
        "rate_chunks_s": 10.451893055555555
      },
      {
        "elapsed_s": 225,
        "outcome": "Attempted",
        "rate_chunks_s": 11.056059722222223
      },
      {
        "elapsed_s": 240,
        "outcome": "Attempted",
        "rate_chunks_s": 11.850001234567902
      },
      {
        "elapsed_s": 255,
        "outcome": "Attempted",
        "rate_chunks_s": 12.222222222222223
      },
      {
        "elapsed_s": 270,
        "outcome": "Attempted",
        "rate_chunks_s": 12.592592592592593
      },
      {
        "elapsed_s": 285,
        "outcome": "Attempted",
        "rate_chunks_s": 12.592592592592593
      },
      {
        "elapsed_s": 300,
        "outcome": "Attempted",
        "rate_chunks_s": 13.703703703703704
      },
      {
        "elapsed_s": 315,
        "outcome": "Attempted",
        "rate_chunks_s": 13.703703703703704
      },
      {
        "elapsed_s": 330,
        "outcome": "Attempted",
        "rate_chunks_s": 14.074074074074074
      },
      {
        "elapsed_s": 345,
        "outcome": "Attempted",
        "rate_chunks_s": 14.074074074074074
      },
      {
        "elapsed_s": 360,
        "outcome": "Attempted",
        "rate_chunks_s": 14.814814814814815
      },
      {
        "elapsed_s": 375,
        "outcome": "Attempted",
        "rate_chunks_s": 14.814814814814815
      },
      {
        "elapsed_s": 390,
        "outcome": "Attempted",
        "rate_chunks_s": 16.296296296296298
      },
      {
        "elapsed_s": 405,
        "outcome": "Attempted",
        "rate_chunks_s": 16.296296296296298
      },
      {
        "elapsed_s": 420,
        "outcome": "Attempted",
        "rate_chunks_s": 17.40740740740741
      },
      {
        "elapsed_s": 435,
        "outcome": "Attempted",
        "rate_chunks_s": 17.40740740740741
      },
      {
        "elapsed_s": 450,
        "outcome": "Attempted",
        "rate_chunks_s": 25.348148148148148
      },
      {
        "elapsed_s": 465,
        "outcome": "Attempted",
        "rate_chunks_s": 25.348148148148148
      },
      {
        "elapsed_s": 480,
        "outcome": "Attempted",
        "rate_chunks_s": 29.185185185185187
      },
      {
        "elapsed_s": 495,
        "outcome": "Attempted",
        "rate_chunks_s": 29.185185185185187
      },
      {
        "elapsed_s": 510,
        "outcome": "Attempted",
        "rate_chunks_s": 37.85925925925926
      },
      {
        "elapsed_s": 525,
        "outcome": "Attempted",
        "rate_chunks_s": 37.85925925925926
      },
      {
        "elapsed_s": 540,
        "outcome": "Attempted",
        "rate_chunks_s": 49.72592592592593
      },
      {
        "elapsed_s": 555,
        "outcome": "Attempted",
        "rate_chunks_s": 49.72592592592593
      },
      {
        "elapsed_s": 570,
        "outcome": "Attempted",
        "rate_chunks_s": 55.470370370370375
      },
      {
        "elapsed_s": 585,
        "outcome": "Attempted",
        "rate_chunks_s": 55.470370370370375
      },
      {
        "elapsed_s": 600,
        "outcome": "Attempted",
        "rate_chunks_s": 64.33703703703705
      },
      {
        "elapsed_s": 615,
        "outcome": "Attempted",
        "rate_chunks_s": 64.33703703703705
      },
      {
        "elapsed_s": 630,
        "outcome": "Attempted",
        "rate_chunks_s": 113.51851851851852
      },
      {
        "elapsed_s": 645,
        "outcome": "Attempted",
        "rate_chunks_s": 113.51851851851852
      },
      {
        "elapsed_s": 660,
        "outcome": "Attempted",
        "rate_chunks_s": 118.46296296296296
      },
      {
        "elapsed_s": 675,
        "outcome": "Attempted",
        "rate_chunks_s": 118.46296296296296
      },
      {
        "elapsed_s": 690,
        "outcome": "Attempted",
        "rate_chunks_s": 129.33333333333331
      },
      {
        "elapsed_s": 705,
        "outcome": "Attempted",
        "rate_chunks_s": 129.33333333333331
      },
      {
        "elapsed_s": 720,
        "outcome": "Attempted",
        "rate_chunks_s": 130.4
      },
      {
        "elapsed_s": 735,
        "outcome": "Attempted",
        "rate_chunks_s": 130.4
      },
      {
        "elapsed_s": 750,
        "outcome": "Attempted",
        "rate_chunks_s": 211.52592592592595
      },
      {
        "elapsed_s": 765,
        "outcome": "Attempted",
        "rate_chunks_s": 211.52592592592595
      },
      {
        "elapsed_s": 780,
        "outcome": "Attempted",
        "rate_chunks_s": 322.6962962962963
      },
      {
        "elapsed_s": 795,
        "outcome": "Attempted",
        "rate_chunks_s": 322.6962962962963
      },
      {
        "elapsed_s": 810,
        "outcome": "Attempted",
        "rate_chunks_s": 340.77777777777777
      },
      {
        "elapsed_s": 825,
        "outcome": "Attempted",
        "rate_chunks_s": 340.77777777777777
      },
      {
        "elapsed_s": 840,
        "outcome": "Attempted",
        "rate_chunks_s": 342.35185185185185
      },
      {
        "elapsed_s": 855,
        "outcome": "Attempted",
        "rate_chunks_s": 342.35185185185185
      },
      {
        "elapsed_s": 870,
        "outcome": "Attempted",
        "rate_chunks_s": 342.4259259259259
      },
      {
        "elapsed_s": 885,
        "outcome": "Attempted",
        "rate_chunks_s": 342.4259259259259
      },
      {
        "elapsed_s": 900,
        "outcome": "Attempted",
        "rate_chunks_s": 357.56296296296296
      },
      {
        "elapsed_s": 915,
        "outcome": "Attempted",
        "rate_chunks_s": 357.56296296296296
      },
      {
        "elapsed_s": 930,
        "outcome": "Attempted",
        "rate_chunks_s": 435.31481481481484
      },
      {
        "elapsed_s": 945,
        "outcome": "Attempted",
        "rate_chunks_s": 435.31481481481484
      },
      {
        "elapsed_s": 960,
        "outcome": "Attempted",
        "rate_chunks_s": 522.1555555555556
      },
      {
        "elapsed_s": 975,
        "outcome": "Attempted",
        "rate_chunks_s": 522.1555555555556
      },
      {
        "elapsed_s": 990,
        "outcome": "Attempted",
        "rate_chunks_s": 519.9185185185186
      },
      {
        "elapsed_s": 1005,
        "outcome": "Attempted",
        "rate_chunks_s": 519.9185185185186
      },
      {
        "elapsed_s": 1020,
        "outcome": "Attempted",
        "rate_chunks_s": 438.7185185185185
      },
      {
        "elapsed_s": 1035,
        "outcome": "Attempted",
        "rate_chunks_s": 438.7185185185185
      },
      {
        "elapsed_s": 1050,
        "outcome": "Attempted",
        "rate_chunks_s": 326.8666666666667
      },
      {
        "elapsed_s": 1065,
        "outcome": "Attempted",
        "rate_chunks_s": 326.8666666666667
      },
      {
        "elapsed_s": 1080,
        "outcome": "Attempted",
        "rate_chunks_s": 301.77777777777777
      },
      {
        "elapsed_s": 1095,
        "outcome": "Attempted",
        "rate_chunks_s": 301.77777777777777
      },
      {
        "elapsed_s": 1110,
        "outcome": "Attempted",
        "rate_chunks_s": 300.63703703703703
      },
      {
        "elapsed_s": 1125,
        "outcome": "Attempted",
        "rate_chunks_s": 300.63703703703703
      },
      {
        "elapsed_s": 1140,
        "outcome": "Attempted",
        "rate_chunks_s": 295.93333333333334
      },
      {
        "elapsed_s": 1155,
        "outcome": "Attempted",
        "rate_chunks_s": 295.93333333333334
      },
      {
        "elapsed_s": 1170,
        "outcome": "Attempted",
        "rate_chunks_s": 236.93333333333334
      },
      {
        "elapsed_s": 1185,
        "outcome": "Attempted",
        "rate_chunks_s": 236.93333333333334
      },
      {
        "elapsed_s": 1200,
        "outcome": "Attempted",
        "rate_chunks_s": 159.54814814814816
      },
      {
        "elapsed_s": 1215,
        "outcome": "Attempted",
        "rate_chunks_s": 159.54814814814816
      },
      {
        "elapsed_s": 1230,
        "outcome": "Attempted",
        "rate_chunks_s": 63.25185185185185
      },
      {
        "elapsed_s": 1245,
        "outcome": "Attempted",
        "rate_chunks_s": 63.25185185185185
      },
      {
        "elapsed_s": 1260,
        "outcome": "Attempted",
        "rate_chunks_s": 60.762962962962966
      },
      {
        "elapsed_s": 1275,
        "outcome": "Attempted",
        "rate_chunks_s": 60.762962962962966
      },
      {
        "elapsed_s": 1290,
        "outcome": "Attempted",
        "rate_chunks_s": 58.800000000000004
      },
      {
        "elapsed_s": 1305,
        "outcome": "Attempted",
        "rate_chunks_s": 58.800000000000004
      },
      {
        "elapsed_s": 1320,
        "outcome": "Attempted",
        "rate_chunks_s": 56.90370370370371
      },
      {
        "elapsed_s": 1335,
        "outcome": "Attempted",
        "rate_chunks_s": 56.90370370370371
      },
      {
        "elapsed_s": 1350,
        "outcome": "Attempted",
        "rate_chunks_s": 53.11851851851852
      },
      {
        "elapsed_s": 1365,
        "outcome": "Attempted",
        "rate_chunks_s": 53.11851851851852
      },
      {
        "elapsed_s": 1380,
        "outcome": "Attempted",
        "rate_chunks_s": 52.31111111111111
      },
      {
        "elapsed_s": 1395,
        "outcome": "Attempted",
        "rate_chunks_s": 52.31111111111111
      },
      {
        "elapsed_s": 1410,
        "outcome": "Attempted",
        "rate_chunks_s": 54.974074074074075
      },
      {
        "elapsed_s": 1425,
        "outcome": "Attempted",
        "rate_chunks_s": 54.974074074074075
      },
      {
        "elapsed_s": 1440,
        "outcome": "Attempted",
        "rate_chunks_s": 55.07777777777778
      },
      {
        "elapsed_s": 1455,
        "outcome": "Attempted",
        "rate_chunks_s": 55.07777777777778
      },
      {
        "elapsed_s": 1470,
        "outcome": "Attempted",
        "rate_chunks_s": 93.94074074074075
      },
      {
        "elapsed_s": 1485,
        "outcome": "Attempted",
        "rate_chunks_s": 93.94074074074075
      },
      {
        "elapsed_s": 1500,
        "outcome": "Attempted",
        "rate_chunks_s": 95.78518518518518
      },
      {
        "elapsed_s": 1515,
        "outcome": "Attempted",
        "rate_chunks_s": 95.78518518518518
      },
      {
        "elapsed_s": 1530,
        "outcome": "Attempted",
        "rate_chunks_s": 99.20740740740742
      },
      {
        "elapsed_s": 1545,
        "outcome": "Attempted",
        "rate_chunks_s": 99.20740740740742
      },
      {
        "elapsed_s": 1560,
        "outcome": "Attempted",
        "rate_chunks_s": 107.86666666666667
      },
      {
        "elapsed_s": 1575,
        "outcome": "Attempted",
        "rate_chunks_s": 107.86666666666667
      },
      {
        "elapsed_s": 1590,
        "outcome": "Attempted",
        "rate_chunks_s": 106.63703703703705
      },
      {
        "elapsed_s": 1605,
        "outcome": "Attempted",
        "rate_chunks_s": 106.63703703703705
      },
      {
        "elapsed_s": 1620,
        "outcome": "Attempted",
        "rate_chunks_s": 110.01851851851852
      },
      {
        "elapsed_s": 1635,
        "outcome": "Attempted",
        "rate_chunks_s": 110.01851851851852
      },
      {
        "elapsed_s": 1650,
        "outcome": "Attempted",
        "rate_chunks_s": 108.33333333333334
      },
      {
        "elapsed_s": 1665,
        "outcome": "Attempted",
        "rate_chunks_s": 108.33333333333334
      },
      {
        "elapsed_s": 1680,
        "outcome": "Attempted",
        "rate_chunks_s": 106.9962962962963
      },
      {
        "elapsed_s": 1695,
        "outcome": "Attempted",
        "rate_chunks_s": 106.9962962962963
      },
      {
        "elapsed_s": 1710,
        "outcome": "Attempted",
        "rate_chunks_s": 103.12962962962963
      },
      {
        "elapsed_s": 1725,
        "outcome": "Attempted",
        "rate_chunks_s": 103.12962962962963
      },
      {
        "elapsed_s": 1740,
        "outcome": "Attempted",
        "rate_chunks_s": 59.4962962962963
      },
      {
        "elapsed_s": 1755,
        "outcome": "Attempted",
        "rate_chunks_s": 59.4962962962963
      },
      {
        "elapsed_s": 1770,
        "outcome": "Attempted",
        "rate_chunks_s": 57.03703703703704
      },
      {
        "elapsed_s": 1785,
        "outcome": "Attempted",
        "rate_chunks_s": 57.03703703703704
      },
      {
        "elapsed_s": 1800,
        "outcome": "Attempted",
        "rate_chunks_s": 54.87407407407407
      },
      {
        "elapsed_s": 1815,
        "outcome": "Attempted",
        "rate_chunks_s": 54.87407407407407
      },
      {
        "elapsed_s": 1830,
        "outcome": "Attempted",
        "rate_chunks_s": 47.76296296296296
      },
      {
        "elapsed_s": 1845,
        "outcome": "Attempted",
        "rate_chunks_s": 47.76296296296296
      },
      {
        "elapsed_s": 1860,
        "outcome": "Attempted",
        "rate_chunks_s": 51.02962962962963
      },
      {
        "elapsed_s": 1875,
        "outcome": "Attempted",
        "rate_chunks_s": 51.02962962962963
      },
      {
        "elapsed_s": 1890,
        "outcome": "Attempted",
        "rate_chunks_s": 52.02962962962963
      },
      {
        "elapsed_s": 1905,
        "outcome": "Attempted",
        "rate_chunks_s": 52.02962962962963
      },
      {
        "elapsed_s": 1920,
        "outcome": "Attempted",
        "rate_chunks_s": 50.38148148148149
      },
      {
        "elapsed_s": 1935,
        "outcome": "Attempted",
        "rate_chunks_s": 50.38148148148149
      },
      {
        "elapsed_s": 1950,
        "outcome": "Attempted",
        "rate_chunks_s": 46.63703703703704
      },
      {
        "elapsed_s": 1965,
        "outcome": "Attempted",
        "rate_chunks_s": 46.63703703703704
      },
      {
        "elapsed_s": 1980,
        "outcome": "Attempted",
        "rate_chunks_s": 47
      },
      {
        "elapsed_s": 1995,
        "outcome": "Attempted",
        "rate_chunks_s": 47
      },
      {
        "elapsed_s": 2010,
        "outcome": "Attempted",
        "rate_chunks_s": 48.44444444444444
      },
      {
        "elapsed_s": 2025,
        "outcome": "Attempted",
        "rate_chunks_s": 48.44444444444444
      },
      {
        "elapsed_s": 2040,
        "outcome": "Attempted",
        "rate_chunks_s": 50.51111111111111
      },
      {
        "elapsed_s": 2055,
        "outcome": "Attempted",
        "rate_chunks_s": 50.51111111111111
      },
      {
        "elapsed_s": 2070,
        "outcome": "Attempted",
        "rate_chunks_s": 49.592592592592595
      },
      {
        "elapsed_s": 2085,
        "outcome": "Attempted",
        "rate_chunks_s": 49.592592592592595
      },
      {
        "elapsed_s": 2100,
        "outcome": "Attempted",
        "rate_chunks_s": 46.20740740740741
      },
      {
        "elapsed_s": 2115,
        "outcome": "Attempted",
        "rate_chunks_s": 46.20740740740741
      },
      {
        "elapsed_s": 2130,
        "outcome": "Attempted",
        "rate_chunks_s": 131.66666666666669
      },
      {
        "elapsed_s": 2145,
        "outcome": "Attempted",
        "rate_chunks_s": 131.66666666666669
      },
      {
        "elapsed_s": 2160,
        "outcome": "Attempted",
        "rate_chunks_s": 128.33703703703705
      },
      {
        "elapsed_s": 2175,
        "outcome": "Attempted",
        "rate_chunks_s": 128.33703703703705
      },
      {
        "elapsed_s": 2190,
        "outcome": "Attempted",
        "rate_chunks_s": 126.02962962962962
      },
      {
        "elapsed_s": 2205,
        "outcome": "Attempted",
        "rate_chunks_s": 126.02962962962962
      },
      {
        "elapsed_s": 2220,
        "outcome": "Attempted",
        "rate_chunks_s": 129.3851851851852
      },
      {
        "elapsed_s": 2235,
        "outcome": "Attempted",
        "rate_chunks_s": 129.3851851851852
      },
      {
        "elapsed_s": 2250,
        "outcome": "Attempted",
        "rate_chunks_s": 128.2962962962963
      },
      {
        "elapsed_s": 2265,
        "outcome": "Attempted",
        "rate_chunks_s": 128.2962962962963
      },
      {
        "elapsed_s": 2280,
        "outcome": "Attempted",
        "rate_chunks_s": 123.34814814814816
      },
      {
        "elapsed_s": 0,
        "outcome": "Skipped",
        "rate_chunks_s": 2.201533333333333
      },
      {
        "elapsed_s": 15,
        "outcome": "Skipped",
        "rate_chunks_s": 2.8682
      },
      {
        "elapsed_s": 30,
        "outcome": "Skipped",
        "rate_chunks_s": 3.0866888888888893
      },
      {
        "elapsed_s": 45,
        "outcome": "Skipped",
        "rate_chunks_s": 3.6700222222222223
      },
      {
        "elapsed_s": 60,
        "outcome": "Skipped",
        "rate_chunks_s": 4.0568333333333335
      },
      {
        "elapsed_s": 75,
        "outcome": "Skipped",
        "rate_chunks_s": 4.612388888888889
      },
      {
        "elapsed_s": 90,
        "outcome": "Skipped",
        "rate_chunks_s": 4.6511499999999995
      },
      {
        "elapsed_s": 105,
        "outcome": "Skipped",
        "rate_chunks_s": 5.151149999999999
      },
      {
        "elapsed_s": 120,
        "outcome": "Skipped",
        "rate_chunks_s": 6.027893333333333
      },
      {
        "elapsed_s": 135,
        "outcome": "Skipped",
        "rate_chunks_s": 6.561226666666666
      },
      {
        "elapsed_s": 150,
        "outcome": "Skipped",
        "rate_chunks_s": 7.3901666666666666
      },
      {
        "elapsed_s": 165,
        "outcome": "Skipped",
        "rate_chunks_s": 7.945722222222223
      },
      {
        "elapsed_s": 180,
        "outcome": "Skipped",
        "rate_chunks_s": 8.015490476190475
      },
      {
        "elapsed_s": 195,
        "outcome": "Skipped",
        "rate_chunks_s": 8.5393
      },
      {
        "elapsed_s": 210,
        "outcome": "Skipped",
        "rate_chunks_s": 10.451893055555555
      },
      {
        "elapsed_s": 225,
        "outcome": "Skipped",
        "rate_chunks_s": 11.056059722222223
      },
      {
        "elapsed_s": 240,
        "outcome": "Skipped",
        "rate_chunks_s": 11.850001234567902
      },
      {
        "elapsed_s": 255,
        "outcome": "Skipped",
        "rate_chunks_s": 12.222222222222223
      },
      {
        "elapsed_s": 270,
        "outcome": "Skipped",
        "rate_chunks_s": 12.592592592592593
      },
      {
        "elapsed_s": 285,
        "outcome": "Skipped",
        "rate_chunks_s": 12.592592592592593
      },
      {
        "elapsed_s": 300,
        "outcome": "Skipped",
        "rate_chunks_s": 13.703703703703704
      },
      {
        "elapsed_s": 315,
        "outcome": "Skipped",
        "rate_chunks_s": 13.703703703703704
      },
      {
        "elapsed_s": 330,
        "outcome": "Skipped",
        "rate_chunks_s": 14.074074074074074
      },
      {
        "elapsed_s": 345,
        "outcome": "Skipped",
        "rate_chunks_s": 14.074074074074074
      },
      {
        "elapsed_s": 360,
        "outcome": "Skipped",
        "rate_chunks_s": 14.814814814814815
      },
      {
        "elapsed_s": 375,
        "outcome": "Skipped",
        "rate_chunks_s": 14.814814814814815
      },
      {
        "elapsed_s": 390,
        "outcome": "Skipped",
        "rate_chunks_s": 16.296296296296298
      },
      {
        "elapsed_s": 405,
        "outcome": "Skipped",
        "rate_chunks_s": 16.296296296296298
      },
      {
        "elapsed_s": 420,
        "outcome": "Skipped",
        "rate_chunks_s": 17.40740740740741
      },
      {
        "elapsed_s": 435,
        "outcome": "Skipped",
        "rate_chunks_s": 17.40740740740741
      },
      {
        "elapsed_s": 450,
        "outcome": "Skipped",
        "rate_chunks_s": 25.348148148148148
      },
      {
        "elapsed_s": 465,
        "outcome": "Skipped",
        "rate_chunks_s": 25.348148148148148
      },
      {
        "elapsed_s": 480,
        "outcome": "Skipped",
        "rate_chunks_s": 29.185185185185187
      },
      {
        "elapsed_s": 495,
        "outcome": "Skipped",
        "rate_chunks_s": 29.185185185185187
      },
      {
        "elapsed_s": 510,
        "outcome": "Skipped",
        "rate_chunks_s": 37.85925925925926
      },
      {
        "elapsed_s": 525,
        "outcome": "Skipped",
        "rate_chunks_s": 37.85925925925926
      },
      {
        "elapsed_s": 540,
        "outcome": "Skipped",
        "rate_chunks_s": 49.72592592592593
      },
      {
        "elapsed_s": 555,
        "outcome": "Skipped",
        "rate_chunks_s": 49.72592592592593
      },
      {
        "elapsed_s": 570,
        "outcome": "Skipped",
        "rate_chunks_s": 55.470370370370375
      },
      {
        "elapsed_s": 585,
        "outcome": "Skipped",
        "rate_chunks_s": 55.470370370370375
      },
      {
        "elapsed_s": 600,
        "outcome": "Skipped",
        "rate_chunks_s": 64.33703703703705
      },
      {
        "elapsed_s": 615,
        "outcome": "Skipped",
        "rate_chunks_s": 64.33703703703705
      },
      {
        "elapsed_s": 630,
        "outcome": "Skipped",
        "rate_chunks_s": 113.51851851851852
      },
      {
        "elapsed_s": 645,
        "outcome": "Skipped",
        "rate_chunks_s": 113.51851851851852
      },
      {
        "elapsed_s": 660,
        "outcome": "Skipped",
        "rate_chunks_s": 118.46296296296296
      },
      {
        "elapsed_s": 675,
        "outcome": "Skipped",
        "rate_chunks_s": 118.46296296296296
      },
      {
        "elapsed_s": 690,
        "outcome": "Skipped",
        "rate_chunks_s": 129.33333333333331
      },
      {
        "elapsed_s": 705,
        "outcome": "Skipped",
        "rate_chunks_s": 129.33333333333331
      },
      {
        "elapsed_s": 720,
        "outcome": "Skipped",
        "rate_chunks_s": 130.4
      },
      {
        "elapsed_s": 735,
        "outcome": "Skipped",
        "rate_chunks_s": 130.4
      },
      {
        "elapsed_s": 750,
        "outcome": "Skipped",
        "rate_chunks_s": 211.52592592592595
      },
      {
        "elapsed_s": 765,
        "outcome": "Skipped",
        "rate_chunks_s": 211.52592592592595
      },
      {
        "elapsed_s": 780,
        "outcome": "Skipped",
        "rate_chunks_s": 322.6962962962963
      },
      {
        "elapsed_s": 795,
        "outcome": "Skipped",
        "rate_chunks_s": 322.6962962962963
      },
      {
        "elapsed_s": 810,
        "outcome": "Skipped",
        "rate_chunks_s": 340.77777777777777
      },
      {
        "elapsed_s": 825,
        "outcome": "Skipped",
        "rate_chunks_s": 340.77777777777777
      },
      {
        "elapsed_s": 840,
        "outcome": "Skipped",
        "rate_chunks_s": 342.35185185185185
      },
      {
        "elapsed_s": 855,
        "outcome": "Skipped",
        "rate_chunks_s": 342.35185185185185
      },
      {
        "elapsed_s": 870,
        "outcome": "Skipped",
        "rate_chunks_s": 342.4259259259259
      },
      {
        "elapsed_s": 885,
        "outcome": "Skipped",
        "rate_chunks_s": 342.4259259259259
      },
      {
        "elapsed_s": 900,
        "outcome": "Skipped",
        "rate_chunks_s": 357.56296296296296
      },
      {
        "elapsed_s": 915,
        "outcome": "Skipped",
        "rate_chunks_s": 357.56296296296296
      },
      {
        "elapsed_s": 930,
        "outcome": "Skipped",
        "rate_chunks_s": 435.31481481481484
      },
      {
        "elapsed_s": 945,
        "outcome": "Skipped",
        "rate_chunks_s": 435.31481481481484
      },
      {
        "elapsed_s": 960,
        "outcome": "Skipped",
        "rate_chunks_s": 522.1555555555556
      },
      {
        "elapsed_s": 975,
        "outcome": "Skipped",
        "rate_chunks_s": 522.1555555555556
      },
      {
        "elapsed_s": 990,
        "outcome": "Skipped",
        "rate_chunks_s": 519.9185185185186
      },
      {
        "elapsed_s": 1005,
        "outcome": "Skipped",
        "rate_chunks_s": 519.9185185185186
      },
      {
        "elapsed_s": 1020,
        "outcome": "Skipped",
        "rate_chunks_s": 438.7185185185185
      },
      {
        "elapsed_s": 1035,
        "outcome": "Skipped",
        "rate_chunks_s": 438.7185185185185
      },
      {
        "elapsed_s": 1050,
        "outcome": "Skipped",
        "rate_chunks_s": 326.8666666666667
      },
      {
        "elapsed_s": 1065,
        "outcome": "Skipped",
        "rate_chunks_s": 326.8666666666667
      },
      {
        "elapsed_s": 1080,
        "outcome": "Skipped",
        "rate_chunks_s": 301.77777777777777
      },
      {
        "elapsed_s": 1095,
        "outcome": "Skipped",
        "rate_chunks_s": 301.77777777777777
      },
      {
        "elapsed_s": 1110,
        "outcome": "Skipped",
        "rate_chunks_s": 300.63703703703703
      },
      {
        "elapsed_s": 1125,
        "outcome": "Skipped",
        "rate_chunks_s": 300.63703703703703
      },
      {
        "elapsed_s": 1140,
        "outcome": "Skipped",
        "rate_chunks_s": 295.93333333333334
      },
      {
        "elapsed_s": 1155,
        "outcome": "Skipped",
        "rate_chunks_s": 295.93333333333334
      },
      {
        "elapsed_s": 1170,
        "outcome": "Skipped",
        "rate_chunks_s": 236.93333333333334
      },
      {
        "elapsed_s": 1185,
        "outcome": "Skipped",
        "rate_chunks_s": 236.93333333333334
      },
      {
        "elapsed_s": 1200,
        "outcome": "Skipped",
        "rate_chunks_s": 159.54814814814816
      },
      {
        "elapsed_s": 1215,
        "outcome": "Skipped",
        "rate_chunks_s": 159.54814814814816
      },
      {
        "elapsed_s": 1230,
        "outcome": "Skipped",
        "rate_chunks_s": 63.25185185185185
      },
      {
        "elapsed_s": 1245,
        "outcome": "Skipped",
        "rate_chunks_s": 63.25185185185185
      },
      {
        "elapsed_s": 1260,
        "outcome": "Skipped",
        "rate_chunks_s": 60.762962962962966
      },
      {
        "elapsed_s": 1275,
        "outcome": "Skipped",
        "rate_chunks_s": 60.762962962962966
      },
      {
        "elapsed_s": 1290,
        "outcome": "Skipped",
        "rate_chunks_s": 58.800000000000004
      },
      {
        "elapsed_s": 1305,
        "outcome": "Skipped",
        "rate_chunks_s": 58.800000000000004
      },
      {
        "elapsed_s": 1320,
        "outcome": "Skipped",
        "rate_chunks_s": 56.90370370370371
      },
      {
        "elapsed_s": 1335,
        "outcome": "Skipped",
        "rate_chunks_s": 56.90370370370371
      },
      {
        "elapsed_s": 1350,
        "outcome": "Skipped",
        "rate_chunks_s": 53.11851851851852
      },
      {
        "elapsed_s": 1365,
        "outcome": "Skipped",
        "rate_chunks_s": 53.11851851851852
      },
      {
        "elapsed_s": 1380,
        "outcome": "Skipped",
        "rate_chunks_s": 52.31111111111111
      },
      {
        "elapsed_s": 1395,
        "outcome": "Skipped",
        "rate_chunks_s": 52.31111111111111
      },
      {
        "elapsed_s": 1410,
        "outcome": "Skipped",
        "rate_chunks_s": 54.974074074074075
      },
      {
        "elapsed_s": 1425,
        "outcome": "Skipped",
        "rate_chunks_s": 54.974074074074075
      },
      {
        "elapsed_s": 1440,
        "outcome": "Skipped",
        "rate_chunks_s": 55.07777777777778
      },
      {
        "elapsed_s": 1455,
        "outcome": "Skipped",
        "rate_chunks_s": 55.07777777777778
      },
      {
        "elapsed_s": 1470,
        "outcome": "Skipped",
        "rate_chunks_s": 93.94074074074075
      },
      {
        "elapsed_s": 1485,
        "outcome": "Skipped",
        "rate_chunks_s": 93.94074074074075
      },
      {
        "elapsed_s": 1500,
        "outcome": "Skipped",
        "rate_chunks_s": 95.78518518518518
      },
      {
        "elapsed_s": 1515,
        "outcome": "Skipped",
        "rate_chunks_s": 95.78518518518518
      },
      {
        "elapsed_s": 1530,
        "outcome": "Skipped",
        "rate_chunks_s": 99.20740740740742
      },
      {
        "elapsed_s": 1545,
        "outcome": "Skipped",
        "rate_chunks_s": 99.20740740740742
      },
      {
        "elapsed_s": 1560,
        "outcome": "Skipped",
        "rate_chunks_s": 107.86666666666667
      },
      {
        "elapsed_s": 1575,
        "outcome": "Skipped",
        "rate_chunks_s": 107.86666666666667
      },
      {
        "elapsed_s": 1590,
        "outcome": "Skipped",
        "rate_chunks_s": 106.63703703703705
      },
      {
        "elapsed_s": 1605,
        "outcome": "Skipped",
        "rate_chunks_s": 106.63703703703705
      },
      {
        "elapsed_s": 1620,
        "outcome": "Skipped",
        "rate_chunks_s": 110.01851851851852
      },
      {
        "elapsed_s": 1635,
        "outcome": "Skipped",
        "rate_chunks_s": 110.01851851851852
      },
      {
        "elapsed_s": 1650,
        "outcome": "Skipped",
        "rate_chunks_s": 108.33333333333334
      },
      {
        "elapsed_s": 1665,
        "outcome": "Skipped",
        "rate_chunks_s": 108.33333333333334
      },
      {
        "elapsed_s": 1680,
        "outcome": "Skipped",
        "rate_chunks_s": 106.9962962962963
      },
      {
        "elapsed_s": 1695,
        "outcome": "Skipped",
        "rate_chunks_s": 106.9962962962963
      },
      {
        "elapsed_s": 1710,
        "outcome": "Skipped",
        "rate_chunks_s": 103.12962962962963
      },
      {
        "elapsed_s": 1725,
        "outcome": "Skipped",
        "rate_chunks_s": 103.12962962962963
      },
      {
        "elapsed_s": 1740,
        "outcome": "Skipped",
        "rate_chunks_s": 59.4962962962963
      },
      {
        "elapsed_s": 1755,
        "outcome": "Skipped",
        "rate_chunks_s": 59.4962962962963
      },
      {
        "elapsed_s": 1770,
        "outcome": "Skipped",
        "rate_chunks_s": 57.03703703703704
      },
      {
        "elapsed_s": 1785,
        "outcome": "Skipped",
        "rate_chunks_s": 57.03703703703704
      },
      {
        "elapsed_s": 1800,
        "outcome": "Skipped",
        "rate_chunks_s": 54.87407407407407
      },
      {
        "elapsed_s": 1815,
        "outcome": "Skipped",
        "rate_chunks_s": 54.87407407407407
      },
      {
        "elapsed_s": 1830,
        "outcome": "Skipped",
        "rate_chunks_s": 47.76296296296296
      },
      {
        "elapsed_s": 1845,
        "outcome": "Skipped",
        "rate_chunks_s": 47.76296296296296
      },
      {
        "elapsed_s": 1860,
        "outcome": "Skipped",
        "rate_chunks_s": 51.02962962962963
      },
      {
        "elapsed_s": 1875,
        "outcome": "Skipped",
        "rate_chunks_s": 51.02962962962963
      },
      {
        "elapsed_s": 1890,
        "outcome": "Skipped",
        "rate_chunks_s": 52.02962962962963
      },
      {
        "elapsed_s": 1905,
        "outcome": "Skipped",
        "rate_chunks_s": 52.02962962962963
      },
      {
        "elapsed_s": 1920,
        "outcome": "Skipped",
        "rate_chunks_s": 50.38148148148149
      },
      {
        "elapsed_s": 1935,
        "outcome": "Skipped",
        "rate_chunks_s": 50.38148148148149
      },
      {
        "elapsed_s": 1950,
        "outcome": "Skipped",
        "rate_chunks_s": 46.63703703703704
      },
      {
        "elapsed_s": 1965,
        "outcome": "Skipped",
        "rate_chunks_s": 46.63703703703704
      },
      {
        "elapsed_s": 1980,
        "outcome": "Skipped",
        "rate_chunks_s": 47
      },
      {
        "elapsed_s": 1995,
        "outcome": "Skipped",
        "rate_chunks_s": 47
      },
      {
        "elapsed_s": 2010,
        "outcome": "Skipped",
        "rate_chunks_s": 48.44444444444444
      },
      {
        "elapsed_s": 2025,
        "outcome": "Skipped",
        "rate_chunks_s": 48.44444444444444
      },
      {
        "elapsed_s": 2040,
        "outcome": "Skipped",
        "rate_chunks_s": 50.51111111111111
      },
      {
        "elapsed_s": 2055,
        "outcome": "Skipped",
        "rate_chunks_s": 50.51111111111111
      },
      {
        "elapsed_s": 2070,
        "outcome": "Skipped",
        "rate_chunks_s": 49.592592592592595
      },
      {
        "elapsed_s": 2085,
        "outcome": "Skipped",
        "rate_chunks_s": 49.592592592592595
      },
      {
        "elapsed_s": 2100,
        "outcome": "Skipped",
        "rate_chunks_s": 46.20740740740741
      },
      {
        "elapsed_s": 2115,
        "outcome": "Skipped",
        "rate_chunks_s": 46.20740740740741
      },
      {
        "elapsed_s": 2130,
        "outcome": "Skipped",
        "rate_chunks_s": 131.66666666666669
      },
      {
        "elapsed_s": 2145,
        "outcome": "Skipped",
        "rate_chunks_s": 131.66666666666669
      },
      {
        "elapsed_s": 2160,
        "outcome": "Skipped",
        "rate_chunks_s": 128.33703703703705
      },
      {
        "elapsed_s": 2175,
        "outcome": "Skipped",
        "rate_chunks_s": 128.33703703703705
      },
      {
        "elapsed_s": 2190,
        "outcome": "Skipped",
        "rate_chunks_s": 126.02962962962962
      },
      {
        "elapsed_s": 2205,
        "outcome": "Skipped",
        "rate_chunks_s": 126.02962962962962
      },
      {
        "elapsed_s": 2220,
        "outcome": "Skipped",
        "rate_chunks_s": 129.3851851851852
      },
      {
        "elapsed_s": 2235,
        "outcome": "Skipped",
        "rate_chunks_s": 129.3851851851852
      },
      {
        "elapsed_s": 2250,
        "outcome": "Skipped",
        "rate_chunks_s": 128.2962962962963
      },
      {
        "elapsed_s": 2265,
        "outcome": "Skipped",
        "rate_chunks_s": 128.2962962962963
      },
      {
        "elapsed_s": 2280,
        "outcome": "Skipped",
        "rate_chunks_s": 123.34814814814816
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
      "title": "Elapsed time from first exported sample (s)"
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

## Evidence that NVMe itself was working

The secondary tier was not merely configured; demand traffic reached it.

| Signal | NVMe control, $N=0$ | NVMe nominal prefetch, $N=100$ |
|---|---:|---:|
| External KV prompt-token share, average | 3.368% | 3.351% |
| External prefix-cache hit rate, average | 27.321% | 27.014% |
| Async tiering stall rate, average (events/s) | 0.744 | 0.747 |
| Async tiering lookup mean latency (s) | 0.729 | 0.702 |
| Async tiering lookup P90 (s) | 1.200 | 1.214 |
| Active NVMe read throughput, average | 40.60 MB/s | 39.57 MB/s |
| Active NVMe write throughput, average | 314.27 MB/s | 308.38 MB/s |
| CPU cache occupancy, average / max | 0.659% / 3.125% | 0.615% / 3.125% |
| Preemptions | 0 | 0 |
| Allocation failures | no series | no series |

The prefetch run also exported zero-valued series for an unrelated NVMe device on another node. The table uses only the active benchmark node/device, avoiding the misleading average across active and zero-valued series.

## Code-confirmed root cause

The failure is structural, not an unlucky choice of $N$.

### 1. The hook is downstream of a terminal prefix miss

In `OffloadingConnectorScheduler._maximal_prefix_lookup()`, demand lookup walks keys in prefix order. On `HIT` it advances; on `RETRY` it continues probing so the asynchronous backend can resolve the batch; only a resolved `MISS` enters the Phase 1 hook. The hook then passes the *later* keys to prefetch:

```python
case LookupResult.MISS:
    upcoming = keys[local_idx + 1 : local_idx + 1 + n]
    prefetch(upcoming, req_context)
    break
```

### 2. Later keys are chained to the missing prefix

`hash_block_tokens()` computes each block hash from the parent block hash plus the current tokens. Thus key $K_{i+1}$ identifies the whole prefix through block $i+1$ and cannot be produced independently of $K_i$:

$
K_{i+1} = H(K_i,\; \text{tokens}_{i+1},\; \text{extra keys}).
$

The offloading connector uses these chained request hashes as filesystem keys.

### 3. The filesystem tier preserves the prefix-store invariant

Successful GPU→CPU stores are cascaded to every secondary tier. The filesystem store is append-like: it writes a missing destination atomically and leaves an existing file in place. There is no normal filesystem-tier LRU deletion. Therefore, under successful normal stores:

$
K_{i+1} \in \text{NVMe} \Longrightarrow K_i \in \text{NVMe}.
$

Taking the contrapositive:

$
K_i \notin \text{NVMe} \Longrightarrow K_{i+1},K_{i+2},\ldots \notin \text{NVMe}.
$

That is exactly where the hook runs. It asks NVMe for keys that a genuine terminal miss has already proven cannot form a stored continuation. This explains the observed 100% skipped fraction.

### 4. The metric label rules out CPU capacity as the cause

`prefetch()` records a failed promotion with the concrete tier label (for this run, `1:fs`) when the key exists in NVMe but the CPU primary tier cannot allocate space. It uses the generic `prefetch` label when `_try_promote()` returns no tier. Every exported skip used the generic label, so the run did not reach the “NVMe hit, CPU full” branch.

Reactive NVMe activity is compatible with this diagnosis: ordinary lookup found reusable chunks *before* the terminal miss. Those hits were already promoted by the demand path. Phase 1 only examined chunks *after* that boundary.

### Secondary implementation defect: asynchronous RETRY is also collapsed into skipped

`_try_promote()` recognizes only a secondary `HIT`. A filesystem `RETRY`—the normal first response while its asynchronous existence check is pending—falls through to `(False, None)` and is counted as a generic skip. `prefetch()` has no deferred retry state.

This defect can falsely classify a real key as skipped if it was not already probed by the demand scan. It is not required to explain this experiment because the first-miss/prefix invariant already makes the selected continuation unavailable, but it must be fixed before reusing `prefetch()` with any genuinely speculative candidate source.

### Why unit tests did not expose it

The manager tests manually insert arbitrary independent keys into a mock secondary tier and call `manager.prefetch()` directly. That proves promotion and accounting after a known secondary hit, but it bypasses chained prefix hashes and the scheduler's terminal-miss trigger.

The scheduler lookup tests mock `OffloadingManager`, which has no `prefetch` API, and do not assert an end-to-end post-miss promotion. There is therefore no test representing the production invariant: an append-only secondary tier populated from chained request prefixes.

## Corrective direction

The same-request “next $N$ keys after first miss” algorithm should be retired for the filesystem tier. Merely moving the hook a few lines or changing $N$ is insufficient:

- Existing secondary hits in the current request are already discovered and promoted by the normal maximal-prefix demand scan.
- Keys after the first genuine miss are not stored continuations.
- Useful speculative work must come from information outside that terminal scan—for example a predicted future request/turn, a trajectory/session association, or a secondary-tier index that proposes keys known to exist.

Any replacement should also make prefetch asynchronous: distinguish `HIT`, `MISS`, and `RETRY`; retain pending candidates across scheduler steps; and count `skipped` only after a resolved miss or explicit policy rejection.

## Recommendation

Do **not** sweep or optimize `prefetch_blocks` yet. No numeric value can compensate for a candidate set with zero secondary-tier availability.

For the next code-validation run:

1. Replace post-miss same-request read-ahead with a candidate source that proposes keys already known to exist in the secondary tier, preferably from a predicted future request/turn or trajectory/session relationship.
2. Add deferred handling for asynchronous secondary-tier `RETRY` results before counting a candidate as skipped. Preserve `prefetch_chunks=100` only as a high-signal plumbing value for the redesigned source; it is not an optimum.
3. Require `promoted > 0`, `useful + wasted > 0`, and a skip fraction below 100% before comparing latency.
4. Once the mechanism gate passes, run repeated paired cells for $N \in \{0, 32, 64, 100, 128, 256\}$ and choose the smallest value near the latency minimum with a strong useful ratio and no occupancy/allocation pressure.

## Run registry

- [NVMe control — 988f03995bb745659749110472019c6b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/988f03995bb745659749110472019c6b?workspace=benchflow) — accepted as the mechanism control.
- [NVMe nominal prefetch $N=100$ — 96d01b33a71f4f1bbb2d55a53a8aaacd](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/96d01b33a71f4f1bbb2d55a53a8aaacd?workspace=benchflow) — rejected for prefetch-effect and tuning conclusions because every candidate was skipped.

## Conclusion

The secondary-tier hypothesis is now resolved: NVMe demand lookup worked, but Phase 1 proactive promotion did not. The decisive next step is to correct when/how candidates are selected, validate with $N=100$, and only then tune $N$.