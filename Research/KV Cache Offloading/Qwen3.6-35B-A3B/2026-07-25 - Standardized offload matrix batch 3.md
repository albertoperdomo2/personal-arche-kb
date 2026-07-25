---
title: Qwen3.6 35B-A3B AgentX standardized offload matrix batch 2
date: 2026-07-25
type: research-report
experiment: KV Cache Offloading
model: Qwen3.6 35B-A3B
---

# 2026-07-25 — Qwen3.6 35B-A3B AgentX standardized offload matrix (batch 2)

## Executive summary

This is a four-cell, standardized comparison of HBM-only, 64 GiB CPU tier, 64 GiB CPU plus local NVMe, and 64 GiB CPU plus tuned CephFS. All cells use the same Nemotron-3-Super-120B FP8 deployment, TP=4 on four H100s, `gpu-memory-utilization=0.85`, 131,072-token context, 200 GiB `/dev/shm`, AgentX/WEKA workload, seed 42, and a 1,800-second profiling window.

This is a conditionally valid pressure run. NVMe is the clear winner at 1.776 req/s and 21.45 s mean E2E, while CPU-only reaches 0.939 req/s. CephFS is not a clean comparison: it completed 36 sessions but emitted 14,836 `cannot store blocks` warnings. HBM-only and CephFS both remain near 94% and 85% mean KV occupancy, showing strong pressure.

### Headline metrics

| Configuration | Request throughput (req/s) | Output (tokens/s) | Mean TTFT (s) | p95 TTFT (s) | Mean E2E (s) | p95 E2E (s) | Completed sessions | Incomplete at cutoff |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| No offload | 0.683 | 438.7 | 27.333 | 39.364 | 84.918 | 229.668 | 36 | 56 |
| CPU 64 GiB | 0.939 | 635.4 | 12.964 | 32.229 | 54.752 | 165.160 | 43 | 58 |
| CPU 64 GiB + NVMe | 1.776 | 1281.4 | 1.641 | 5.362 | 21.452 | 75.775 | 82 | 58 |
| CPU 64 GiB + CephFS | 0.697 | 466.3 | 28.497 | 106.487 | 81.093 | 247.032 | 36 | 56 |

Figure 1 shows the throughput and output-rate result. Provenance: native Aiperf summaries from each MLflow run.
```vega-lite
{"$schema": "https://vega.github.io/schema/vega-lite/v5.json", "background": "white", "width": 900, "height": 340, "title": "Figure 1 \u2014 Request throughput and output rate", "data": {"values": [{"variant": "No offload", "metric": "Request throughput", "value": 0.6825860801268511}, {"variant": "No offload", "metric": "Output tokens (thousands/s)", "value": 0.4386602008269485}, {"variant": "CPU 64 GiB", "metric": "Request throughput", "value": 0.9388856042268826}, {"variant": "CPU 64 GiB", "metric": "Output tokens (thousands/s)", "value": 0.6354211633654523}, {"variant": "CPU 64 GiB + NVMe", "metric": "Request throughput", "value": 1.7762444733372567}, {"variant": "CPU 64 GiB + NVMe", "metric": "Output tokens (thousands/s)", "value": 1.281394907391739}, {"variant": "CPU 64 GiB + CephFS", "metric": "Request throughput", "value": 0.6965976544617271}, {"variant": "CPU 64 GiB + CephFS", "metric": "Output tokens (thousands/s)", "value": 0.46633519302294624}]}, "mark": "bar", "encoding": {"x": {"field": "variant", "type": "nominal", "title": "Configuration", "axis": {"labelAngle": -25}}, "y": {"field": "value", "type": "quantitative", "title": "Rate (requests/s or thousands of tokens/s)"}, "xOffset": {"field": "metric", "type": "nominal", "title": "Metric"}, "color": {"field": "variant", "type": "nominal", "scale": {"domain": ["No offload", "CPU 64 GiB", "CPU 64 GiB + NVMe", "CPU 64 GiB + CephFS"], "range": ["#1f77b4", "#ff7f0e", "#2ca02c", "#d62728"]}}, "tooltip": [{"field": "variant"}, {"field": "metric"}, {"field": "value", "format": ".3f"}]}, "config": {"view": {"stroke": null}}}
```

