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
selected_runs: 15
comparison:
  - "Forced load/current behavior"
  - "Forced recompute diagnostic control"
  - "Combined 1024 + 8 proposed policy"
status: "complete"
---

# Qwen3-32B combined token and queue selective-loading validation

This experiment validates the exact proposed selective-loading configuration—`minExternalReusableTokens: 1024` plus an endpoint-local waiting-queue EWMA gate that closes at 8 requests and reopens at 4—against two static controls: forced load/current behavior and forced recompute. The report treats the combined policy as one deployable unit; it does not attempt to isolate the incremental contribution of either gate component.

## Executive summary

The combined `1024 + 8` policy beat the better static control in four of five workload cells and improved geometric-mean request throughput by 8.8% relative to the better static action selected independently in each cell. It improved geometric-mean request and output-token throughput by 20.6% over forced load and 12.6% over forced recompute. Forced load developed severe TTFT tails at the pressure-heavy cells, reaching 57.5 seconds p99 at 8K/C64, while the combined policy held p99 TTFT to 5.48 seconds. Total-token throughput shows the same policy ordering within each workload cell.

The combined policy also preserved decode responsiveness: mean ITL remained between forced load and forced recompute in every cell. This is the intended system-level tradeoff—avoid costly restore-path stalls without paying the full decode-interference cost of recomputing every reusable prefix.

The archived 15-second waiting-queue samples never reached the configured close threshold of 8 in a combined-policy run; the highest observed value was 7 at 8K/C64. The EPP consumed endpoint updates more frequently and maintained a two-second EWMA, so sub-scrape crossings are possible, but there is no reason-labeled policy-action artifact showing that the queue gate changed any request. Therefore this matrix validates the exact `1024 + 8` configuration against static controls; it does not prove that the queue gate fired or quantify its incremental benefit.

## Validity verdict: Conditionally valid

The outcome comparison is valid for this exact same-node deployment and five-cell workload matrix. All 15 selected runs finished, error rates were at most 0.116%, each run completed thousands of requests, the deployment profiles differed in the intended policy, and NVMe cleanup was enabled between deployments.

Mechanism attribution to the queue component is inconclusive. There was one sequential run per policy and cell, policy order was fixed, no action counter labels decisions specifically caused by waiting-queue pressure, and Prometheus archived queue state only every 15 seconds. The report consequently evaluates the combined configuration as a whole and makes no queue-gate-firing claim.

## Main takeaways

- Measured: combined selective loading exceeded the better static request throughput in four cells and missed it by 0.69% at 4K/C32.
- Measured: combined request-throughput gains over the better static arm were 2.7%, 7.7%, -0.7%, 16.9%, and 18.7% from the shortest to longest tested cell.
- Measured: geometric-mean request and output-token throughput improved by 20.6% over forced load and 12.6% over forced recompute.
- Measured: forced-load p99 TTFT reached 29.9 seconds at 8K/C32 and 57.5 seconds at 8K/C64; the combined policy reduced those tails to 1.83 and 5.48 seconds.
- Measured: no archived 15-second combined-policy waiting sample reached the close threshold of 8; the maximum was 7.
- Inference: the exact `1024 + 8` policy is a useful proposed configuration for this deployment, but this matrix cannot determine which internal gate component produced any individual decision.
- Decision: present the result as combined-policy validation against static controls, retain 8 as a deployment-specific experimental safety threshold, and require reason-labeled action telemetry before claiming that the queue veto fired.

## Configuration

The three selected policy arms were:

| Policy | Role | Load behavior |
|---|---|---|
| Forced load/current behavior | Static production-behavior control | Default vLLM external loading; no selective policy |
| Forced recompute | Diagnostic static control | External loading disabled |
| Combined 1,024 + 8 | Proposed policy | Load only when externally reusable tokens are at least 1,024 and the endpoint-local waiting EWMA is below 8; reopen after falling to 4 |

All arms used Qwen3-32B, four TP2 replicas colocated on `diadochos-hqxzk-gpu-h100-mt46x`, a 256 GiB CPU tier, local NVMe with 64 read and 64 write threads, 300 GiB shared memory, and cache cleanup. Each GuideLLM run used a 2,048-request warmup followed by a 300-second concurrent phase, 256 uncached suffix tokens, 128 output tokens, one turn, and static seed 20260909.

