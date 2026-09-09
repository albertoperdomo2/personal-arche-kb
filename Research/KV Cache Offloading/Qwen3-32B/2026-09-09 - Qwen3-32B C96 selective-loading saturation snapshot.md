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

## Executive summary

At concurrency 96, recomputation beat external KV loading in every balanced comparison. Forced loading delivered 20.7–36.8% less request throughput and increased mean TTFT by 7.8–14.9×. This was not an idle-path result: workload-node NVMe busy time was 92–100%, the native vLLM waiting queue averaged 56–96 requests, derived deferred KV-lookup occupancy averaged 25–67 request-equivalents, and asynchronous lookup reached 0.53–6.55 seconds.

The defensible interpretation is narrow: when this deployment's restore path is saturated, its queued lookup and promotion cost can exceed the prefill compute saved by reuse. The result supports selective loading under pressure; it does not prove that recomputation is universally preferable or establish a static crossover threshold.

## Result

Yes: at concurrency 96, recomputation beat external KV loading in every balanced policy pair. This statement intentionally excludes the 2,048-, 4,096-, and 16,384-token pairs affected by the repeatedly slow replica on `gjfjh`.

| Reusable prefix | Forced load | Forced recompute | Load delta vs recompute | Recompute gain vs load |
|---:|---:|---:|---:|---:|
| 512 tokens | 24.610 req/s | 33.983 req/s | -27.6% | +38.1% |
| 1,024 tokens | 18.050 req/s | 28.550 req/s | -36.8% | +58.2% |
| 8,192 tokens | 6.177 req/s | 7.787 req/s | -20.7% | +26.1% |

## Decision equations

The report uses forced recomputation as the baseline. Loading delta is $\Delta_{load}(L,C)=100\left(\frac{RPS_{load}(L,C)}{RPS_{recompute}(L,C)}-1\right)$.

A negative value means recomputation completed more requests per second. A useful first-order crossover estimate is $L_{crossover}\approx\frac{T_{restore,p99}}{t_{prefill/token}}$.

Under contention, $T_{restore,p99}$ must include storage lookup, queueing, CPU staging, and CPU-to-GPU promotion. The C96 evidence shows why storage service time alone is insufficient: queueing dominated the observed restore cost.

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
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Balanced C96 mean TTFT comparisons","width":680,"height":300,"data":{"values":[{"prefix":"512","policy":"Forced load","ttft_ms":1347.5},{"prefix":"512","policy":"Forced recompute","ttft_ms":131.7},{"prefix":"1,024","policy":"Forced load","ttft_ms":2401.5},{"prefix":"1,024","policy":"Forced recompute","ttft_ms":161.1},{"prefix":"8,192","policy":"Forced load","ttft_ms":8278.0},{"prefix":"8,192","policy":"Forced recompute","ttft_ms":1059.0}]},"mark":"bar","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","8,192"],"title":"Externally reusable prefix (tokens)"},"xOffset":{"field":"policy"},"y":{"field":"ttft_ms","type":"quantitative","title":"Mean TTFT (ms)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"prefix","type":"nominal","title":"Reusable prefix (tokens)"},{"field":"policy","type":"nominal"},{"field":"ttft_ms","type":"quantitative","title":"Mean TTFT (ms)","format":",.1f"}]}}
```

## Restore-path pressure

The forced-load arms were not lightly loaded. Multiple independent signals show that the secondary-tier lookup and promotion path was under heavy pressure.

“Deferred KV-lookup occupancy” is not a native gauge or a count of unique requests. It is derived from `rate(vllm:kv_offload_lookup_async_delay_seconds_sum[5m])`, summed across pod and engine. The histogram sum accumulates request-seconds between a lookup first deferring and subsequently resolving or finishing; dividing its increase by wall-clock time produces average concurrent request-equivalents. By contrast, `vllm:num_requests_waiting` is a native instantaneous scheduler gauge. Figure 5 compares these related but distinct pressure signals; it does not claim they represent the same request population.

| Reusable prefix | External prompt share | Mean async lookup | Deferred lookup occupancy | Mean vLLM waiting queue | CPU→GPU transfer | Workload-node NVMe busy |
|---:|---:|---:|---:|---:|---:|---:|
| 512 tokens | 4.10% | 0.532 s | 24.90 request-equiv. | 56.1 requests | 0.366 GiB/s | 92.1% |
| 1,024 tokens | 5.37% | 1.314 s | 44.15 request-equiv. | 61.4 requests | 0.569 GiB/s | 95.4% |
| 8,192 tokens | 8.56% | 6.551 s | 67.25 request-equiv. | 95.7 requests | 1.767 GiB/s | 100.0% |

The evidence is internally consistent:

- NVMe busy time was already above 92% for the two small-prefix cases and reached 100% at 8,192 tokens.
- Derived deferred KV-lookup occupancy rose from about 25 to 67 request-equivalents.
- The mean waiting queue rose from about 56 to 96 requests.
- Mean asynchronous lookup time grew from 0.53 seconds to 6.55 seconds.
- CPU-to-GPU traffic increased with prefix length, showing active promotion rather than an idle loader.
- Only 4.1–8.6% of prompt tokens came from external KV in these cells. The system paid the congested restore-path cost while receiving relatively little prefill work reduction.
- Reported vLLM CPU-cache utilization remained approximately 2% or lower, so the evidence points to secondary-tier I/O and queued promotion activity rather than exhaustion of configured CPU-cache capacity.


Figure 3 expresses the same throughput result directly as the loading delta from the equation above. Every accepted C96 point is below zero; the 1,024-token restore produced the largest loss.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3. Forced-load throughput delta at C96","width":680,"height":280,"data":{"values":[{"prefix":"512","delta":-27.6},{"prefix":"1,024","delta":-36.8},{"prefix":"8,192","delta":-20.7}]},"mark":{"type":"bar","color":"#d62728"},"encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","8,192"],"title":"Externally reusable prefix (tokens)"},"y":{"field":"delta","type":"quantitative","title":"Load throughput delta vs recompute (%)","scale":{"domain":[-45,0]}},"tooltip":[{"field":"prefix","type":"nominal","title":"Reusable prefix (tokens)"},{"field":"delta","type":"quantitative","title":"Load delta (%)","format":".1f"}]}}
```

