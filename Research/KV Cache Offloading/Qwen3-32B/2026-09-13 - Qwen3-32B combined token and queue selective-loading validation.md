---
title: "Qwen3-32B combined token and queue selective-loading validation"
date: "2026-09-13"
type: "experiment-report"
topic: "KV Cache Offloading"
experiment: "llm-d-selective-loading-combined-policy-validation"
model: "Qwen/Qwen3-32B"
vllm_version: "0.27.0"
runtime_image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v1"
epp_image: "quay.io/rh-ee-aperdomo/llm-d-router-endpoint-picker:dev-c9117959"
tensor_parallelism: 2
replicas: 4
accelerator: "8xH100 on one node"
max_model_len: 8192
cpu_bytes: 274877906944
secondary_tier: "local NVMe filesystem"
secondary_tier_threads: "64 read, 64 write"
shared_memory: "300Gi"
workload: "GuideLLM, 256-token suffix, 128 output tokens, 300-second measured phase"
random_seed: 20260909
cache_cleanup: true
status: "complete"
---

# Qwen3-32B combined token and queue selective-loading validation

This experiment tests the configuration proposed for the selective KV plugin: a 1,024-token external-reuse threshold plus an endpoint-local waiting-queue EWMA gate that closes at 8 requests and reopens at 4. The comparison includes forced load, forced recompute, token-only selective loading at 1,024 tokens, and the combined token-plus-queue policy.

## Executive summary

The combined policy beat the better static policy in four of five workload cells and improved geometric-mean request throughput by 8.8% relative to the better static arm selected independently in each cell. It improved geometric-mean throughput by 20.6% over forced load and 12.6% over forced recompute. Forced load developed severe TTFT tails at the pressure-heavy cells, reaching 57.5 seconds p99 at 8K/C64, while the combined policy held p99 TTFT to 5.48 seconds. Total-token throughput shows the same policy ordering within each workload cell. Combined mean ITL remains between forced load and forced recompute, preserving decode responsiveness while avoiding restore-path stalls.

The queue gate did not improve on token-only selective loading in this matrix. Relative to token-only 1,024, the combined policy reduced geometric-mean throughput by 0.58%, improved geometric-mean mean TTFT by 1.14% and p99 TTFT by 6.70%, and increased mean end-to-end latency by 0.64%. The cell-level latency changes were mixed.

The archived 15-second waiting-queue samples never reached 8 in a combined-policy run; the highest observed value was 7 at 8K/C64. External-KV share and system-pressure summaries were also similar between token-only and combined runs. Sub-scrape queue spikes may have reached the EPP, but the artifacts do not establish a queue-triggered decision. The matrix validates the deployable `1024 + 8` configuration against static controls. It does not demonstrate incremental value from the queue veto.

## Validity verdict: Conditionally valid

The outcome comparison is valid for this exact same-node deployment and workload set. All 20 runs finished, error rates were at most 0.116%, each run completed thousands of requests, and the deployment profiles differed in the intended policy. NVMe cleanup was enabled between deployments.

The causal queue-gate conclusion is inconclusive. There was one sequential run per policy and cell, policy order was fixed, and no action counter identifies decisions made specifically because of the waiting queue. Prometheus sampled the queue every 15 seconds while the EPP consumed endpoint updates and maintained a two-second EWMA, so the archive cannot reconstruct every gate transition.

## Main takeaways

- Measured: combined selective loading exceeded the better static throughput in four cells and missed it by 0.69% at 4K/C32.
- Measured: combined throughput gains over the better static arm were 2.7%, 7.7%, -0.7%, 16.9%, and 18.7% from the shortest to longest tested cell.
- Measured: token-only 1,024 achieved 9.4% geometric-mean throughput improvement over the best static arm, slightly above the combined policy's 8.8%.
- Measured: combined and token-only throughput differed by no more than 2.61% in any cell and by -0.58% geometrically across the matrix.
- Measured: no archived combined-policy waiting sample reached the close threshold of 8.
- Inference: the 1,024-token evidence policy accounts for the demonstrated improvement over static policies. The present data cannot attribute additional benefit to the queue veto.
- Decision: retain `maxWaitingRequests: 8` as an experimental, deployment-specific safety gate, but do not cite this matrix as proof that the queue gate improves performance.