## Headline request throughput

| Cell | Forced load req/s | Forced recompute req/s | Combined req/s | Combined vs best static |
|---|---:|---:|---:|---:|
| 1K / C64 | 25.493 | 23.827 | 26.187 | +2.7% |
| 2K / C64 | 18.290 | 19.777 | 21.307 | +7.7% |
| 4K / C32 | 11.640 | 10.510 | 11.560 | -0.7% |
| 8K / C32 | 6.790 | 7.720 | 9.023 | +16.9% |
| 8K / C64 | 7.473 | 10.187 | 12.087 | +18.7% |
| Geometric delta | | | | **+8.8%** |

Figure 1 shows successful request throughput from the MLflow GuideLLM metrics for all 15 selected runs.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Request throughput for the proposed policy and static controls","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","rps":25.493},{"cell":"1K / C64","policy":"Forced recompute","rps":23.827},{"cell":"1K / C64","policy":"Combined 1024 + 8","rps":26.187},{"cell":"2K / C64","policy":"Forced load","rps":18.29},{"cell":"2K / C64","policy":"Forced recompute","rps":19.777},{"cell":"2K / C64","policy":"Combined 1024 + 8","rps":21.307},{"cell":"4K / C32","policy":"Forced load","rps":11.64},{"cell":"4K / C32","policy":"Forced recompute","rps":10.51},{"cell":"4K / C32","policy":"Combined 1024 + 8","rps":11.56},{"cell":"8K / C32","policy":"Forced load","rps":6.79},{"cell":"8K / C32","policy":"Forced recompute","rps":7.72},{"cell":"8K / C32","policy":"Combined 1024 + 8","rps":9.023},{"cell":"8K / C64","policy":"Forced load","rps":7.473},{"cell":"8K / C64","policy":"Forced recompute","rps":10.187},{"cell":"8K / C64","policy":"Combined 1024 + 8","rps":12.087}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"rps","type":"quantitative","title":"Successful request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Combined 1024 + 8"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"rps","type":"quantitative","title":"Throughput (requests/s)","format":".3f"}]}}
```

The combined policy has the highest request throughput in four cells. At 4K/C32 it trails forced load by 0.69%, while still exceeding forced recompute.

Figure 2 isolates the proposed policy's request-throughput delta from the better static control in each workload cell.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Proposed-policy request-throughput delta versus the better static control","width":720,"height":280,"data":{"values":[{"cell":"1K / C64","delta":2.72},{"cell":"2K / C64","delta":7.74},{"cell":"4K / C32","delta":-0.69},{"cell":"8K / C32","delta":16.88},{"cell":"8K / C64","delta":18.65}]},"layer":[{"mark":{"type":"rule","color":"#666666","strokeDash":[4,4]},"encoding":{"y":{"datum":0}}},{"mark":{"type":"bar","color":"#1f77b4"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"y":{"field":"delta","type":"quantitative","title":"Combined throughput delta (%)","scale":{"zero":true}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"delta","type":"quantitative","title":"Throughput delta (%)","format":".2f"}]}}]}
```

The benefit grows in the two 8K cells, where forced loading incurs the greatest restore pressure. The single negative cell is small enough that repetition is required before treating it as a stable regression.

## Total token throughput

Total token throughput is GuideLLM's `throughput/total_tokens_per_second`: input plus output tokens completed per second. Because reusable-prefix length changes across cells, comparisons are valid within a cell rather than across absolute prefix sizes.

| Cell | Forced load tok/s | Forced recompute tok/s | Combined tok/s |
|---|---:|---:|---:|
| 1K / C64 | 36,495 | 34,170 | **37,521** |
| 2K / C64 | 45,098 | 48,876 | **52,652** |
| 4K / C32 | **52,755** | 47,718 | 52,421 |
| 8K / C32 | 58,733 | 67,121 | **78,361** |
| 8K / C64 | 64,844 | 89,367 | **105,897** |

