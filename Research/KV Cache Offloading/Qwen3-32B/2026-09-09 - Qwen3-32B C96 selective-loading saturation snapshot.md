---
title: "Qwen3-32B C96 selective-loading saturation snapshot"
date: "2026-09-09"
type: "experiment-note"
topic: "KV Cache Offloading"
model: "Qwen/Qwen3-32B"
concurrency: 96
tensor_parallelism: 2
replicas: 4
accelerator: "8x H100"
runtime_image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v1"
secondary_tier: "filesystem on local NVMe"
status: "valid-balanced-subset"
---

# Qwen3-32B C96 selective-loading saturation snapshot

## Result

Yes: at concurrency 96, recomputation beat external KV loading in every balanced policy pair. This statement intentionally excludes the 2,048-, 4,096-, and 16,384-token pairs affected by the repeatedly slow replica on `gjfjh`.

| Reusable prefix | Forced load | Forced recompute | Load delta vs recompute | Recompute gain vs load |
|---:|---:|---:|---:|---:|
| 512 tokens | 24.610 req/s | 33.983 req/s | -27.6% | +38.1% |
| 1,024 tokens | 18.050 req/s | 28.550 req/s | -36.8% | +58.2% |
| 8,192 tokens | 6.177 req/s | 7.787 req/s | -20.7% | +26.1% |

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Balanced C96 throughput comparisons","width":680,"height":300,"data":{"values":[{"prefix":"512","policy":"Forced load","rps":24.610},{"prefix":"512","policy":"Forced recompute","rps":33.983},{"prefix":"1,024","policy":"Forced load","rps":18.050},{"prefix":"1,024","policy":"Forced recompute","rps":28.550},{"prefix":"8,192","policy":"Forced load","rps":6.177},{"prefix":"8,192","policy":"Forced recompute","rps":7.787}]},"mark":"bar","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","8,192"],"title":"Externally reusable prefix (tokens)"},"xOffset":{"field":"policy"},"y":{"field":"rps","type":"quantitative","title":"Successful requests/s","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"prefix","type":"nominal","title":"Reusable prefix (tokens)"},{"field":"policy","type":"nominal"},{"field":"rps","type":"quantitative","title":"Successful requests/s","format":".3f"}]}}
```

TTFT shows an even larger penalty than aggregate throughput. Loading increased mean TTFT by approximately 10.2× at 512 tokens, 14.9× at 1,024 tokens, and 7.8× at 8,192 tokens.

| Reusable prefix | Forced-load mean TTFT | Forced-recompute mean TTFT | Load/recompute ratio |
|---:|---:|---:|---:|
| 512 tokens | 1,347.5 ms | 131.7 ms | 10.2× |
| 1,024 tokens | 2,401.5 ms | 161.1 ms | 14.9× |
| 8,192 tokens | 8,278.0 ms | 1,059.0 ms | 7.8× |

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Balanced C96 mean TTFT comparisons","width":680,"height":300,"data":{"values":[{"prefix":"512","policy":"Forced load","ttft_ms":1347.5},{"prefix":"512","policy":"Forced recompute","ttft_ms":131.7},{"prefix":"1,024","policy":"Forced load","ttft_ms":2401.5},{"prefix":"1,024","policy":"Forced recompute","ttft_ms":161.1},{"prefix":"8,192","policy":"Forced load","ttft_ms":8278.0},{"prefix":"8,192","policy":"Forced recompute","ttft_ms":1059.0}]},"mark":"bar","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","8,192"],"title":"Externally reusable prefix (tokens)"},"xOffset":{"field":"policy"},"y":{"field":"ttft_ms","type":"quantitative","title":"Mean TTFT (ms)","scale":{"type":"log"}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"prefix","type":"nominal","title":"Reusable prefix (tokens)"},{"field":"policy","type":"nominal"},{"field":"ttft_ms","type":"quantitative","title":"Mean TTFT (ms)","format":",.1f"}]}}
```