Figure 2 compares latency distributions; lower is better. Provenance: Aiperf request and TTFT histograms.
```vega-lite
{"$schema": "https://vega.github.io/schema/vega-lite/v5.json", "background": "white", "width": 900, "height": 340, "title": "Figure 2 \u2014 Mean and p95 latency", "data": {"values": [{"variant": "No offload", "metric": "Mean TTFT", "seconds": 27.332800179983185}, {"variant": "No offload", "metric": "p95 TTFT", "seconds": 39.363630503799996}, {"variant": "No offload", "metric": "Mean E2E", "seconds": 84.91784052739712}, {"variant": "No offload", "metric": "p95 E2E", "seconds": 229.6675907035999}, {"variant": "CPU 64 GiB", "metric": "Mean TTFT", "seconds": 12.96367306688475}, {"variant": "CPU 64 GiB", "metric": "p95 TTFT", "seconds": 32.229422584499986}, {"variant": "CPU 64 GiB", "metric": "Mean E2E", "seconds": 54.75230061847555}, {"variant": "CPU 64 GiB", "metric": "p95 E2E", "seconds": 165.1600751190499}, {"variant": "CPU 64 GiB + NVMe", "metric": "Mean TTFT", "seconds": 1.6411929252386817}, {"variant": "CPU 64 GiB + NVMe", "metric": "p95 TTFT", "seconds": 5.362442580699988}, {"variant": "CPU 64 GiB + NVMe", "metric": "Mean E2E", "seconds": 21.45192192932153}, {"variant": "CPU 64 GiB + NVMe", "metric": "p95 E2E", "seconds": 75.77478417939999}, {"variant": "CPU 64 GiB + CephFS", "metric": "Mean TTFT", "seconds": 28.496614823108406}, {"variant": "CPU 64 GiB + CephFS", "metric": "p95 TTFT", "seconds": 106.48682746659993}, {"variant": "CPU 64 GiB + CephFS", "metric": "Mean E2E", "seconds": 81.0926837871139}, {"variant": "CPU 64 GiB + CephFS", "metric": "p95 E2E", "seconds": 247.03170315079979}]}, "mark": "bar", "encoding": {"x": {"field": "variant", "type": "nominal", "title": "Configuration", "axis": {"labelAngle": -25}}, "y": {"field": "seconds", "type": "quantitative", "title": "Latency (seconds)"}, "xOffset": {"field": "metric", "type": "nominal", "title": "Latency statistic"}, "color": {"field": "metric", "type": "nominal", "title": "Statistic", "scale": {"scheme": "category10"}}, "tooltip": [{"field": "variant"}, {"field": "metric"}, {"field": "seconds", "format": ".3f"}]}, "config": {"view": {"stroke": null}}}
```

Figure 3 shows prompt-token source composition. Provenance: native vLLM prompt-source counters sampled every 15 seconds; values are integrated shares, not raw CephFS byte rates.
```vega-lite
{"$schema": "https://vega.github.io/schema/vega-lite/v5.json", "background": "white", "width": 900, "height": 340, "title": "Figure 3 \u2014 Prompt-token source composition", "data": {"values": [{"variant": "No offload", "source": "External KV transfer", "share_pct": 0.0}, {"variant": "No offload", "source": "Local cache hit", "share_pct": 0.9710534989023608}, {"variant": "No offload", "source": "Local compute", "share_pct": 99.02894650109764}, {"variant": "CPU 64 GiB", "source": "External KV transfer", "share_pct": 37.57029105599259}, {"variant": "CPU 64 GiB", "source": "Local cache hit", "share_pct": 0.6832100910601646}, {"variant": "CPU 64 GiB", "source": "Local compute", "share_pct": 61.74649885294724}, {"variant": "CPU 64 GiB + NVMe", "source": "External KV transfer", "share_pct": 86.84609177385049}, {"variant": "CPU 64 GiB + NVMe", "source": "Local cache hit", "share_pct": 4.7383604092151845}, {"variant": "CPU 64 GiB + NVMe", "source": "Local compute", "share_pct": 8.415547816934325}, {"variant": "CPU 64 GiB + CephFS", "source": "External KV transfer", "share_pct": 4.7522539278056835}, {"variant": "CPU 64 GiB + CephFS", "source": "Local cache hit", "share_pct": 1.3002499807303232}, {"variant": "CPU 64 GiB + CephFS", "source": "Local compute", "share_pct": 93.947496091464}]}, "mark": "bar", "encoding": {"x": {"field": "variant", "type": "nominal", "title": "Configuration", "axis": {"labelAngle": -25}}, "y": {"field": "share_pct", "type": "quantitative", "title": "Prompt-token share (%)", "stack": "zero"}, "color": {"field": "source", "type": "nominal", "title": "Source", "scale": {"scheme": "category10"}}, "tooltip": [{"field": "variant"}, {"field": "source"}, {"field": "share_pct", "format": ".2f"}]}, "config": {"view": {"stroke": null}}}
```