Figure 3 shows total token throughput from the same 15 GuideLLM summaries used for request throughput.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3. Total token throughput for the proposed policy and static controls","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","total_tps":36494.5},{"cell":"1K / C64","policy":"Forced recompute","total_tps":34170},{"cell":"1K / C64","policy":"Combined 1024 + 8","total_tps":37520.6},{"cell":"2K / C64","policy":"Forced load","total_tps":45097.9},{"cell":"2K / C64","policy":"Forced recompute","total_tps":48876.5},{"cell":"2K / C64","policy":"Combined 1024 + 8","total_tps":52651.7},{"cell":"4K / C32","policy":"Forced load","total_tps":52754.8},{"cell":"4K / C32","policy":"Forced recompute","total_tps":47717.9},{"cell":"4K / C32","policy":"Combined 1024 + 8","total_tps":52420.6},{"cell":"8K / C32","policy":"Forced load","total_tps":58732.7},{"cell":"8K / C32","policy":"Forced recompute","total_tps":67121.4},{"cell":"8K / C32","policy":"Combined 1024 + 8","total_tps":78360.7},{"cell":"8K / C64","policy":"Forced load","total_tps":64843.7},{"cell":"8K / C64","policy":"Forced recompute","total_tps":89367.1},{"cell":"8K / C64","policy":"Combined 1024 + 8","total_tps":105896.8}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"total_tps","type":"quantitative","title":"Total throughput (tokens/s)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Combined 1024 + 8"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"total_tps","type":"quantitative","title":"Total throughput (tokens/s)","format":",.0f"}]}}
```

The combined policy exceeds both static controls in four cells and is 0.63% below forced load at 4K/C32. This mirrors request throughput because request shape is fixed within each cell.

## Output-token throughput

Every response requested 128 output tokens, so generated-token throughput is the most direct serving-capacity view for the PR-level policy comparison.

| Cell | Forced load output tok/s | Forced recompute output tok/s | Combined output tok/s |
|---|---:|---:|---:|
| 1K / C64 | 3,276.8 | 3,065.4 | **3,367.2** |
| 2K / C64 | 2,350.8 | 2,545.6 | **2,744.4** |
| 4K / C32 | **1,495.9** | 1,352.1 | 1,487.1 |
| 8K / C32 | 873.1 | 995.0 | **1,162.2** |
| 8K / C64 | 962.5 | 1,319.1 | **1,565.6** |

Figure 4 shows output-token throughput for all selected runs.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4. Output-token throughput for the proposed policy and static controls","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","output_tps":3276.8},{"cell":"1K / C64","policy":"Forced recompute","output_tps":3065.4},{"cell":"1K / C64","policy":"Combined 1024 + 8","output_tps":3367.2},{"cell":"2K / C64","policy":"Forced load","output_tps":2350.8},{"cell":"2K / C64","policy":"Forced recompute","output_tps":2545.6},{"cell":"2K / C64","policy":"Combined 1024 + 8","output_tps":2744.4},{"cell":"4K / C32","policy":"Forced load","output_tps":1495.9},{"cell":"4K / C32","policy":"Forced recompute","output_tps":1352.1},{"cell":"4K / C32","policy":"Combined 1024 + 8","output_tps":1487.1},{"cell":"8K / C32","policy":"Forced load","output_tps":873.1},{"cell":"8K / C32","policy":"Forced recompute","output_tps":995},{"cell":"8K / C32","policy":"Combined 1024 + 8","output_tps":1162.2},{"cell":"8K / C64","policy":"Forced load","output_tps":962.5},{"cell":"8K / C64","policy":"Forced recompute","output_tps":1319.1},{"cell":"8K / C64","policy":"Combined 1024 + 8","output_tps":1565.6}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"output_tps","type":"quantitative","title":"Output throughput (tokens/s)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Combined 1024 + 8"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"output_tps","type":"quantitative","title":"Output throughput (tokens/s)","format":",.1f"}]}}
```

The combined policy improves geometric-mean output-token throughput by 20.6% over forced load and 12.6% over forced recompute. At 8K/C64 it produces 1,565.6 output tokens/s, compared with 962.5 for forced load and 1,319.1 for forced recompute.

## Time to first token