## Restore-path pressure

The forced-load arms were not lightly loaded. Multiple independent signals show that the secondary-tier lookup and promotion path was under heavy pressure:

| Reusable prefix | External prompt share | Mean async lookup | Mean blocked loading work | Mean vLLM waiting queue | CPU→GPU transfer | Workload-node NVMe busy |
|---:|---:|---:|---:|---:|---:|---:|
| 512 tokens | 4.10% | 0.532 s | 24.90 requests | 56.1 requests | 0.366 GiB/s | 92.1% |
| 1,024 tokens | 5.37% | 1.314 s | 44.15 requests | 61.4 requests | 0.569 GiB/s | 95.4% |
| 8,192 tokens | 8.56% | 6.551 s | 67.25 requests | 95.7 requests | 1.767 GiB/s | 100.0% |

The evidence is internally consistent:

- NVMe busy time was already above 92% for the two small-prefix cases and reached 100% at 8,192 tokens.
- Mean blocked loading work rose from about 25 to 67 requests.
- The mean waiting queue rose from about 56 to 96 requests.
- Mean asynchronous lookup time grew from 0.53 seconds to 6.55 seconds.
- CPU-to-GPU traffic increased with prefix length, showing active promotion rather than an idle loader.
- Only 4.1–8.6% of prompt tokens came from external KV in these cells. The system paid the congested restore-path cost while receiving relatively little prefill work reduction.
- Reported vLLM CPU-cache utilization remained approximately 2% or lower, so the evidence points to secondary-tier I/O and queued promotion activity rather than exhaustion of configured CPU-cache capacity.

## Interpretation

This is evidence for the selective-load use case, not evidence that recomputation always beats loading. Under this specific high-concurrency, NVMe-saturated state, the restore queue was expensive enough that recomputing won even for the balanced 8,192-token comparison. At lower pressure, the 512- and 1,024-token results were approximately neutral, demonstrating that the preferred action depends on system pressure as well as reusable-prefix length.

The current data therefore support two conclusions:

1. A static threshold can be calibrated for a defined deployment and operating range, but the earlier 1,024-token threshold is too permissive for this C96 saturation regime.
2. A future dynamic gate should combine reusable token count with restore-path pressure—especially queued lookup/promotion cost—rather than use prefix length alone.

## Measurement caveats

NVMe and transfer values are means from 15-second Prometheus samples during the C96 stage. Rate-based queries used five-minute windows, equal to the benchmark stage duration, so some preceding-stage activity can bleed into the reported means. The convergence of NVMe busy time, lookup latency, blocked work, and the waiting queue makes the heavy-pressure conclusion strong despite that limitation. GPU, PCIe, and request-level decision-to-source telemetry were unavailable.

## Source runs

- 512 forced load: [5f7b650cf85b4ceab01cfa3878ef3f0d](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/408/runs/5f7b650cf85b4ceab01cfa3878ef3f0d?workspace=benchflow)
- 512 forced recompute: [c1affdf3bc4e431f8dd668577ab78212](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/408/runs/c1affdf3bc4e431f8dd668577ab78212?workspace=benchflow)
- 1,024 forced load: [f2140ae669bb449ebd24ed03aa7a5c80](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/409/runs/f2140ae669bb449ebd24ed03aa7a5c80?workspace=benchflow)
- 1,024 forced recompute: [7287712fe61a43cd8a868d83f22d584d](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/409/runs/7287712fe61a43cd8a868d83f22d584d?workspace=benchflow)
- 8,192 forced load: [7c7c3043f80841a8afddf310b94727df](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/412/runs/7c7c3043f80841a8afddf310b94727df?workspace=benchflow)
- 8,192 forced recompute: [6df507e1623f4cbcb35d9acfcfea3b93](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/412/runs/6df507e1623f4cbcb35d9acfcfea3b93?workspace=benchflow)

Related report: [[2026-09-09 - Qwen3-32B selective-loading crossover sweep]].