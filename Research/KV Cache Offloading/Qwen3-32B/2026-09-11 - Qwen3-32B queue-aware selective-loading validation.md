---
title: "Qwen3-32B queue-aware selective-loading validation"
date: "2026-09-11"
type: "experiment-report"
topic: "KV Cache Offloading"
model: "Qwen/Qwen3-32B"
status: "ongoing"
experiment: "llm-d-selective-loading-queue-gate-validation"
matrix: "matrix-4951aa"
followup_experiment: "llm-d-selective-loading-queue-gate-threshold-4"
updated: "2026-09-12"
---

# Qwen3-32B queue-aware selective-loading validation

This is an ongoing selective-loading investigation. The problem is that restoring reusable KV can save prefill compute, but under endpoint pressure the restore path can compete with inference and make loading slower than recomputation. This experiment tests a first endpoint-local policy that permits loading when external KV exists and vetoes it when the selected vLLM endpoint's smoothed waiting queue is high.

## Executive summary

The queue-aware arm produced the highest request throughput in four of five tested cells and finished only 0.95% behind forced load in the fifth. Across the five cells, its geometric-mean throughput was 9.4% above an oracle that chooses the better of the two static policies independently for each cell, 19.4% above forced load, and 13.5% above forced recompute.

The strongest improvements occurred in the previously identified pressure region. At 8K/C32, queue-aware reached 8.927 requests/s, 15.9% above forced recompute and 32.0% above forced load. At 8K/C64, it reached 12.133 requests/s, 19.6% above forced recompute and 54.0% above forced load. Mean TTFT was lower than both static arms in four cells. At 8K/C64 it traded a higher mean TTFT than recompute, 664.5 ms versus 535.0 ms, for 19.6% more throughput; it still avoided forced load's 2,447 ms mean and 47.3 s p99 TTFT.

A five-run sensitivity follow-up lowered `maxWaitingRequests` from 8 to 4. It did not improve the result: geometric-mean throughput fell 0.67%, mean TTFT increased 3.46%, and p99 TTFT increased 5.67% relative to threshold 8. More importantly, the archived 15-second queue samples never reached 8 in the threshold-8 runs. The benefit over forced loading therefore cannot yet be attributed primarily to the queue veto; the precise-prefix evidence screen may be avoiding futile external lookups when no reusable external tokens exist.

## Validity verdict — Conditionally valid

The threshold-4 and threshold-8 runs are configuration-matched and suitable for sensitivity comparison, but the mechanism attribution is incomplete. Prometheus samples the queue every 15 seconds while EPP evaluates endpoint metrics much more frequently, and the plugin does not expose a reason-labeled decision counter. The outcome ranking is valid; the number of queue-triggered vetoes is unknown.

## Question and policy under test

The queue-aware policy was configured with `minExternalReusableTokens: 1` and `maxWaitingRequests: 8`. Therefore, the token screen permits loading only when the selected endpoint reports reusable KV outside GPU memory. Requests with zero external tier evidence are recomputed even when the queue gate is open. Either screen can therefore disable loading:

$$
\operatorname{load} = (T_{\mathrm{external}} \ge 1) \land (\text{queue gate is open})
$$

The endpoint-local EWMA has a fixed two-second half-life. The gate closes when EWMA waiting reaches 8 and reopens at 4, providing hysteresis. Missing queue identity or timestamp fails open. Forced load provides the restore baseline; forced recompute disables external KV loading while retaining local GPU prefix-cache reuse.

## Experimental design

All 15 combinations completed. Each used Qwen3-32B with four TP2 replicas colocated on the same 8×H100 node, `diadochos-hqxzk-gpu-h100-mt46x`. Every measured workload used a 300-second GuideLLM concurrent phase after a 2,048-request warmup, with 256 uncached prompt tokens, 128 output tokens, a static seed, and one turn. The reusable-prefix/concurrency cells were 1K/C64, 2K/C64, 4K/C32, 8K/C32, and 8K/C64.

The runtime image was `quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v1`; the queue-aware EPP image was `quay.io/rh-ee-aperdomo/llm-d-router-endpoint-picker:dev-c9117959`. The experiment used the detailed metrics profile. Error rates were at most 0.083%, so request failures do not explain the throughput ordering.

## Throughput result