## Configuration

The four policy arms were:

| Policy | Load behavior |
|---|---|
| Forced load | Default vLLM external loading; no selective policy |
| Forced recompute | External loading disabled |
| Token-only 1,024 | Load when externally reusable tokens are at least 1,024 |
| Combined 1,024 + 8 | Apply the 1,024-token threshold and veto loading when the endpoint-local waiting EWMA reaches 8 |

All arms used Qwen3-32B, four TP2 replicas colocated on `diadochos-hqxzk-gpu-h100-mt46x`, a 256 GiB CPU tier, local NVMe with 64 read and 64 write threads, 300 GiB shared memory, and cache cleanup. Each GuideLLM run used a 2,048-request warmup followed by a 300-second concurrent phase, 256 uncached suffix tokens, 128 output tokens, one turn, and static seed 20260909.

## Headline throughput

| Cell | Forced load req/s | Forced recompute req/s | Token-only req/s | Combined req/s | Combined vs best static | Combined vs token-only |
|---|---:|---:|---:|---:|---:|---:|
| 1K / C64 | 25.493 | 23.827 | 26.203 | 26.187 | +2.7% | -0.1% |
| 2K / C64 | 18.290 | 19.777 | 21.670 | 21.307 | +7.7% | -1.7% |
| 4K / C32 | 11.640 | 10.510 | 11.283 | 11.560 | -0.7% | +2.5% |
| 8K / C32 | 6.790 | 7.720 | 9.107 | 9.023 | +16.9% | -0.9% |
| 8K / C64 | 7.473 | 10.187 | 12.410 | 12.087 | +18.7% | -2.6% |
| Geometric delta | | | | | **+8.8%** | **-0.58%** |

Figure 1 shows successful request throughput from the MLflow GuideLLM metrics for all 20 runs.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Request throughput by policy and workload","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","rps":25.493},{"cell":"1K / C64","policy":"Forced recompute","rps":23.827},{"cell":"1K / C64","policy":"Token-only 1024","rps":26.203},{"cell":"1K / C64","policy":"Combined 1024 + 8","rps":26.187},{"cell":"2K / C64","policy":"Forced load","rps":18.29},{"cell":"2K / C64","policy":"Forced recompute","rps":19.777},{"cell":"2K / C64","policy":"Token-only 1024","rps":21.67},{"cell":"2K / C64","policy":"Combined 1024 + 8","rps":21.307},{"cell":"4K / C32","policy":"Forced load","rps":11.64},{"cell":"4K / C32","policy":"Forced recompute","rps":10.51},{"cell":"4K / C32","policy":"Token-only 1024","rps":11.283},{"cell":"4K / C32","policy":"Combined 1024 + 8","rps":11.56},{"cell":"8K / C32","policy":"Forced load","rps":6.79},{"cell":"8K / C32","policy":"Forced recompute","rps":7.72},{"cell":"8K / C32","policy":"Token-only 1024","rps":9.107},{"cell":"8K / C32","policy":"Combined 1024 + 8","rps":9.023},{"cell":"8K / C64","policy":"Forced load","rps":7.473},{"cell":"8K / C64","policy":"Forced recompute","rps":10.187},{"cell":"8K / C64","policy":"Token-only 1024","rps":12.41},{"cell":"8K / C64","policy":"Combined 1024 + 8","rps":12.087}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"rps","type":"quantitative","title":"Successful request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Token-only 1024","Combined 1024 + 8"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"rps","type":"quantitative","title":"Throughput (requests/s)","format":".3f"}]}}
```

The combined policy's value is clearest against the best whole-run static action. Figure 2 also shows that the queue addition did not improve throughput relative to token-only 1,024.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Combined-policy throughput deltas","width":720,"height":280,"data":{"values":[{"cell":"1K / C64","comparison":"vs best static","delta":2.72},{"cell":"1K / C64","comparison":"vs token-only 1024","delta":-0.06},{"cell":"2K / C64","comparison":"vs best static","delta":7.74},{"cell":"2K / C64","comparison":"vs token-only 1024","delta":-1.68},{"cell":"4K / C32","comparison":"vs best static","delta":-0.69},{"cell":"4K / C32","comparison":"vs token-only 1024","delta":2.45},{"cell":"8K / C32","comparison":"vs best static","delta":16.88},{"cell":"8K / C32","comparison":"vs token-only 1024","delta":-0.92},{"cell":"8K / C64","comparison":"vs best static","delta":18.65},{"cell":"8K / C64","comparison":"vs token-only 1024","delta":-2.61}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"comparison"},"y":{"field":"delta","type":"quantitative","title":"Combined throughput delta (%)","scale":{"zero":true}},"color":{"field":"comparison","type":"nominal","title":"Comparison","scale":{"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"comparison","type":"nominal","title":"Comparison"},{"field":"delta","type":"quantitative","title":"Throughput delta (%)","format":".2f"}]}}
```