Figure 4 shows the growth in asynchronous lookup time as each requested external prefix becomes larger. At 8,192 tokens, mean lookup time was already 6.55 seconds.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4. Asynchronous lookup time under forced load at C96","width":680,"height":280,"data":{"values":[{"prefix":"512","seconds":0.532},{"prefix":"1,024","seconds":1.314},{"prefix":"8,192","seconds":6.551}]},"mark":{"type":"line","point":true,"strokeWidth":3,"color":"#ff7f0e"},"encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","8,192"],"title":"Externally reusable prefix (tokens)"},"y":{"field":"seconds","type":"quantitative","title":"Mean asynchronous lookup (s)","scale":{"zero":true}},"tooltip":[{"field":"prefix","type":"nominal","title":"Reusable prefix (tokens)"},{"field":"seconds","type":"quantitative","title":"Mean lookup (s)","format":".3f"}]}}
```

Figure 5 places the native vLLM waiting-queue gauge beside the derived deferred-lookup occupancy estimate. Both increased with prefix size, reaching approximately 96 waiting requests and 67 deferred request-equivalents at 8,192 tokens. The values are comparable as average concurrency pressure, but only the waiting queue is a directly sampled request count.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 5. Scheduler waiting versus deferred-lookup occupancy at C96","width":680,"height":290,"data":{"values":[{"prefix":"512","metric":"Native waiting gauge","request_equivalents":56.1},{"prefix":"512","metric":"Derived deferred-lookup occupancy","request_equivalents":24.90},{"prefix":"1,024","metric":"Native waiting gauge","request_equivalents":61.4},{"prefix":"1,024","metric":"Derived deferred-lookup occupancy","request_equivalents":44.15},{"prefix":"8,192","metric":"Native waiting gauge","request_equivalents":95.7},{"prefix":"8,192","metric":"Derived deferred-lookup occupancy","request_equivalents":67.25}]},"mark":"bar","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","8,192"],"title":"Externally reusable prefix (tokens)"},"xOffset":{"field":"metric"},"y":{"field":"request_equivalents","type":"quantitative","title":"Mean requests or request-equivalents","scale":{"zero":true}},"color":{"field":"metric","type":"nominal","title":"Pressure signal","scale":{"domain":["Native waiting gauge","Derived deferred-lookup occupancy"],"range":["#9467bd","#d62728"]}},"tooltip":[{"field":"prefix","type":"nominal","title":"Reusable prefix (tokens)"},{"field":"metric","type":"nominal"},{"field":"request_equivalents","type":"quantitative","title":"Mean value","format":".1f"}]}}
```

Figure 6 isolates the storage-pressure signal. Workload-node NVMe busy time was above 92% in all three accepted cells and reached 100% in the 8,192-token cell.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 6. Workload-node NVMe busy time under forced load at C96","width":680,"height":280,"data":{"values":[{"prefix":"512","percent":92.1},{"prefix":"1,024","percent":95.4},{"prefix":"8,192","percent":100.0}]},"mark":{"type":"bar","color":"#9467bd"},"encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","8,192"],"title":"Externally reusable prefix (tokens)"},"y":{"field":"percent","type":"quantitative","title":"NVMe busy time (%)","scale":{"domain":[0,100]}},"tooltip":[{"field":"prefix","type":"nominal","title":"Reusable prefix (tokens)"},{"field":"percent","type":"quantitative","title":"NVMe busy (%)","format":".1f"}]}}
```

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