Figure 1 shows the absolute successful-request throughput. Queue-aware is best in four cells and essentially tied with forced load at 4K/C32.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Request throughput by policy and workload","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","rps":25.897,"p99":563},{"cell":"1K / C64","policy":"Forced recompute","rps":23.807,"p99":528},{"cell":"1K / C64","policy":"Queue-aware","rps":26.253,"p99":212.2},{"cell":"2K / C64","policy":"Forced load","rps":18.12,"p99":12866.3},{"cell":"2K / C64","policy":"Forced recompute","rps":19.1,"p99":858.7},{"cell":"2K / C64","policy":"Queue-aware","rps":21.51,"p99":1929.1},{"cell":"4K / C32","policy":"Forced load","rps":11.62,"p99":1070.1},{"cell":"4K / C32","policy":"Forced recompute","rps":10.507,"p99":570.7},{"cell":"4K / C32","policy":"Queue-aware","rps":11.51,"p99":420.1},{"cell":"8K / C32","policy":"Forced load","rps":6.763,"p99":33063.7},{"cell":"8K / C32","policy":"Forced recompute","rps":7.7,"p99":1788.5},{"cell":"8K / C32","policy":"Queue-aware","rps":8.927,"p99":1886.2},{"cell":"8K / C64","policy":"Forced load","rps":7.877,"p99":47289},{"cell":"8K / C64","policy":"Forced recompute","rps":10.147,"p99":2976.7},{"cell":"8K / C64","policy":"Queue-aware","rps":12.133,"p99":6017}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"rps","type":"quantitative","title":"Successful request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Queue-aware"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"rps","type":"quantitative","title":"Throughput (requests/s)","format":".3f"}]}}
```

Figure 1. Successful request throughput from GuideLLM run metrics. Higher is better.

The more demanding comparison is against the best static policy in each cell. Figure 2 uses

$$
\Delta_{\mathrm{oracle}} = 100 \left(\frac{R_{\mathrm{queue}}}{\max(R_{\mathrm{load}}, R_{\mathrm{recompute}})} - 1\right)
$$

where $R$ is successful requests per second.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Queue-aware throughput versus the best static policy","width":720,"height":260,"data":{"values":[{"cell":"1K / C64","delta":1.4,"result":"Better","oracle":"load"},{"cell":"2K / C64","delta":12.6,"result":"Better","oracle":"recompute"},{"cell":"4K / C32","delta":-0.9,"result":"Worse","oracle":"load"},{"cell":"8K / C32","delta":15.9,"result":"Better","oracle":"recompute"},{"cell":"8K / C64","delta":19.6,"result":"Better","oracle":"recompute"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"y":{"field":"delta","type":"quantitative","title":"Throughput delta versus best static policy (%)","scale":{"zero":true}},"color":{"field":"result","type":"nominal","title":"Queue-aware result","scale":{"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"delta","type":"quantitative","title":"Throughput delta (%)","format":".1f"},{"field":"oracle","type":"nominal","title":"Best static policy"}]}}
```

Figure 2. Queue-aware throughput relative to the better static arm in the same workload cell. Positive values mean the dynamic policy exceeded both static runs.

| Cell | Forced load req/s | Forced recompute req/s | Queue-aware req/s | Better static arm | Queue vs best static |
|---|---:|---:|---:|---|---:|
| 1K / C64 | 25.897 | 23.807 | 26.253 | Load | +1.4% |
| 2K / C64 | 18.120 | 19.100 | 21.510 | Recompute | +12.6% |
| 4K / C32 | 11.620 | 10.507 | 11.510 | Load | −0.9% |
| 8K / C32 | 6.763 | 7.700 | 8.927 | Recompute | +15.9% |
| 8K / C64 | 7.877 | 10.147 | 12.133 | Recompute | +19.6% |

A dynamic policy can exceed the better whole-run static arm because it can load during lower-pressure intervals and recompute during higher-pressure intervals. That is the intended mechanism, but the present experiment does not directly count those transitions, so this remains an inference from outcomes.

## Latency result