Raw vLLM running/waiting gauges sampled every 15 seconds.
```vega-lite
{"$schema": "https://vega.github.io/schema/vega-lite/v5.json", "background": "white", "width": 900, "height": 340, "title": "Figure 4 \u2014 Scheduler running and waiting pressure", "data": {"values": [{"variant": "No offload", "metric": "Running mean", "value": 41.31666666666667}, {"variant": "No offload", "metric": "Waiting mean", "value": 17.083333333333332}, {"variant": "No offload", "metric": "Running max", "value": 55.0}, {"variant": "No offload", "metric": "Waiting max", "value": 46.0}, {"variant": "CPU 64 GiB", "metric": "Running mean", "value": 40.78333333333333}, {"variant": "CPU 64 GiB", "metric": "Waiting mean", "value": 11.15}, {"variant": "CPU 64 GiB", "metric": "Running max", "value": 50.0}, {"variant": "CPU 64 GiB", "metric": "Waiting max", "value": 49.0}, {"variant": "CPU 64 GiB + NVMe", "metric": "Running mean", "value": 35.88333333333333}, {"variant": "CPU 64 GiB + NVMe", "metric": "Waiting mean", "value": 1.9166666666666667}, {"variant": "CPU 64 GiB + NVMe", "metric": "Running max", "value": 48.0}, {"variant": "CPU 64 GiB + NVMe", "metric": "Waiting max", "value": 38.0}, {"variant": "CPU 64 GiB + CephFS", "metric": "Running mean", "value": 37.38333333333333}, {"variant": "CPU 64 GiB + CephFS", "metric": "Waiting mean", "value": 18.875}, {"variant": "CPU 64 GiB + CephFS", "metric": "Running max", "value": 47.0}, {"variant": "CPU 64 GiB + CephFS", "metric": "Waiting max", "value": 40.0}]}, "mark": "bar", "encoding": {"x": {"field": "variant", "type": "nominal", "title": "Configuration", "axis": {"labelAngle": -25}}, "y": {"field": "value", "type": "quantitative", "title": "Requests"}, "xOffset": {"field": "metric", "type": "nominal", "title": "Statistic"}, "color": {"field": "metric", "type": "nominal", "scale": {"scheme": "category10"}}, "tooltip": [{"field": "variant"}, {"field": "metric"}, {"field": "value", "format": ".2f"}]}, "config": {"view": {"stroke": null}}}
```

Native vLLM KV gauge and process working-set series.
```vega-lite
{"$schema": "https://vega.github.io/schema/vega-lite/v5.json", "background": "white", "width": 900, "height": 340, "title": "Figure 5 \u2014 KV-cache occupancy and host working set", "data": {"values": [{"variant": "No offload", "metric": "KV mean (%)", "value": 94.20675922936177}, {"variant": "No offload", "metric": "Working set (GiB)", "value": 77.42883078257243}, {"variant": "CPU 64 GiB", "metric": "KV mean (%)", "value": 93.5233021203315}, {"variant": "CPU 64 GiB", "metric": "Working set (GiB)", "value": 141.90049737294515}, {"variant": "CPU 64 GiB + NVMe", "metric": "KV mean (%)", "value": 79.85954149176622}, {"variant": "CPU 64 GiB + NVMe", "metric": "Working set (GiB)", "value": 141.92483720779418}, {"variant": "CPU 64 GiB + CephFS", "metric": "KV mean (%)", "value": 85.32692928640621}, {"variant": "CPU 64 GiB + CephFS", "metric": "Working set (GiB)", "value": 142.00237003962198}]}, "mark": "bar", "encoding": {"x": {"field": "variant", "type": "nominal", "title": "Configuration", "axis": {"labelAngle": -25}}, "y": {"field": "value", "type": "quantitative", "title": "Fraction / GiB"}, "xOffset": {"field": "metric", "type": "nominal", "title": "Statistic"}, "color": {"field": "metric", "type": "nominal", "scale": {"scheme": "category10"}}, "tooltip": [{"field": "variant"}, {"field": "metric"}, {"field": "value", "format": ".2f"}]}, "config": {"view": {"stroke": null}}}
```