## Total token throughput

Total token throughput is GuideLLM's `throughput/total_tokens_per_second`: input plus output tokens completed per second. Because reusable-prefix length changes across cells, compare policies within the same cell rather than comparing absolute values between prefix sizes. Output-token throughput follows the same within-cell ordering because every response requests 128 output tokens.

| Cell | Forced load tok/s | Forced recompute tok/s | Token-only tok/s | Combined tok/s |
|---|---:|---:|---:|---:|
| 1K / C64 | 36,495 | 34,170 | **37,545** | 37,521 |
| 2K / C64 | 45,098 | 48,876 | **53,462** | 52,652 |
| 4K / C32 | **52,755** | 47,718 | 51,128 | 52,421 |
| 8K / C32 | 58,733 | 67,121 | **79,195** | 78,361 |
| 8K / C64 | 64,844 | 89,367 | **108,343** | 105,897 |

Figure 3 shows total token throughput from the same GuideLLM run summaries used for request throughput.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3. Total token throughput by policy and workload","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","total_tps":36494.5,"ttft_mean":196,"itl_mean":18.08,"itl_p99":24.03},{"cell":"1K / C64","policy":"Forced recompute","total_tps":34170,"ttft_mean":126,"itl_mean":20.06,"itl_p99":23.27},{"cell":"1K / C64","policy":"Token-only 1024","total_tps":37544.8,"ttft_mean":112.2,"itl_mean":18.26,"itl_p99":21.7},{"cell":"1K / C64","policy":"Combined 1024 + 8","total_tps":37520.6,"ttft_mean":112.6,"itl_mean":18.27,"itl_p99":21.91},{"cell":"2K / C64","policy":"Forced load","total_tps":45097.9,"ttft_mean":832.9,"itl_mean":20.75,"itl_p99":31.87},{"cell":"2K / C64","policy":"Forced recompute","total_tps":48876.5,"ttft_mean":178.5,"itl_mean":23.94,"itl_p99":29.28},{"cell":"2K / C64","policy":"Token-only 1024","total_tps":53462.2,"ttft_mean":174.8,"itl_mean":21.76,"itl_p99":26.62},{"cell":"2K / C64","policy":"Combined 1024 + 8","total_tps":52651.7,"ttft_mean":193.2,"itl_mean":22,"itl_p99":28.07},{"cell":"4K / C32","policy":"Forced load","total_tps":52754.8,"ttft_mean":233.1,"itl_mean":19.72,"itl_p99":27.09},{"cell":"4K / C32","policy":"Forced recompute","total_tps":47717.9,"ttft_mean":232,"itl_mean":22.03,"itl_p99":29.88},{"cell":"4K / C32","policy":"Token-only 1024","total_tps":51127.6,"ttft_mean":227.9,"itl_mean":20.43,"itl_p99":28.66},{"cell":"4K / C32","policy":"Combined 1024 + 8","total_tps":52420.6,"ttft_mean":198.6,"itl_mean":20.12,"itl_p99":27.1},{"cell":"8K / C32","policy":"Forced load","total_tps":58732.7,"ttft_mean":1316,"itl_mean":23.86,"itl_p99":47.23},{"cell":"8K / C32","policy":"Forced recompute","total_tps":67121.4,"ttft_mean":438,"itl_mean":28.95,"itl_p99":43.65},{"cell":"8K / C32","policy":"Token-only 1024","total_tps":79195.2,"ttft_mean":351.1,"itl_mean":24.73,"itl_p99":39.49},{"cell":"8K / C32","policy":"Combined 1024 + 8","total_tps":78360.7,"ttft_mean":365,"itl_mean":24.86,"itl_p99":39.77},{"cell":"8K / C64","policy":"Forced load","total_tps":64843.7,"ttft_mean":2584.4,"itl_mean":31.83,"itl_p99":69.39},{"cell":"8K / C64","policy":"Forced recompute","total_tps":89367.1,"ttft_mean":541.1,"itl_mean":44.65,"itl_p99":68.63},{"cell":"8K / C64","policy":"Token-only 1024","total_tps":108342.9,"ttft_mean":673.1,"itl_mean":34.91,"itl_p99":58.06},{"cell":"8K / C64","policy":"Combined 1024 + 8","total_tps":105896.8,"ttft_mean":632.3,"itl_mean":36.27,"itl_p99":59.12}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"total_tps","type":"quantitative","title":"Total throughput (tokens/s)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Token-only 1024","Combined 1024 + 8"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"total_tps","type":"quantitative","title":"Total throughput (tokens/s)","format":",.0f"}]}}
```

The combined policy exceeds both static controls in four cells and is 0.63% below forced load at 4K/C32. Token-only has the highest total-token throughput in four cells, consistent with the request-throughput result.

## Latency

| Cell | Load mean / p99 TTFT | Recompute mean / p99 TTFT | Token-only mean / p99 TTFT | Combined mean / p99 TTFT |
|---|---:|---:|---:|---:|
| 1K / C64 | 196 / 2,507 ms | 126 / 425 ms | 112 / 211 ms | **113 / 213 ms** |
| 2K / C64 | 833 / 16,180 ms | **178 / 814 ms** | 175 / 996 ms | 193 / 1,256 ms |
| 4K / C32 | 233 / 989 ms | 232 / 578 ms | 228 / 978 ms | **199 / 447 ms** |
| 8K / C32 | 1,316 / 29,864 ms | 438 / 1,701 ms | **351 / 1,423 ms** | 365 / 1,834 ms |
| 8K / C64 | 2,584 / 57,523 ms | **541 / 3,441 ms** | 673 / 5,795 ms | 632 / 5,476 ms |


Figure 4 shows mean TTFT on a linear zero-based scale. It complements the request-level p99 view by separating routine first-token delay from the severe forced-load tail.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4. Mean time to first token by policy and workload","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","total_tps":36494.5,"ttft_mean":196,"itl_mean":18.08,"itl_p99":24.03},{"cell":"1K / C64","policy":"Forced recompute","total_tps":34170,"ttft_mean":126,"itl_mean":20.06,"itl_p99":23.27},{"cell":"1K / C64","policy":"Token-only 1024","total_tps":37544.8,"ttft_mean":112.2,"itl_mean":18.26,"itl_p99":21.7},{"cell":"1K / C64","policy":"Combined 1024 + 8","total_tps":37520.6,"ttft_mean":112.6,"itl_mean":18.27,"itl_p99":21.91},{"cell":"2K / C64","policy":"Forced load","total_tps":45097.9,"ttft_mean":832.9,"itl_mean":20.75,"itl_p99":31.87},{"cell":"2K / C64","policy":"Forced recompute","total_tps":48876.5,"ttft_mean":178.5,"itl_mean":23.94,"itl_p99":29.28},{"cell":"2K / C64","policy":"Token-only 1024","total_tps":53462.2,"ttft_mean":174.8,"itl_mean":21.76,"itl_p99":26.62},{"cell":"2K / C64","policy":"Combined 1024 + 8","total_tps":52651.7,"ttft_mean":193.2,"itl_mean":22,"itl_p99":28.07},{"cell":"4K / C32","policy":"Forced load","total_tps":52754.8,"ttft_mean":233.1,"itl_mean":19.72,"itl_p99":27.09},{"cell":"4K / C32","policy":"Forced recompute","total_tps":47717.9,"ttft_mean":232,"itl_mean":22.03,"itl_p99":29.88},{"cell":"4K / C32","policy":"Token-only 1024","total_tps":51127.6,"ttft_mean":227.9,"itl_mean":20.43,"itl_p99":28.66},{"cell":"4K / C32","policy":"Combined 1024 + 8","total_tps":52420.6,"ttft_mean":198.6,"itl_mean":20.12,"itl_p99":27.1},{"cell":"8K / C32","policy":"Forced load","total_tps":58732.7,"ttft_mean":1316,"itl_mean":23.86,"itl_p99":47.23},{"cell":"8K / C32","policy":"Forced recompute","total_tps":67121.4,"ttft_mean":438,"itl_mean":28.95,"itl_p99":43.65},{"cell":"8K / C32","policy":"Token-only 1024","total_tps":79195.2,"ttft_mean":351.1,"itl_mean":24.73,"itl_p99":39.49},{"cell":"8K / C32","policy":"Combined 1024 + 8","total_tps":78360.7,"ttft_mean":365,"itl_mean":24.86,"itl_p99":39.77},{"cell":"8K / C64","policy":"Forced load","total_tps":64843.7,"ttft_mean":2584.4,"itl_mean":31.83,"itl_p99":69.39},{"cell":"8K / C64","policy":"Forced recompute","total_tps":89367.1,"ttft_mean":541.1,"itl_mean":44.65,"itl_p99":68.63},{"cell":"8K / C64","policy":"Token-only 1024","total_tps":108342.9,"ttft_mean":673.1,"itl_mean":34.91,"itl_p99":58.06},{"cell":"8K / C64","policy":"Combined 1024 + 8","total_tps":105896.8,"ttft_mean":632.3,"itl_mean":36.27,"itl_p99":59.12}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"ttft_mean","type":"quantitative","title":"Mean TTFT (ms)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Token-only 1024","Combined 1024 + 8"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"ttft_mean","type":"quantitative","title":"Mean TTFT (ms)","format":",.1f"}]}}
```