Queue-aware reduced mean TTFT below both static arms at 1K/C64, 2K/C64, 4K/C32, and 8K/C32. At 8K/C64, forced recompute retained the lowest mean TTFT. Figure 3 compares the mean directly; Figure 4 then shows the request-level p99 tail.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3. Mean time to first token by policy and workload","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","mean_ttft":157.3},{"cell":"1K / C64","policy":"Forced recompute","mean_ttft":128.1},{"cell":"1K / C64","policy":"Queue-aware","mean_ttft":114.8},{"cell":"2K / C64","policy":"Forced load","mean_ttft":829.1},{"cell":"2K / C64","policy":"Forced recompute","mean_ttft":218.6},{"cell":"2K / C64","policy":"Queue-aware","mean_ttft":206.4},{"cell":"4K / C32","policy":"Forced load","mean_ttft":231.7},{"cell":"4K / C32","policy":"Forced recompute","mean_ttft":229.7},{"cell":"4K / C32","policy":"Queue-aware","mean_ttft":199},{"cell":"8K / C32","policy":"Forced load","mean_ttft":1321.4},{"cell":"8K / C32","policy":"Forced recompute","mean_ttft":438.5},{"cell":"8K / C32","policy":"Queue-aware","mean_ttft":375},{"cell":"8K / C64","policy":"Forced load","mean_ttft":2447.3},{"cell":"8K / C64","policy":"Forced recompute","mean_ttft":535},{"cell":"8K / C64","policy":"Queue-aware","mean_ttft":664.5}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"mean_ttft","type":"quantitative","title":"Mean TTFT (ms)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Queue-aware"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"mean_ttft","type":"quantitative","title":"Mean TTFT (ms)","format":".1f"}]}}
```

Figure 3. GuideLLM request-level mean TTFT for all three policies. Lower is better. Queue-aware has the lowest mean in four cells; recompute wins at 8K/C64.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4. Request-level p99 time to first token","width":720,"height":330,"data":{"values":[{"cell":"1K / C64","policy":"Forced load","rps":25.897,"p99":563},{"cell":"1K / C64","policy":"Forced recompute","rps":23.807,"p99":528},{"cell":"1K / C64","policy":"Queue-aware","rps":26.253,"p99":212.2},{"cell":"2K / C64","policy":"Forced load","rps":18.12,"p99":12866.3},{"cell":"2K / C64","policy":"Forced recompute","rps":19.1,"p99":858.7},{"cell":"2K / C64","policy":"Queue-aware","rps":21.51,"p99":1929.1},{"cell":"4K / C32","policy":"Forced load","rps":11.62,"p99":1070.1},{"cell":"4K / C32","policy":"Forced recompute","rps":10.507,"p99":570.7},{"cell":"4K / C32","policy":"Queue-aware","rps":11.51,"p99":420.1},{"cell":"8K / C32","policy":"Forced load","rps":6.763,"p99":33063.7},{"cell":"8K / C32","policy":"Forced recompute","rps":7.7,"p99":1788.5},{"cell":"8K / C32","policy":"Queue-aware","rps":8.927,"p99":1886.2},{"cell":"8K / C64","policy":"Forced load","rps":7.877,"p99":47289},{"cell":"8K / C64","policy":"Forced recompute","rps":10.147,"p99":2976.7},{"cell":"8K / C64","policy":"Queue-aware","rps":12.133,"p99":6017}]},"mark":{"type":"point","filled":true,"size":110},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"policy"},"y":{"field":"p99","type":"quantitative","title":"TTFT p99 (ms, logarithmic scale)","scale":{"type":"log","domain":[100,60000]}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced load","Forced recompute","Queue-aware"],"scheme":"category10"}},"shape":{"field":"policy","type":"nominal","title":"Policy"},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"policy","type":"nominal","title":"Policy"},{"field":"p99","type":"quantitative","title":"TTFT p99 (ms)","format":".1f"}]}}
```

Figure 4. GuideLLM request-level p99 TTFT on a logarithmic scale. Lower is better.

| Cell | Load mean / p99 TTFT | Recompute mean / p99 TTFT | Queue-aware mean / p99 TTFT |
|---|---:|---:|---:|
| 1K / C64 | 157 / 563 ms | 128 / 528 ms | **115 / 212 ms** |
| 2K / C64 | 829 / 12,866 ms | 219 / **859 ms** | **206** / 1,929 ms |
| 4K / C32 | 232 / 1,070 ms | 230 / 571 ms | **199 / 420 ms** |
| 8K / C32 | 1,321 / 33,064 ms | 439 / **1,789 ms** | **375** / 1,886 ms |
| 8K / C64 | 2,447 / 47,289 ms | **535 / 2,977 ms** | 664 / 6,017 ms |