| Cell | Load mean / p99 TTFT | Recompute mean / p99 TTFT | Combined mean / p99 TTFT |
|---|---:|---:|---:|
| 1K / C64 | 196 / 2,507 ms | 126 / 425 ms | **113 / 213 ms** |
| 2K / C64 | 833 / 16,180 ms | **178 / 814 ms** | 193 / 1,256 ms |
| 4K / C32 | 233 / 989 ms | 232 / 578 ms | **199 / 447 ms** |
| 8K / C32 | 1,316 / 29,864 ms | **438 / 1,701 ms** | 365 / 1,834 ms |
| 8K / C64 | 2,584 / 57,523 ms | **541 / 3,441 ms** | 632 / 5,476 ms |

Figure 5 shows mean TTFT on a linear zero-based scale, separating routine first-token delay from the severe forced-load tail.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 5. Mean TTFT for the proposed policy and static controls","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","ttft_mean":196},{"cell":"1K / C64","policy":"Forced recompute","ttft_mean":126},{"cell":"1K / C64","policy":"Combined 1024 + 8","ttft_mean":112.6},{"cell":"2K / C64","policy":"Forced load","ttft_mean":832.9},{"cell":"2K / C64","policy":"Forced recompute","ttft_mean":178.5},{"cell":"2K / C64","policy":"Combined 1024 + 8","ttft_mean":193.2},{"cell":"4K / C32","policy":"Forced load","ttft_mean":233.1},{"cell":"4K / C32","policy":"Forced recompute","ttft_mean":232},{"cell":"4K / C32","policy":"Combined 1024 + 8","ttft_mean":198.6},{"cell":"8K / C32","policy":"Forced load","ttft_mean":1316},{"cell":"8K / C32","policy":"Forced recompute","ttft_mean":438},{"cell":"8K / C32","policy":"Combined 1024 + 8","ttft_mean":365},{"cell":"8K / C64","policy":"Forced load","ttft_mean":2584.4},{"cell":"8K / C64","policy":"Forced recompute","ttft_mean":541.1},{"cell":"8K / C64","policy":"Combined 1024 + 8","ttft_mean":632.3}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"ttft_mean","type":"quantitative","title":"Mean TTFT (ms)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Combined 1024 + 8"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"ttft_mean","type":"quantitative","title":"Mean TTFT (ms)","format":",.1f"}]}}
```

The combined policy reduces mean TTFT relative to forced load in all five cells. It also beats forced recompute in three cells; recompute remains lower at 2K/C64 and 8K/C64.

Figure 6 uses a logarithmic axis because forced-load tails are an order of magnitude larger in the high-pressure cells.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 6. Request-level p99 TTFT for the proposed policy and static controls","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","p99":2507.2},{"cell":"1K / C64","policy":"Forced recompute","p99":425},{"cell":"1K / C64","policy":"Combined 1024 + 8","p99":212.8},{"cell":"2K / C64","policy":"Forced load","p99":16179.8},{"cell":"2K / C64","policy":"Forced recompute","p99":814.2},{"cell":"2K / C64","policy":"Combined 1024 + 8","p99":1255.8},{"cell":"4K / C32","policy":"Forced load","p99":989.2},{"cell":"4K / C32","policy":"Forced recompute","p99":578.2},{"cell":"4K / C32","policy":"Combined 1024 + 8","p99":446.8},{"cell":"8K / C32","policy":"Forced load","p99":29864.4},{"cell":"8K / C32","policy":"Forced recompute","p99":1701.2},{"cell":"8K / C32","policy":"Combined 1024 + 8","p99":1834},{"cell":"8K / C64","policy":"Forced load","p99":57523},{"cell":"8K / C64","policy":"Forced recompute","p99":3440.6},{"cell":"8K / C64","policy":"Combined 1024 + 8","p99":5476.2}]},"mark":{"type":"point","filled":true,"size":100},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"p99","type":"quantitative","title":"TTFT p99 (ms, logarithmic scale)","scale":{"type":"log"}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Combined 1024 + 8"],"scheme":"category10"}},"shape":{"field":"policy","type":"nominal","title":"Policy"},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"p99","type":"quantitative","title":"TTFT p99 (ms)","format":".1f"}]}}
```

The combined policy substantially reduces forced-load p99 TTFT in every cell. It beats forced recompute at 1K/C64 and 4K/C32, while recompute has lower p99 at 2K/C64, 8K/C32, and 8K/C64. The policy is therefore valuable as a throughput-and-tail compromise rather than a universal TTFT minimum.