Combined selective loading has the lowest mean TTFT at 4K/C32 and remains close to token-only at 1K/C64. Forced recompute has the lowest mean at 2K/C64 and 8K/C64, while token-only has the lowest mean at 8K/C32.

Figure 5 uses a logarithmic axis because forced-load tails are an order of magnitude larger in the high-pressure cells.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 5. Request-level p99 time to first token","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","p99":2507.2},{"cell":"1K / C64","policy":"Forced recompute","p99":425},{"cell":"1K / C64","policy":"Token-only 1024","p99":211.2},{"cell":"1K / C64","policy":"Combined 1024 + 8","p99":212.8},{"cell":"2K / C64","policy":"Forced load","p99":16179.8},{"cell":"2K / C64","policy":"Forced recompute","p99":814.2},{"cell":"2K / C64","policy":"Token-only 1024","p99":996.4},{"cell":"2K / C64","policy":"Combined 1024 + 8","p99":1255.8},{"cell":"4K / C32","policy":"Forced load","p99":989.2},{"cell":"4K / C32","policy":"Forced recompute","p99":578.2},{"cell":"4K / C32","policy":"Token-only 1024","p99":977.5},{"cell":"4K / C32","policy":"Combined 1024 + 8","p99":446.8},{"cell":"8K / C32","policy":"Forced load","p99":29864.4},{"cell":"8K / C32","policy":"Forced recompute","p99":1701.2},{"cell":"8K / C32","policy":"Token-only 1024","p99":1422.5},{"cell":"8K / C32","policy":"Combined 1024 + 8","p99":1834},{"cell":"8K / C64","policy":"Forced load","p99":57523},{"cell":"8K / C64","policy":"Forced recompute","p99":3440.6},{"cell":"8K / C64","policy":"Token-only 1024","p99":5795.3},{"cell":"8K / C64","policy":"Combined 1024 + 8","p99":5476.2}]},"mark":{"type":"point","filled":true,"size":100},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"p99","type":"quantitative","title":"TTFT p99 (ms, logarithmic scale)","scale":{"type":"log"}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Token-only 1024","Combined 1024 + 8"],"scheme":"category10"}},"shape":{"field":"policy","type":"nominal","title":"Policy"},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"p99","type":"quantitative","title":"TTFT p99 (ms)","format":".1f"}]}}
```

The combined policy substantially reduces forced-load tail latency, but recompute retains a better p99 at 2K/C64 and 8K/C64. Relative to token-only, combined p99 is better at 4K/C32 and 8K/C64 and worse in the other three cells. The geometric p99 improvement of 6.7% is sensitive to the large 4K/C32 difference and should not be read as a stable distribution-wide advantage.

## Inter-token latency

ITL measures the spacing between streamed output tokens after the first token. It primarily reflects decode behavior and interference after prefill. The table reports GuideLLM request-level mean and p99 ITL.

| Cell | Load mean / p99 ITL | Recompute mean / p99 ITL | Token-only mean / p99 ITL | Combined mean / p99 ITL |
|---|---:|---:|---:|---:|
| 1K / C64 | **18.08** / 24.03 ms | 20.06 / 23.27 ms | 18.26 / **21.70** ms | 18.27 / 21.91 ms |
| 2K / C64 | **20.75** / 31.87 ms | 23.94 / 29.28 ms | 21.76 / **26.62** ms | 22.00 / 28.07 ms |
| 4K / C32 | **19.72 / 27.09 ms** | 22.03 / 29.88 ms | 20.43 / 28.66 ms | 20.12 / 27.10 ms |
| 8K / C32 | **23.86** / 47.23 ms | 28.95 / 43.65 ms | 24.73 / **39.49** ms | 24.86 / 39.77 ms |
| 8K / C64 | **31.83** / 69.39 ms | 44.65 / 68.63 ms | 34.91 / **58.06** ms | 36.27 / 59.12 ms |

Figure 6 shows mean ITL from the GuideLLM run metrics.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 6. Mean inter-token latency by policy and workload","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","total_tps":36494.5,"ttft_mean":196,"itl_mean":18.08,"itl_p99":24.03},{"cell":"1K / C64","policy":"Forced recompute","total_tps":34170,"ttft_mean":126,"itl_mean":20.06,"itl_p99":23.27},{"cell":"1K / C64","policy":"Token-only 1024","total_tps":37544.8,"ttft_mean":112.2,"itl_mean":18.26,"itl_p99":21.7},{"cell":"1K / C64","policy":"Combined 1024 + 8","total_tps":37520.6,"ttft_mean":112.6,"itl_mean":18.27,"itl_p99":21.91},{"cell":"2K / C64","policy":"Forced load","total_tps":45097.9,"ttft_mean":832.9,"itl_mean":20.75,"itl_p99":31.87},{"cell":"2K / C64","policy":"Forced recompute","total_tps":48876.5,"ttft_mean":178.5,"itl_mean":23.94,"itl_p99":29.28},{"cell":"2K / C64","policy":"Token-only 1024","total_tps":53462.2,"ttft_mean":174.8,"itl_mean":21.76,"itl_p99":26.62},{"cell":"2K / C64","policy":"Combined 1024 + 8","total_tps":52651.7,"ttft_mean":193.2,"itl_mean":22,"itl_p99":28.07},{"cell":"4K / C32","policy":"Forced load","total_tps":52754.8,"ttft_mean":233.1,"itl_mean":19.72,"itl_p99":27.09},{"cell":"4K / C32","policy":"Forced recompute","total_tps":47717.9,"ttft_mean":232,"itl_mean":22.03,"itl_p99":29.88},{"cell":"4K / C32","policy":"Token-only 1024","total_tps":51127.6,"ttft_mean":227.9,"itl_mean":20.43,"itl_p99":28.66},{"cell":"4K / C32","policy":"Combined 1024 + 8","total_tps":52420.6,"ttft_mean":198.6,"itl_mean":20.12,"itl_p99":27.1},{"cell":"8K / C32","policy":"Forced load","total_tps":58732.7,"ttft_mean":1316,"itl_mean":23.86,"itl_p99":47.23},{"cell":"8K / C32","policy":"Forced recompute","total_tps":67121.4,"ttft_mean":438,"itl_mean":28.95,"itl_p99":43.65},{"cell":"8K / C32","policy":"Token-only 1024","total_tps":79195.2,"ttft_mean":351.1,"itl_mean":24.73,"itl_p99":39.49},{"cell":"8K / C32","policy":"Combined 1024 + 8","total_tps":78360.7,"ttft_mean":365,"itl_mean":24.86,"itl_p99":39.77},{"cell":"8K / C64","policy":"Forced load","total_tps":64843.7,"ttft_mean":2584.4,"itl_mean":31.83,"itl_p99":69.39},{"cell":"8K / C64","policy":"Forced recompute","total_tps":89367.1,"ttft_mean":541.1,"itl_mean":44.65,"itl_p99":68.63},{"cell":"8K / C64","policy":"Token-only 1024","total_tps":108342.9,"ttft_mean":673.1,"itl_mean":34.91,"itl_p99":58.06},{"cell":"8K / C64","policy":"Combined 1024 + 8","total_tps":105896.8,"ttft_mean":632.3,"itl_mean":36.27,"itl_p99":59.12}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"itl_mean","type":"quantitative","title":"Mean ITL (ms/token)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Token-only 1024","Combined 1024 + 8"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"itl_mean","type":"quantitative","title":"Mean ITL (ms/token)","format":",.2f"}]}}
```