The 8K/C64 result is a throughput/tail tradeoff, not an unconditional latency win. Queue-aware improves throughput and mean end-to-end latency over recompute, 5.141 s versus 6.109 s, but recompute has the better TTFT tail.

## Low-pressure mechanism sanity check

The raw 15-second `kserve_vllm:num_requests_waiting` series was inspected for queue-aware 1K/C64 over the final 300 seconds. Across 160 engine-series samples, mean waiting was 0.1, p95 was 1, and the maximum was 1. The close threshold of 8 was never approached, so this cell is a negative-control case where the queue veto should remain open.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 5. Low-pressure waiting queue versus gate threshold","width":720,"height":230,"data":{"values":[{"stat":"Mean","waiting":0.1},{"stat":"p95","waiting":1},{"stat":"Maximum","waiting":1}]},"layer":[{"mark":{"type":"bar","color":"#1f77b4"},"encoding":{"x":{"field":"stat","type":"ordinal","sort":["Mean","p95","Maximum"],"title":"Statistic over 160 engine-series samples"},"y":{"field":"waiting","type":"quantitative","title":"Waiting requests (requests)","scale":{"zero":true,"domain":[0,9]}},"tooltip":[{"field":"stat","type":"ordinal","title":"Statistic"},{"field":"waiting","type":"quantitative","title":"Waiting requests","format":".1f"}]}},{"mark":{"type":"rule","color":"#d62728","strokeWidth":2,"strokeDash":[6,4]},"encoding":{"y":{"datum":8}}}]}
```

Figure 5. Waiting-queue evidence for queue-aware 1K/C64. The measured gauge remained far below the gate threshold.

The same run retained substantial external loading: the archived summary reports external-KV prompt-token share averaging 15.5% over the collected interval and 21.7% at the latest sample. Its CPU→GPU transfer series was also non-zero. This is consistent with the queue gate staying open. The approximately +1.4% throughput difference from forced load should therefore be treated as run variation, not as a queue-veto benefit.

The waiting gauge is the policy input, while the Prometheus artifact is a 15-second observation of it. The EPP computes its own endpoint-local EWMA from endpoint metric updates with a two-second half-life, so the archived series cannot reconstruct every exact close/reopen transition.

## Threshold-4 sensitivity follow-up

A follow-up changed only the queue close threshold from 8 to 4; hysteresis therefore reopened at 2 instead of 4. It reused the same five workloads, four TP2 replicas, node, runtime image, EPP image, warmup, and collection profile.

| Cell | Threshold 8 req/s | Threshold 4 req/s | Throughput delta | T8 mean TTFT | T4 mean TTFT | TTFT delta | T8 p99 TTFT | T4 p99 TTFT |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 1K / C64 | 26.253 | 26.283 | +0.11% | 114.8 ms | 112.5 ms | −1.95% | 212 ms | 212 ms |
| 2K / C64 | 21.510 | 21.323 | −0.87% | 206.4 ms | 216.3 ms | +4.78% | 1,929 ms | 2,171 ms |
| 4K / C32 | 11.510 | 11.573 | +0.55% | 199.0 ms | 200.1 ms | +0.59% | 420 ms | 442 ms |
| 8K / C32 | 8.927 | 8.713 | −2.39% | 375.0 ms | 414.5 ms | +10.53% | 1,886 ms | 1,917 ms |
| 8K / C64 | 12.133 | 12.043 | −0.74% | 664.5 ms | 689.4 ms | +3.75% | 6,017 ms | 6,597 ms |
| Geometric mean | — | — | **−0.67%** | — | — | **+3.46%** | — | **+5.67%** |

Figure 6 compares successful-request throughput from the GuideLLM run metrics at both thresholds.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 6. Throughput sensitivity to waiting-queue threshold","width":720,"height":320,"data":{"values":[{"cell":"1K / C64","threshold":"Threshold 8","value":26.253},{"cell":"1K / C64","threshold":"Threshold 4","value":26.283},{"cell":"2K / C64","threshold":"Threshold 8","value":21.51},{"cell":"2K / C64","threshold":"Threshold 4","value":21.323},{"cell":"4K / C32","threshold":"Threshold 8","value":11.51},{"cell":"4K / C32","threshold":"Threshold 4","value":11.573},{"cell":"8K / C32","threshold":"Threshold 8","value":8.927},{"cell":"8K / C32","threshold":"Threshold 4","value":8.713},{"cell":"8K / C64","threshold":"Threshold 8","value":12.133},{"cell":"8K / C64","threshold":"Threshold 4","value":12.043}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"threshold"},"y":{"field":"value","type":"quantitative","title":"Successful request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"threshold","type":"nominal","title":"Queue close threshold","scale":{"domain":["Threshold 8","Threshold 4"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"threshold","type":"nominal","title":"Policy"},{"field":"value","type":"quantitative","title":"Throughput (requests/s)","format":".3f"}]}}
```