## Inter-token latency

ITL measures spacing between streamed output tokens after the first token and primarily reflects decode behavior and interference after prefill.

| Cell | Load mean / p99 ITL | Recompute mean / p99 ITL | Combined mean / p99 ITL |
|---|---:|---:|---:|
| 1K / C64 | **18.08** / 24.03 ms | 20.06 / 23.27 ms | 18.27 / **21.91 ms** |
| 2K / C64 | **20.75** / 31.87 ms | 23.94 / 29.28 ms | 22.00 / **28.07 ms** |
| 4K / C32 | **19.72 / 27.09 ms** | 22.03 / 29.88 ms | 20.12 / 27.10 ms |
| 8K / C32 | **23.86** / 47.23 ms | 28.95 / 43.65 ms | 24.86 / **39.77 ms** |
| 8K / C64 | **31.83** / 69.39 ms | 44.65 / 68.63 ms | 36.27 / **59.12 ms** |

Figure 7 shows mean ITL from the 15 GuideLLM run summaries.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 7. Mean ITL for the proposed policy and static controls","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","itl_mean":18.08},{"cell":"1K / C64","policy":"Forced recompute","itl_mean":20.06},{"cell":"1K / C64","policy":"Combined 1024 + 8","itl_mean":18.27},{"cell":"2K / C64","policy":"Forced load","itl_mean":20.75},{"cell":"2K / C64","policy":"Forced recompute","itl_mean":23.94},{"cell":"2K / C64","policy":"Combined 1024 + 8","itl_mean":22},{"cell":"4K / C32","policy":"Forced load","itl_mean":19.72},{"cell":"4K / C32","policy":"Forced recompute","itl_mean":22.03},{"cell":"4K / C32","policy":"Combined 1024 + 8","itl_mean":20.12},{"cell":"8K / C32","policy":"Forced load","itl_mean":23.86},{"cell":"8K / C32","policy":"Forced recompute","itl_mean":28.95},{"cell":"8K / C32","policy":"Combined 1024 + 8","itl_mean":24.86},{"cell":"8K / C64","policy":"Forced load","itl_mean":31.83},{"cell":"8K / C64","policy":"Forced recompute","itl_mean":44.65},{"cell":"8K / C64","policy":"Combined 1024 + 8","itl_mean":36.27}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"itl_mean","type":"quantitative","title":"Mean ITL (ms/token)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Combined 1024 + 8"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"itl_mean","type":"quantitative","title":"Mean ITL (ms/token)","format":",.2f"}]}}
```

Forced load has the lowest mean ITL in all five cells, but its TTFT and overall throughput collapse under restore pressure. Combined selective loading reduces geometric-mean ITL by 11.8% relative to forced recompute while remaining 5.4% above forced load. This is the expected tradeoff: selective recomputation consumes more GPU prefill work than always loading, but it avoids restore-path stalls that dominate TTFT and completed throughput.

## Queue and mechanism evidence for the proposed policy

The policy source metric is `vllm:num_requests_waiting`. MLflow archives it at 15-second cadence, whereas EPP consumes endpoint updates more frequently and applies a two-second EWMA. The archive is therefore a coarse guardrail check, not a reconstruction of gate decisions.

| Cell | Combined queue avg / max | Combined external KV share avg |
|---|---:|---:|
| 1K / C64 | 0.032 / 1 | 14.00% |
| 2K / C64 | 0.089 / 3 | 6.54% |
| 4K / C32 | 0.052 / 3 | 6.75% |
| 8K / C32 | 0.147 / 5 | 7.82% |
| 8K / C64 | 0.643 / 7 | 7.81% |

Figure 8 shows every per-run maximum available for the five combined-policy runs against the configured close threshold.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 8. Combined-policy archived waiting-queue maxima versus the close threshold","width":720,"height":270,"data":{"values":[{"cell":"1K / C64","max_waiting":1},{"cell":"2K / C64","max_waiting":3},{"cell":"4K / C32","max_waiting":3},{"cell":"8K / C32","max_waiting":5},{"cell":"8K / C64","max_waiting":7}]},"layer":[{"mark":{"type":"bar","color":"#1f77b4"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"y":{"field":"max_waiting","type":"quantitative","title":"Maximum waiting requests (requests)","scale":{"zero":true,"domain":[0,9]}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"max_waiting","type":"quantitative","title":"Maximum waiting requests"}]}},{"mark":{"type":"rule","color":"#d62728","strokeWidth":2,"strokeDash":[6,4]},"encoding":{"y":{"datum":8}}}]}
```