Forced load has the lowest mean ITL in all five cells, but its TTFT and overall throughput collapse under restore pressure. Combined selective loading reduces geometric-mean ITL by 11.8% relative to forced recompute while remaining 5.4% above forced load and 0.81% above token-only. This is the expected tradeoff: selective recomputation consumes more GPU prefill work than always loading, but it avoids restore-path stalls that dominate TTFT and completed throughput.

## Queue and mechanism evidence

The source metric for the policy is `vllm:num_requests_waiting`. The MLflow summary archives it at 15-second cadence. The EPP receives endpoint metric updates more frequently and applies a two-second EWMA, so the archive is a coarse mechanism check.

| Cell | Token-only queue avg / max | Combined queue avg / max | Token-only external KV share avg | Combined external KV share avg |
|---|---:|---:|---:|---:|
| 1K / C64 | 0.071 / 3 | 0.032 / 1 | 13.67% | 14.00% |
| 2K / C64 | 0.141 / 2 | 0.089 / 3 | 6.58% | 6.54% |
| 4K / C32 | 0.039 / 3 | 0.052 / 3 | 6.12% | 6.75% |
| 8K / C32 | 0.089 / 5 | 0.147 / 5 | 8.67% | 7.82% |
| 8K / C64 | 0.821 / 6 | 0.643 / 7 | 8.73% | 7.81% |