Figure 6. Threshold 4 is effectively tied at low pressure and slightly worse in three cells. The largest throughput regression is 2.39% at 8K/C32.

Figure 7 compares GuideLLM request-level mean TTFT using the same five pairs.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 7. Mean TTFT sensitivity to waiting-queue threshold","width":720,"height":320,"data":{"values":[{"cell":"1K / C64","threshold":"Threshold 8","value":114.8},{"cell":"1K / C64","threshold":"Threshold 4","value":112.5},{"cell":"2K / C64","threshold":"Threshold 8","value":206.4},{"cell":"2K / C64","threshold":"Threshold 4","value":216.3},{"cell":"4K / C32","threshold":"Threshold 8","value":199},{"cell":"4K / C32","threshold":"Threshold 4","value":200.1},{"cell":"8K / C32","threshold":"Threshold 8","value":375},{"cell":"8K / C32","threshold":"Threshold 4","value":414.5},{"cell":"8K / C64","threshold":"Threshold 8","value":664.5},{"cell":"8K / C64","threshold":"Threshold 4","value":689.4}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K / C64","2K / C64","4K / C32","8K / C32","8K / C64"],"title":"Reusable prefix / concurrency","axis":{"labelAngle":0}},"xOffset":{"field":"threshold"},"y":{"field":"value","type":"quantitative","title":"Mean TTFT (ms)","scale":{"zero":true}},"color":{"field":"threshold","type":"nominal","title":"Queue close threshold","scale":{"domain":["Threshold 8","Threshold 4"],"scheme":"category10"}},"tooltip":[{"field":"cell","type":"ordinal","title":"Workload"},{"field":"threshold","type":"nominal","title":"Policy"},{"field":"value","type":"quantitative","title":"Mean TTFT (ms)","format":".1f"}]}}
```

Figure 7. Threshold 4 improves mean TTFT only at 1K/C64. Its largest regression is 10.53% at 8K/C32.

Raw `kserve_vllm:num_requests_waiting` was preserved at 15-second cadence. Across the final 300 seconds, no threshold-8 sample reached 8. Threshold-4 samples reached at least 4 in only 1.88% of samples at 2K/C64 and 3.75% at 8K/C64; the other three cells never reached 4. The EPP's two-second EWMA can still react to sub-scrape spikes, so these samples do not prove that the gate never closed. They do show that sustained queue pressure near either threshold was uncommon.

Threshold 4 still beats the better static policy in four of five cells and has an 8.67% geometric-mean advantage over that per-cell static oracle. However, because lowering the queue threshold barely changes outcomes and generally worsens latency, the earlier benefit should be attributed to the combined selective policy, not specifically to the waiting-queue veto.

## Interpretation

The matrix supports three conclusions.

1. The old static rule is insufficient. Forced load wins 1K/C64 and 4K/C32, while forced recompute wins 2K/C64 and both 8K cells. Prefix length alone does not order the decision under changing concurrency and endpoint pressure.
2. The combined selective policy is useful, but this experiment does not isolate the queue veto from the precise-prefix evidence screen. Thresholds 4 and 8 produce nearly identical outcomes, and sustained observed queues were below both close points.
3. Threshold 8 is preferable to threshold 4 for this deployment, but neither value is production-calibrated. Model, TP, replica layout, HBM capacity, CPU tier, storage path, and traffic shape can move the close point.

The result does not show that NVMe bandwidth alone should drive the gate. The policy uses endpoint waiting, and the available low-pressure raw slice was dominated by NVMe writes rather than reads. In addition, the archived node-level NVMe query includes both `mt46x` and `gjfjh` because it joins all release pods; node-level storage plots must be filtered to the model-serving node before causal interpretation.

## Validity and limitations

- The comparison is same-node and configuration-matched, removing the earlier cross-node `gjfjh` confounder.
- There is one run per policy/cell. GuideLLM contributes thousands of requests, but there is no run-level confidence interval for environmental drift.
- Matrix placement was sequential, and the static-load, static-recompute, and queue-aware groups ran in that order. A monotonic time drift could bias the policy-wide result.
- Existing DEBUG logging records `loadAction`, but not whether a disable was caused by the token screen or the queue veto. The experiment artifact set does not provide a direct queue-veto transition count.
- The token threshold of 1 intentionally removes token-size calibration from this test. This report validates the combined mechanism, not a final two-dimensional token/queue policy.
- The p99 values are GuideLLM request-level statistics. Prometheus histogram time series serve as system-state context and should not replace them for request outcome comparison.

## Recommendation

Retain `maxWaitingRequests: 8`, reopen at 4, and the fixed two-second half-life as the experimental default. Threshold 4 provides no measurable benefit and should not replace it.

Do not run another threshold sweep yet. The next smallest decisive test is explicit decision attribution on the same five cells: A single counter labeled `action=load|recompute` and `reason=token_threshold|waiting_queue|preserve`, or equivalent structured logs captured as artifacts, would let us verify that pressure cells contain both actions, align close/reopen transitions with endpoint waiting, and calculate outcome metrics for gated versus ungated intervals. No additional storage heuristic should be added until that attribution is available.

## MLflow run registry

| Policy | Cell | Run |
|---|---|---|
| Forced load | 1K / C64 | [42c4dc94](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/42c4dc9477f24421b5ff8b2acbca28d8?workspace=benchflow) |
| Forced load | 2K / C64 | [9619f8bf](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/9619f8bf7f584aa3be1ec172a887edf7?workspace=benchflow) |
| Forced load | 4K / C32 | [402fa78b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/402fa78bd5cf4afa89a25f97a9bb12c2?workspace=benchflow) |
| Forced load | 8K / C32 | [0aac9c6a](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/0aac9c6a69e945d7ba4ad4be52509774?workspace=benchflow) |
| Forced load | 8K / C64 | [b10b98f9](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/b10b98f9289b4fb6a45a7c29b079060a?workspace=benchflow) |
| Forced recompute | 1K / C64 | [67e77482](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/67e7748250374a69a0d39a3fd0938644?workspace=benchflow) |
| Forced recompute | 2K / C64 | [3da2217c](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/3da2217c99fe44e08dcce7292e7385d6?workspace=benchflow) |
| Forced recompute | 4K / C32 | [b548198b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/b548198bcbec4e3592a1ea656766d866?workspace=benchflow) |
| Forced recompute | 8K / C32 | [a6d2863e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/a6d2863e6a63442ab763eacab95d4b83?workspace=benchflow) |
| Forced recompute | 8K / C64 | [1efc3534](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/1efc353424dc4250b857a088035e2c3d?workspace=benchflow) |
| Queue-aware | 1K / C64 | [f794b327](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/f794b327bb6b4ca3994ced9a4d0b2353?workspace=benchflow) |
| Queue-aware | 2K / C64 | [a7e7f439](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/a7e7f439205e4c79821e440692f396bd?workspace=benchflow) |
| Queue-aware | 4K / C32 | [bab9cf05](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/bab9cf05cf1447e6818f4ba947431663?workspace=benchflow) |
| Queue-aware | 8K / C32 | [bfdf9660](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/bfdf96609cb04ea59c5c07dc05fb50c5?workspace=benchflow) |
| Queue-aware | 8K / C64 | [72c46bf2](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/72c46bf2188c4cbdbed08ab2246c7e49?workspace=benchflow) |
| Queue-aware, threshold 4 | 1K / C64 | [2e52fada](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/2e52fada5e0946ba947f6ea29d76dc7f?workspace=benchflow) |
| Queue-aware, threshold 4 | 2K / C64 | [7f77bdc0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/7f77bdc022e3493db8c4b458e4fa3f12?workspace=benchflow) |
| Queue-aware, threshold 4 | 4K / C32 | [2a717d62](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/2a717d62795f4263a6d08f70c116811c?workspace=benchflow) |
| Queue-aware, threshold 4 | 8K / C32 | [834441d5](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/834441d563d94170acc6c2f4dfe597dd?workspace=benchflow) |
| Queue-aware, threshold 4 | 8K / C64 | [bbdbd4d1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/bbdbd4d10a154c58850633ee893f2e50?workspace=benchflow) |