No archived bar reaches the red threshold rule; the maximum is 7. Because these samples are 15 seconds apart and there is no reason-labeled gate-action metric, they neither prove nor disprove a shorter EPP-visible threshold crossing.

At 8K/C64, the combined run averaged 0.287 seconds of NVMe busy time per second, approximately 660 MB/s of node NVMe throughput, and 170% CPU use in the collected container query. Its external-KV share averaged 7.81%. The archived `kv_offload_lookup_async_blocked_requests_by_engine` value averaged 0.487; despite its name, this query is the rate of the lookup delay-seconds sum and represents blocked lookup work in seconds per second, not a request count.

These mechanism summaries show that the combined arm exercised external loading under meaningful pressure, but none identifies why an individual request loaded or recomputed. A reason-labeled policy action counter or per-request decision trace is required before claiming queue-veto activation.

## Validity and limitations

- There was one run per policy and workload cell; no uncertainty interval is available.
- Policy order was fixed, so time drift remains possible despite same-node placement and cache cleanup.
- All four replicas were colocated on one H100 node, which improves comparability but limits topology generalization.
- The queue archive has 15-second resolution while the policy consumes faster endpoint updates and a two-second EWMA.
- The maximum archived combined waiting sample was 7, below the close threshold of 8.
- No reason-labeled gate action or per-request decision artifact identifies whether queue pressure changed a request.
- The result applies to the complete `1024 + 8` configuration. It does not estimate the incremental contribution of either component.
- The secondary tier was local NVMe; the result does not generalize to remote storage or P2P without validation.
- Error rates were low but nonzero, at most 0.116%; they should remain a rollout guardrail.
- Throughput and latency comparisons are valid within matching workload cells, not across prefix sizes with different token volume.

## Conclusion

The exact proposed `minExternalReusableTokens: 1024` plus `maxWaitingRequests: 8` policy has useful same-node validation against forced load/current behavior and forced recompute. It exceeds the better static request throughput in four of five cells, improves geometric-mean throughput by 8.8% over the per-cell better static action, and avoids forced-load TTFT tail collapse in the high-pressure cells. It also retains substantially better mean ITL than forced recompute.

The evidence supports presenting `1024 + 8` as a deployment-specific proposed policy configuration. It does not support saying that the queue gate fired or that a queue threshold of 8 independently improves performance: archived queue samples peaked at 7, and no reason-labeled action evidence exists.

The smallest next validation is to add reason-labeled load/recompute counters or per-request decision traces and rerun a workload that sustains the endpoint-local waiting EWMA above 8 on the same pinned node. That follow-up should test the policy's gate transition directly; it is not required to reinterpret this 15-run matrix as an ablation study.

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
| Combined 1,024 + 8 | 1K / C64 | [10b2b918](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/10b2b918f4aa46c58994a3bf5af42001?workspace=benchflow) |
| Combined 1,024 + 8 | 2K / C64 | [421a5b65](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/421a5b65bad145a9ba324e2ae872a823?workspace=benchflow) |
| Combined 1,024 + 8 | 4K / C32 | [86c7f15b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/86c7f15b915b49a3a2127adeac8f4633?workspace=benchflow) |
| Combined 1,024 + 8 | 8K / C32 | [ceccec62](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/ceccec62d1b6453e8c4e4f909cfde803?workspace=benchflow) |
| Combined 1,024 + 8 | 8K / C64 | [6ab2c9ef](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/6ab2c9ef0c4a474598b2bd74040deefb?workspace=benchflow) |

## Related

- [[2026-09-11 - Qwen3-32B queue-aware selective-loading validation]]
- [[2026-09-10 - Qwen3-32B focused selective-loading crossover calibration]]
- [[00 - Index]]

## Provenance

Metrics and artifacts were read from the 15 MLflow runs linked above. All charts use the complete selected three-arm dataset at the finest available per-run categorical grain. No run or chart datum from the removed fourth arm remains in this report.