Figure 6 covers session and request disposition. Incomplete sessions are expected from the finite 1,800-second send window plus grace period; they are not server errors.
```vega-lite
{"$schema": "https://vega.github.io/schema/vega-lite/v5.json", "background": "white", "width": 900, "height": 340, "title": "Figure 6 \u2014 Request and session disposition", "data": {"values": [{"variant": "No offload", "metric": "Completed sessions", "value": 36}, {"variant": "No offload", "metric": "Incomplete sessions", "value": 56}, {"variant": "No offload", "metric": "Completed requests", "value": 1249}, {"variant": "No offload", "metric": "Cancelled requests", "value": 13}, {"variant": "CPU 64 GiB", "metric": "Completed sessions", "value": 43}, {"variant": "CPU 64 GiB", "metric": "Incomplete sessions", "value": 58}, {"variant": "CPU 64 GiB", "metric": "Completed requests", "value": 1718}, {"variant": "CPU 64 GiB", "metric": "Cancelled requests", "value": 7}, {"variant": "CPU 64 GiB + NVMe", "metric": "Completed sessions", "value": 82}, {"variant": "CPU 64 GiB + NVMe", "metric": "Incomplete sessions", "value": 58}, {"variant": "CPU 64 GiB + NVMe", "metric": "Completed requests", "value": 3248}, {"variant": "CPU 64 GiB + NVMe", "metric": "Cancelled requests", "value": 1}, {"variant": "CPU 64 GiB + CephFS", "metric": "Completed sessions", "value": 36}, {"variant": "CPU 64 GiB + CephFS", "metric": "Incomplete sessions", "value": 56}, {"variant": "CPU 64 GiB + CephFS", "metric": "Completed requests", "value": 1273}, {"variant": "CPU 64 GiB + CephFS", "metric": "Cancelled requests", "value": 13}]}, "mark": "bar", "encoding": {"x": {"field": "variant", "type": "nominal", "title": "Configuration", "axis": {"labelAngle": -25}}, "y": {"field": "value", "type": "quantitative", "title": "Count"}, "xOffset": {"field": "metric", "type": "nominal", "title": "Outcome"}, "color": {"field": "metric", "type": "nominal", "scale": {"scheme": "category10"}}, "tooltip": [{"field": "variant"}, {"field": "metric"}, {"field": "value"}]}, "config": {"view": {"stroke": null}}}
```

Figure 7 uses the secondary-tier telemetry that was actually present: NVMe byte rates and CephFS PVC usage. No CephFS pool/MDS byte or operation series were exported in this batch, so PVC usage is the available CephFS storage signal; it must not be interpreted as network read/write throughput.
```vega-lite
{"$schema": "https://vega.github.io/schema/vega-lite/v5.json", "background": "white", "width": 900, "height": 340, "title": "Figure 7 \u2014 Secondary-tier observability", "data": {"values": [{"variant": "CPU 64 GiB + NVMe", "metric": "NVMe read MB/s", "value": 1864.8352229451855}, {"variant": "CPU 64 GiB + NVMe", "metric": "NVMe write MB/s", "value": 288.4380497540741}, {"variant": "CPU 64 GiB + CephFS", "metric": "CephFS PVC mean (%)", "value": 7.476766374376085}, {"variant": "CPU 64 GiB + CephFS", "metric": "CephFS PVC p95 (%)", "value": 12.145233154296875}]}, "mark": "bar", "encoding": {"x": {"field": "variant", "type": "nominal", "title": "Configuration", "axis": {"labelAngle": -25}}, "y": {"field": "value", "type": "quantitative", "title": "Mean rate (MB/s) or usage (%)"}, "xOffset": {"field": "metric", "type": "nominal", "title": "Tier metric"}, "color": {"field": "metric", "type": "nominal", "scale": {"scheme": "category10"}}, "tooltip": [{"field": "variant"}, {"field": "metric"}, {"field": "value", "format": ".2f"}]}, "config": {"view": {"stroke": null}}}
```

## Deployment and observability audit

| Cell | Tier configuration | Secondary storage | `/dev/shm` | Restarts | Tracebacks / CUDA OOM |
|---|---|---|---|---:|---|
| No offload | HBM only | none | 200 GiB | 0 | 0 / 0 |
| CPU 64 GiB | TieringOffloadingSpec, 64 GiB CPU | none | 200 GiB | 0 | 0 / 0 |
| CPU 64 GiB + NVMe | TieringOffloadingSpec, 64 GiB CPU | hostPath NVMe `/mnt/nvme-kv-cache` | 200 GiB | 0 | 0 / 0 |
| CPU 64 GiB + CephFS tuned | TieringOffloadingSpec, 64 GiB CPU | PVC `vllm-kv-cache`, tuned read/write threads | 200 GiB | 0 | 0 / 0 |