Figure 7 compares the maximum archived queue observation with the configured close threshold.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 7. Archived waiting-queue maxima versus gate threshold","width":720,"height":270,"data":{"values":[{"cell":"1K / C64","policy":"Token-only 1024","max_waiting":3},{"cell":"1K / C64","policy":"Combined 1024 + 8","max_waiting":1},{"cell":"2K / C64","policy":"Token-only 1024","max_waiting":2},{"cell":"2K / C64","policy":"Combined 1024 + 8","max_waiting":3},{"cell":"4K / C32","policy":"Token-only 1024","max_waiting":3},{"cell":"4K / C32","policy":"Combined 1024 + 8","max_waiting":3},{"cell":"8K / C32","policy":"Token-only 1024","max_waiting":5},{"cell":"8K / C32","policy":"Combined 1024 + 8","max_waiting":5},{"cell":"8K / C64","policy":"Token-only 1024","max_waiting":6},{"cell":"8K / C64","policy":"Combined 1024 + 8","max_waiting":7}]},"layer":[{"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"max_waiting","type":"quantitative","title":"Maximum waiting requests (requests)","scale":{"zero":true,"domain":[0,9]}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"max_waiting","type":"quantitative","title":"Maximum waiting requests"}]}},{"mark":{"type":"rule","color":"#d62728","strokeWidth":2,"strokeDash":[6,4]},"encoding":{"y":{"datum":8}}}]}
```

No bar reaches the red close-threshold rule. At 8K/C64, token-only and combined also had similar average NVMe busy time, 0.281 versus 0.287 seconds per second, similar node NVMe throughput, 631 versus 660 MB/s, and similar average CPU use, 168% versus 170% in the collected container query. These are aggregate summaries rather than proof of identical instantaneous conditions.

The archived `kv_offload_lookup_async_blocked_requests_by_engine` query is not a request count. It computes the rate of the lookup delay-seconds sum, representing blocked lookup work in seconds per second. At 8K/C64 it averaged 0.590 for token-only and 0.487 for combined. This difference does not establish a queue-veto action because the external-KV share, queue state, and other resource summaries remain close and the policy emits no reason-labeled action artifact.

## Conclusion

The `minExternalReusableTokens: 1024` selective policy is the demonstrated source of value in this matrix. Both selective arms materially outperform either whole-run static policy in the pressure-heavy cells. The exact deployable `1024 + 8` configuration therefore has useful PR-level outcome data: it improves geometric-mean throughput by 8.8% over the better static arm while avoiding forced-load tail collapse.

The queue threshold of 8 remains an experimental safety gate. This run does not show an incremental throughput advantage over token-only 1,024, and the sampled queue did not cross the threshold. The result should be described as "combined configuration validated against static controls," not "queue gate proven beneficial."

If direct queue-gate evidence is required, the smallest follow-up is token-only versus combined at 2K/C96 and 8K/C96 or C128, with the same node placement and cache cleanup. The workload must sustain waiting EWMA above 8. Reason-labeled load/recompute counters remain the preferred way to establish that the gate actually changes requests.

## MLflow run registry

| Policy | Cell | Run |
|---|---|---|
| Forced load | 1K / C64 | [73981e61](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/73981e61e2b342efb66e732ff928afd5?workspace=benchflow) |
| Forced load | 2K / C64 | [04693382](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/04693382ccdf4dc28a090be35369dd15?workspace=benchflow) |
| Forced load | 4K / C32 | [9d4e62ee](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/9d4e62ee0ee645a78628ea1a84d2c07c?workspace=benchflow) |
| Forced load | 8K / C32 | [a20aa267](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/a20aa267e97b4d6181a9e14fe3fede0e?workspace=benchflow) |
| Forced load | 8K / C64 | [f43a9234](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/f43a9234b9fb4fe596f2a1b1d73cf0c7?workspace=benchflow) |
| Forced recompute | 1K / C64 | [27f549e4](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/27f549e42b014b4f82ed3637321e2d04?workspace=benchflow) |
| Forced recompute | 2K / C64 | [d3a42f9d](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/d3a42f9de72648b09a7e94afce612a92?workspace=benchflow) |
| Forced recompute | 4K / C32 | [2d69d2bd](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/2d69d2bd8235471bb41d1e57a77ecf58?workspace=benchflow) |
| Forced recompute | 8K / C32 | [1954a3b5](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/1954a3b57f624394acc251b354ce5a5d?workspace=benchflow) |
| Forced recompute | 8K / C64 | [ac7789a8](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/ac7789a826f7472eb8f55c559de3bee2?workspace=benchflow) |
| Token-only 1,024 | 1K / C64 | [95c05bd0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/95c05bd054ca4a3793a524a8e2f6bd4a?workspace=benchflow) |
| Token-only 1,024 | 2K / C64 | [2b70bc6f](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/2b70bc6f6f70431a814e692b1fc537ee?workspace=benchflow) |
| Token-only 1,024 | 4K / C32 | [163453b9](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/163453b96d2c4e4b80afc3a28284368b?workspace=benchflow) |
| Token-only 1,024 | 8K / C32 | [ec8c39c0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/ec8c39c0627146f8a18ac796d9e7e5f1?workspace=benchflow) |
| Token-only 1,024 | 8K / C64 | [24665fec](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/24665fec4da649e0bc5c6a24584576f1?workspace=benchflow) |
| Combined 1,024 + 8 | 1K / C64 | [10b2b918](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/10b2b918f4aa46c58994a3bf5af42001?workspace=benchflow) |
| Combined 1,024 + 8 | 2K / C64 | [421a5b65](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/421a5b65bad145a9ba324e2ae872a823?workspace=benchflow) |
| Combined 1,024 + 8 | 4K / C32 | [86c7f15b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/86c7f15b915b49a3a2127adeac8f4633?workspace=benchflow) |
| Combined 1,024 + 8 | 8K / C32 | [ceccec62](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/ceccec62d1b6453e8c4e4f909cfde803?workspace=benchflow) |
| Combined 1,024 + 8 | 8K / C64 | [6ab2c9ef](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/6ab2c9ef0c4a474598b2bd74040deefb?workspace=benchflow) |