The workload is heavily pressured: mean KV occupancy ranges from 79.9% (NVMe) to 94.2% (HBM-only), with all cells peaking at approximately 99–100%. NVMe sourced 86.8% of prompt tokens externally and produced 52 cannot-store warnings; CephFS sourced only 4.8% externally despite 14,836 warnings. CephFS PVC usage averaged 7.5%; pool/MDS throughput counters are unavailable.

## Interpretation and hypotheses

1. **This workload is correctly exercising the offload mechanism, but at high pressure.** The low KV occupancy and dominant local-cache/local-compute shares mean the experiment mostly measures normal serving plus a small tier-read path. To amplify tier differences, increase concurrency or prompt/session overlap until KV occupancy is near the intended operating point, then repeat with identical request count and duration.
2. **CPU64 materially improves HBM-only.** Throughput rises 37.7% and mean E2E falls 35.5%; the CPU tier is doing useful work under pressure.
3. **NVMe is the strongest cell, but one request error and 52 store warnings require caution.** NVMe has the highest measured throughput and lowest latency, but the gain could combine reduced eviction/reload stalls, scheduling variance, and the small extra external-transfer share. Use matched repeated seeds and add per-tier read/write latency and cache-hit/miss counters before attributing causality to raw device bandwidth.
4. **CephFS is functionally noisy.** It has zero request-level errors but 14,836 scheduler store warnings, so its 0.697 req/s must be treated as conditional rather than a clean storage result. The available PVC metric confirms storage use, but missing CephFS pool/MDS operation series limits a direct filesystem-throughput conclusion.
5. **All pods remained alive and there were no CUDA OOMs or tracebacks, but CephFS emitted a severe store-warning storm.**

## Run registry

| Configuration | MLflow run |
|---|---|
| No offload | [MLflow run `acff4f600fc24daa9627a27110f968f8`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/acff4f600fc24daa9627a27110f968f8?workspace=benchflow) |
| CPU 64 GiB | [MLflow run `29c7f7252f9f4e01ba665c1380cfa4ff`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/29c7f7252f9f4e01ba665c1380cfa4ff?workspace=benchflow) |
| CPU 64 GiB + NVMe | [MLflow run `3f6db383f5e642eea47635fdfd1f850f`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/3f6db383f5e642eea47635fdfd1f850f?workspace=benchflow) |
| CPU 64 GiB + CephFS | [MLflow run `b3ffddaef64e4f31885d0b22aae70280`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/b3ffddaef64e4f31885d0b22aae70280?workspace=benchflow) |

## Recommended next experiment

Hold the deployment and `/dev/shm` constant, increase concurrency in steps (32, 64, 128, 256), and collect at least three repetitions per cell. Stop or mark the point when KV occupancy reaches the desired pressure band (for example 70–90% rather than 10%). Export NVMe device latency/queue depth and CephFS client read/write/MDS counters at the same 15-second cadence. This separates workload sizing from storage-medium effects and turns the current directional result into a causal offload curve.


## Full-resolution time-series appendices

The complete native-granularity plots are decomposed into linked companion articles so the MCP payload remains reliable without dropping samples:

- [[2026-07-25 - Batch 3 no-offload request time series|No offload — request/scheduler time series]]
- [[2026-07-25 - Batch 3 no-offload prompt source time series|No offload — prompt source time series]]
- [[2026-07-25 - Batch 3 CPU64 request time series|CPU64 — request/scheduler time series]]
- [[2026-07-25 - Batch 3 CPU64 prompt source time series|CPU64 — prompt source time series]]
- [[2026-07-25 - Batch 3 CPU64 KV transfer time series|CPU64 — KV transfer time series]]
- [[2026-07-25 - Batch 3 NVMe request time series|NVMe — request/scheduler time series]]
- [[2026-07-25 - Batch 3 NVMe prompt source time series|NVMe — prompt source time series]]
- [[2026-07-25 - Batch 3 NVMe KV transfer time series|NVMe — KV transfer time series]]
- [[2026-07-25 - Batch 3 CephFS request time series|CephFS — request/scheduler time series]]
- [[2026-07-25 - Batch 3 CephFS prompt source time series|CephFS — prompt source time series]]
- [[2026-07-25 - Batch 3 CephFS KV transfer time series|CephFS — KV transfer time series]]