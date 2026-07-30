---
title: "2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison"
date: "2026-07-30"
type: "experiment-report"
topic: "KV Cache Offloading"
experiment: "MLflow experiment 328"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "not recorded"
runtime_image: "vllm/vllm-openai:v0.23.0"
runtime_image_digest: "sha256:3a1e7f5904e1a1192a02aa0086ceaffc33985d7044c7bb25b3a43d61bdbe3ac0"
vllm_version: "0.23.0"
gpu_type: "H100"
gpu_count: 8
tensor_parallelism: 8
pipeline_parallelism: 1
replicas: 1
gpu_memory_utilization: 0.9
max_model_len: 131072
max_num_seqs: "not explicitly set"
concurrency: 16
cpu_bytes: 274877906944
offload_spec: "TieringOffloadingSpec for tiered configurations"
secondary_tier: "none, NVMe, or CephFS"
shared_memory: "300Gi"
workload: "AIPerf AgentX inferencex-agentx-mvp"
random_seed: 42
duration_seconds: 1800
configuration_count: 4
baseline: "No offload"
status: "directionally valid single-run comparison"
---

# 2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison

## Benchmark overview

This experiment compares `nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8` on one eight-H100, TP8 replica under the AgentX MVP workload at concurrency 16. The stable parameters are vLLM 0.23.0, `gpu-memory-utilization=0.9`, 131,072-token context, FP8 KV cache, prefix caching, seed 42, a 1,800-second send window, and 300 GiB shared memory. The four configurations are no offload, a 256 GiB CPU tier, CPU plus local NVMe, and CPU plus CephFS. The artifacts identify the model as **FP8**; this report uses that exact identifier.

## Executive summary

There is **performance parity, not an offload win**. Relative to no offload, request throughput changes by -0.36% for CPU-only, +0.50% for NVMe, and -0.38% for CephFS. Output-token throughput stays within 0.31% of baseline. These are single-run deltas smaller than ordinary run-to-run uncertainty.

The reason is structural: **offloading is enabled, but useful HBM eviction is not being forced**. The server allocates **2,314,336 GPU KV tokens** (282.5 GiB aggregate), equivalent to 17.66 full 131,072-token contexts. The workload peaks at only **50%** of this shelf—about 1.16 million tokens—and records zero preemptions. All tiered cells write roughly 1.0 TiB out of HBM, but CPU-only restores **0 GiB**, while NVMe and CephFS each restore only **18.7 GiB** and supply just **0.15%** of prompt tokens. More than 91.9% of prompt tokens are already local HBM hits.

The current 256 GiB CPU tier is also slightly smaller than the 282.5 GiB HBM shelf. At the observed 488 MiB/s CPU-store rate, its approximate retention clock is about **537 seconds**, while useful HBM blocks remain resident for roughly 720–1,190 seconds at the reported eviction quantiles. CPU copies expire before they become necessary. NVMe and CephFS retain enough history to produce a small read-back signal, but it is too small to affect outcomes.

**Recommended center point:** keep TP8 and reduce `gpu-memory-utilization` from 0.90 to **0.68**, with a calibration sweep at **0.70, 0.68, and 0.66**. The same-run memory profile projects U0.68 to about 1.16 million KV tokens—almost exactly the observed peak—so it should force spill without starting with the severe queueing expected below U0.64. Verify the actual startup-reported block count and target 90–98% peak occupancy, non-zero external prompt share, non-zero CPU→GPU loads, and near-zero preemptions.

## Headline results

| Configuration | MLflow | Req/s | Output tok/s | Mean TTFT (s) | Mean E2E (s) | Mean ITL (ms) | Sessions | External prompt share | KV mean / peak |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| No offload | [57e480071aa44f40a5fc0f629f280731](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/57e480071aa44f40a5fc0f629f280731?workspace=benchflow) | 0.4327 | 352.7 | 1.101 | 22.21 | 27.38 | 19 / 34 | 0.000% | 25.3% / 50.0% |
| CPU offload 256 GiB | [c40d143bd48d4784b74066788cba65df](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/c40d143bd48d4784b74066788cba65df?workspace=benchflow) | 0.4312 | 353.2 | 1.111 | 22.28 | 27.59 | 20 / 34 | 0.000% | 25.4% / 47.1% |
| CPU + NVMe | [084819f27b5a47d2b3441e5c7743c6fb](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/084819f27b5a47d2b3441e5c7743c6fb?workspace=benchflow) | 0.4349 | 353.8 | 1.168 | 22.15 | 27.35 | 19 / 34 | 0.149% | 25.8% / 49.5% |
| CPU + CephFS | [d293ce35e62546c9ac63a2ad1bd6ee83](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d293ce35e62546c9ac63a2ad1bd6ee83?workspace=benchflow) | 0.4311 | 353.2 | 1.152 | 22.18 | 26.87 | 20 / 34 | 0.150% | 26.7% / 49.6% |

All configurations complete essentially the same request sequence: 786–788 requests, 19–20 of 34 sessions, and 52 branches. The fixed-duration cutoff cancels only 2–3 requests per cell.

## Performance comparison

```vega-lite
{"width":740,"height":300,"title":"Figure 1 \u2014 Completed-request throughput","data":{"values":[{"configuration":"No offload","requests_s":0.43270876395978086},{"configuration":"CPU offload 256 GiB","requests_s":0.43116858133457087},{"configuration":"CPU + NVMe","requests_s":0.4348582021805327},{"configuration":"CPU + CephFS","requests_s":0.43108445807122675}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"requests_s","type":"quantitative","title":"Requests per second","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"requests_s","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Figure 1 shows no material winner. NVMe leads by 0.50% in this single run; that is evidence of parity, not a demonstrated benefit.

```vega-lite
{"width":740,"height":300,"title":"Figure 2 \u2014 Output-token throughput","data":{"values":[{"configuration":"No offload","tokens_s":352.6928759108008},{"configuration":"CPU offload 256 GiB","tokens_s":353.177954760303},{"configuration":"CPU + NVMe","tokens_s":353.7897066539759},{"configuration":"CPU + CephFS","tokens_s":353.2463603146196}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"tokens_s","type":"quantitative","title":"Output tokens per second","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"tokens_s","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Figure 2 is equally flat. The maximum output-throughput difference from baseline is below 0.4%.

```vega-lite
{"width":760,"height":320,"title":"Figure 3 \u2014 Request latency","data":{"values":[{"configuration":"No offload","metric":"TTFT mean","seconds":1.1005646817595418},{"configuration":"No offload","metric":"TTFT P95","seconds":4.466678266},{"configuration":"No offload","metric":"E2E mean","seconds":22.210133569782442},{"configuration":"No offload","metric":"E2E P95","seconds":76.8070140375},{"configuration":"CPU offload 256 GiB","metric":"TTFT mean","seconds":1.1110020004517767},{"configuration":"CPU offload 256 GiB","metric":"TTFT P95","seconds":4.3514258431999995},{"configuration":"CPU offload 256 GiB","metric":"E2E mean","seconds":22.277449352355333},{"configuration":"CPU offload 256 GiB","metric":"E2E P95","seconds":80.07660642629996},{"configuration":"CPU + NVMe","metric":"TTFT mean","seconds":1.16833438211802},{"configuration":"CPU + NVMe","metric":"TTFT P95","seconds":4.595333761299999},{"configuration":"CPU + NVMe","metric":"E2E mean","seconds":22.1464053935368},{"configuration":"CPU + NVMe","metric":"E2E P95","seconds":78.49078641154998},{"configuration":"CPU + CephFS","metric":"TTFT mean","seconds":1.1524553306662435},{"configuration":"CPU + CephFS","metric":"TTFT P95","seconds":4.38970762685},{"configuration":"CPU + CephFS","metric":"E2E mean","seconds":22.178160923611674},{"configuration":"CPU + CephFS","metric":"E2E P95","seconds":78.8916894245}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["TTFT mean","TTFT P95","E2E mean","E2E P95"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"seconds","type":"quantitative","title":"Seconds","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"seconds","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Mean TTFT is 0.95% slower for CPU-only, 6.16% slower for NVMe, and 4.71% slower for CephFS. Mean E2E latency remains within 0.4% of baseline because decode time dominates the complete request.

```vega-lite
{"width":760,"height":320,"title":"Figure 4 \u2014 Request, session, and branch outcomes","data":{"values":[{"configuration":"No offload","metric":"Requests completed","count":786},{"configuration":"No offload","metric":"Requests cancelled","count":3},{"configuration":"No offload","metric":"Sessions completed","count":19},{"configuration":"No offload","metric":"Sessions incomplete","count":15},{"configuration":"No offload","metric":"Branches completed","count":52},{"configuration":"CPU offload 256 GiB","metric":"Requests completed","count":788},{"configuration":"CPU offload 256 GiB","metric":"Requests cancelled","count":2},{"configuration":"CPU offload 256 GiB","metric":"Sessions completed","count":20},{"configuration":"CPU offload 256 GiB","metric":"Sessions incomplete","count":14},{"configuration":"CPU offload 256 GiB","metric":"Branches completed","count":52},{"configuration":"CPU + NVMe","metric":"Requests completed","count":788},{"configuration":"CPU + NVMe","metric":"Requests cancelled","count":3},{"configuration":"CPU + NVMe","metric":"Sessions completed","count":19},{"configuration":"CPU + NVMe","metric":"Sessions incomplete","count":15},{"configuration":"CPU + NVMe","metric":"Branches completed","count":52},{"configuration":"CPU + CephFS","metric":"Requests completed","count":788},{"configuration":"CPU + CephFS","metric":"Requests cancelled","count":2},{"configuration":"CPU + CephFS","metric":"Sessions completed","count":20},{"configuration":"CPU + CephFS","metric":"Sessions incomplete","count":14},{"configuration":"CPU + CephFS","metric":"Branches completed","count":52}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Requests completed","Requests cancelled","Sessions completed","Sessions incomplete","Branches completed"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"count","type":"quantitative","title":"Count","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"count","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Session and branch outcomes are effectively identical. The one-session difference is not credible as a performance effect with one repetition and boundary censoring.

## Why offload does not improve performance

```vega-lite
{"width":760,"height":320,"title":"Figure 5 \u2014 Prompt tokens by source","data":{"values":[{"configuration":"No offload","source":"External KV transfer","share":0.0},{"configuration":"No offload","source":"Local HBM hit","share":91.97435884314889},{"configuration":"No offload","source":"Local compute","share":8.025641156851119},{"configuration":"CPU offload 256 GiB","source":"External KV transfer","share":0.0},{"configuration":"CPU offload 256 GiB","source":"Local HBM hit","share":91.96870351251056},{"configuration":"CPU offload 256 GiB","source":"Local compute","share":8.031296487489435},{"configuration":"CPU + NVMe","source":"External KV transfer","share":0.14906966030176014},{"configuration":"CPU + NVMe","source":"Local HBM hit","share":91.98589442686112},{"configuration":"CPU + NVMe","source":"Local compute","share":7.865035912837115},{"configuration":"CPU + CephFS","source":"External KV transfer","share":0.15013548835661328},{"configuration":"CPU + CephFS","source":"Local HBM hit","share":91.97456518841716},{"configuration":"CPU + CephFS","source":"Local compute","share":7.875299323226222}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"share","type":"quantitative","title":"Integrated prompt-token rate share (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"Prompt source","scale":{"domain":["External KV transfer","Local HBM hit","Local compute"],"range":["#9467bd","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"source","type":"nominal"},{"field":"share","type":"quantitative","format":".4f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

The baseline already serves 91.97% of prompt-token work from local HBM and computes only 8.03%. CPU-only has exactly 0% external restoration. NVMe and CephFS expose a real but tiny 0.149–0.150% external source. There is almost no recomputation left for offload to eliminate.

```vega-lite
{"width":760,"height":320,"title":"Figure 6 \u2014 HBM KV-cache utilization","data":{"values":[{"configuration":"No offload","metric":"Mean","percent":25.313137626231867},{"configuration":"No offload","metric":"Peak","percent":49.95540806802862},{"configuration":"CPU offload 256 GiB","metric":"Mean","percent":25.397399168454413},{"configuration":"CPU offload 256 GiB","metric":"Peak","percent":47.06695703273531},{"configuration":"CPU + NVMe","metric":"Mean","percent":25.76218623921621},{"configuration":"CPU + NVMe","metric":"Peak","percent":49.456946316844686},{"configuration":"CPU + CephFS","metric":"Mean","percent":26.70048156840945},{"configuration":"CPU + CephFS","metric":"Peak","percent":49.561339832002496}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Mean","Peak"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"percent","type":"quantitative","title":"KV-cache utilization (%)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"percent","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

The HBM cache averages 25–27% utilization and peaks below 50% in every cell. A connector can eagerly store completed blocks while those blocks remain locally addressable; write volume alone therefore does not prove that an external tier participates in the critical request path.

```vega-lite
{"width":760,"height":320,"title":"Figure 7 \u2014 Cumulative KV transfer volume","data":{"values":[{"configuration":"CPU offload 256 GiB","direction":"GPU \u2192 CPU","gib":1002.140625},{"configuration":"CPU offload 256 GiB","direction":"CPU \u2192 GPU","gib":0},{"configuration":"CPU + NVMe","direction":"GPU \u2192 CPU","gib":983.69921875},{"configuration":"CPU + NVMe","direction":"CPU \u2192 GPU","gib":18.71484375},{"configuration":"CPU + CephFS","direction":"GPU \u2192 CPU","gib":983.48046875},{"configuration":"CPU + CephFS","direction":"CPU \u2192 GPU","gib":18.71484375}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"direction","type":"nominal","title":"Metric","sort":["GPU \u2192 CPU","CPU \u2192 GPU"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"gib","type":"quantitative","title":"GiB","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"direction","type":"nominal"},{"field":"gib","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU-only writes 1,002.1 GiB and performs no measured CPU→GPU restore. NVMe writes 983.7 GiB and restores 18.7 GiB; CephFS writes 983.5 GiB and restores 18.7 GiB. The secondary tiers prove that the restore mechanism works, but their store/load ratio remains about 52.6:1.

The retention ordering explains the CPU-only result:

$$
t_{CPU} \approx \frac{256\;GiB}{487.9\;MiB/s} = 537\;s
$$

The final reported block reuse gaps are approximately 23–25 seconds at P50, 118–167 seconds at P90, and 281–287 seconds at P99. However, the local HBM eviction telemetry is much longer: about 718–749 seconds at P50 and about 1,190 seconds at P99. A CPU copy can be overwritten while its HBM original is still present. Only the longer-lived filesystem tiers survive long enough to produce the tiny observed restore signal.

## GPU-memory-utilization calibration

The startup profile reports 30.06 GiB of model loading per GPU and 35.31 GiB of available KV cache at U0.90. For this exact image/model/TP combination, the implied non-KV reservation is:

$$
B_{nonKV} \approx 0.90 \times 80 - 35.31 = 36.69\;GiB/GPU
$$

The local projection is therefore:

$$
B_{KV,total}(U) \approx 8 \times (80U - 36.69)
$$

```vega-lite
{"width":760,"height":330,"title":"Figure 8 \u2014 Projected HBM token shelf versus observed demand","data":{"values":[{"gpu_memory_utilization":0.9,"capacity_mtokens":2.31407616,"observed_peak_mtokens":1.1561359928652908},{"gpu_memory_utilization":0.8,"capacity_mtokens":1.78978816,"observed_peak_mtokens":1.1561359928652908},{"gpu_memory_utilization":0.7,"capacity_mtokens":1.2655001600000002,"observed_peak_mtokens":1.1561359928652908},{"gpu_memory_utilization":0.68,"capacity_mtokens":1.1606425600000005,"observed_peak_mtokens":1.1561359928652908},{"gpu_memory_utilization":0.67,"capacity_mtokens":1.1082137600000002,"observed_peak_mtokens":1.1561359928652908},{"gpu_memory_utilization":0.66,"capacity_mtokens":1.0557849600000004,"observed_peak_mtokens":1.1561359928652908},{"gpu_memory_utilization":0.65,"capacity_mtokens":1.00335616,"observed_peak_mtokens":1.1561359928652908},{"gpu_memory_utilization":0.64,"capacity_mtokens":0.9509273600000003,"observed_peak_mtokens":1.1561359928652908}]},"layer":[{"mark":{"type":"line","point":true,"strokeWidth":2},"encoding":{"x":{"field":"gpu_memory_utilization","type":"quantitative","title":"gpu-memory-utilization","scale":{"domain":[0.63,0.91]}},"y":{"field":"capacity_mtokens","type":"quantitative","title":"KV capacity (million tokens)","scale":{"zero":true}},"color":{"value":"#1f77b4"},"tooltip":[{"field":"gpu_memory_utilization","type":"quantitative","format":".2f"},{"field":"capacity_mtokens","type":"quantitative","format":".3f"}]}},{"mark":{"type":"rule","color":"#d62728","strokeDash":[6,4]},"encoding":{"y":{"aggregate":"mean","field":"observed_peak_mtokens","type":"quantitative"}}}],"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

The red rule is the measured peak demand at U0.90. The projection gives:

| U | Aggregate KV shelf | Projected KV tokens | Full-context equivalents | Expected pressure |
|---:|---:|---:|---:|---|
| 0.90 | 282.5 GiB | 2.314 M | 17.66 | Proven underfilled |
| 0.80 | 218.5 GiB | 1.790 M | 13.66 | Probably underfilled |
| 0.70 | 154.5 GiB | 1.266 M | 9.66 | Calibration guardrail |
| 0.68 | 141.7 GiB | 1.161 M | 8.86 | Recommended center |
| 0.66 | 128.9 GiB | 1.056 M | 8.06 | Forced-spill point |
| 0.64 | 116.1 GiB | 0.951 M | 7.26 | Higher queueing risk |

These are projections, not substitutes for the startup log. CUDA-graph profiling is enabled in vLLM 0.23.0, and each run must reconcile the actual `GPU KV cache size` line.

### Recommended calibration sequence

1. Run **no offload and CPU256 at U0.70**. This should raise peak occupancy from ~50% to roughly 91% if demand is stable.
2. Run the same pair at **U0.68**. This is the first setting expected to touch the measured peak and force capacity eviction.
3. If external prompt share remains below 5%, test **U0.66**. Stop lowering if preemptions become sustained, mean waiting rises materially, or TTFT worsens faster than local-compute share falls.
4. Select the highest U that produces 90–98% peak occupancy, non-zero CPU→GPU loads throughout the run, external prompt share above 5%, and lower local-compute share without an error or tail-latency cliff.
5. Lock that U across all four cells, then run at least three paired seeds before claiming performance.

Do not jump directly to U0.55. The current peak demand would exceed that projected shelf by a wide margin, turning the experiment into a scheduler-pressure test rather than a clean offload comparison.

## Tensor parallelism

**Keep TP8 for the next experiment.** The runtime allocates each GPU a cache shape of `(144646, 1, 64, 2, 16, 128)`, which is consistent with one KV-head shard per rank for this architecture. TP8 already provides a clean KV shard and fits the 253B FP8 weights with 30.06 GiB per GPU.

- **TP4 is not the first tuning lever.** Weight memory would approximately double per GPU before graph and activation overhead, leaving a very small KV shelf at U0.90. It would force offload, but it would also halve the compute fabric and change the baseline throughput regime. That confounds “offload helps” with “the model is compute- and memory-constrained.”
- **TP16 is unlikely to help force offload.** It adds GPUs and aggregate HBM, and KV-head replication may occur when TP exceeds the natural KV-head sharding degree. Either effect moves away from the intended pressure point.
- Avoid TP values that do not align with the model's attention partitioning. The [checkpoint config](https://huggingface.co/nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8/blob/main/config.json) uses 128 attention heads and grouped attention; the observed TP8 cache shape is the strongest run-local compatibility evidence.

After U0.68 is validated at TP8, a separate TP4 experiment can answer a different question: whether a lower-GPU deployment can recover enough capacity through CPU/NVMe/CephFS to offset its compute and interconnect disadvantage. It should not replace the TP8 calibration.

## Secondary storage

```vega-lite
{"width":760,"height":320,"title":"Figure 9 \u2014 Secondary-tier mean throughput","data":{"values":[{"configuration":"CPU + NVMe","metric":"Write MiB/s","value":223.32633689129818},{"configuration":"CPU + NVMe","metric":"Read MiB/s","value":4.345641809964727},{"configuration":"CPU + CephFS","metric":"Write MiB/s","value":223.47771325635517},{"configuration":"CPU + CephFS","metric":"Read MiB/s","value":13.902689806734095}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Write MiB/s","Read MiB/s"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"value","type":"quantitative","title":"MiB/s","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"value","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

NVMe averages 223.3 MiB/s writes and 4.35 MiB/s reads, with 615.8 MiB/s peak writes, 21.4 MiB/s peak reads, ~287 write IOPS, and 6.6% mean device busy time. Storage is not saturated.

CephFS pool telemetry averages 223.5 MiB/s writes and 13.9 MiB/s reads, with 605.5 MiB/s peak writes and 197.7 MiB/s peak reads. The PVC grows to 373.0 GiB (12.14% of 3 TiB). CephFS records zero store-refusal warnings and no error storm. Its pool metrics are pool-scoped rather than pod-scoped, so the direct offload counters remain the authoritative application-level transfer measure.

Native time series:

- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/01 - Prompt-token sources - No offload|Prompt-token sources — no offload]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/02 - Prompt-token sources - CPU offload|Prompt-token sources — CPU offload]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/02b - Prompt-token sources - NVMe|Prompt-token sources — NVMe]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/02c - Prompt-token sources - CephFS|Prompt-token sources — CephFS]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/03 - GPU KV-cache utilization|GPU KV-cache utilization]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/04 - Scheduler pressure - Baseline and CPU|Scheduler pressure — baseline and CPU]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/05 - Scheduler pressure - Secondary tiers|Scheduler pressure — NVMe and CephFS]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/06 - KV transfer bandwidth|KV transfer bandwidth]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/07 - Generation throughput|Generation throughput]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/08 - TTFT P90|TTFT P90]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/09 - NVMe throughput|NVMe throughput]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/09a - NVMe IOPS|NVMe IOPS]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/09b - NVMe busy time|NVMe busy time]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/10 - CephFS throughput|CephFS throughput]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/10a - CephFS IOPS|CephFS IOPS]]
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison/10b - CephFS capacity|CephFS capacity]]

## Validity and limitations

- All four AIPerf profiles contain zero request errors. Model and router pods report zero restarts.
- Model logs contain zero CUDA OOMs, tracebacks, `cannot store blocks` warnings, and unavailable-block storms.
- Each run reaches the 1,800-second send cutoff and then the 30-second grace timeout. Only 2–3 in-flight requests are cancelled per cell.
- All cells use the same model identifier, image digest, TP8, U0.90, context, concurrency, workload, seed, and shared-memory size.
- CPU-only is a separate execution and NVMe runs on a different H100 node. CephFS runs after the baseline/NVMe pair. With one repetition, sub-percent deltas are not publication-grade.
- The NVMe hostPath records `cleanup: false`; an explicit clean-state proof is absent. Cache-bust limits cross-play reuse, but the storage cleanliness gap should be fixed in the rerun.
- The run does not pin a model repository commit. The exact model identifier is recorded, but checkpoint revision drift cannot be excluded.

## Conclusions

### Established by the data

- The four configurations are at performance parity.
- Tiering is enabled and writes about 1.0 TiB per tiered run.
- HBM is under pressure only to ~50%, so offloaded blocks rarely become necessary.
- CPU256 is too small relative to the 282.5 GiB HBM shelf to outlive local residency at the observed store rate.
- NVMe and CephFS perform real read-back, but only 18.7 GiB and ~0.15% of prompt tokens—far below a performance-relevant level.
- Storage bandwidth is not the bottleneck in these runs.

### Decision

Do not increase CPU capacity or change TP first. Keep the 256 GiB CPU tier and TP8, then calibrate U around **0.68**. This directly creates the missing condition: the live working set should fit most of the time, while retained conversation history must spill and later be restored.

## Next experiment

Run paired no-offload and CPU256 cells at U0.70, U0.68, and U0.66, seed 42 first. Accept a setting only when:

1. peak HBM KV utilization reaches 90–98%;
2. CPU→GPU restore bandwidth is sustained and external prompt share exceeds 5%;
3. local-compute prompt share falls below the U-matched baseline;
4. preemptions remain zero or transient;
5. mean waiting and P95 TTFT do not show a capacity cliff.

Then run the full four-cell matrix at the selected U with seeds 42, 123, and 456. Clean the NVMe directory explicitly, create a fresh CephFS PVC per run, and retain the direct transfer, prompt-source, scheduler, NVMe, and CephFS telemetry used here.

## Run registry and provenance

| Configuration | Execution / deployment | Node | MLflow | Disposition |
|---|---|---|---|---|
| No offload | `cpu-offloading-bd6e99` / `cpu-kv-offload-distributed-default` | `diadochos-hqxzk-gpu-h100-6kl5z` | [57e480071aa44f40a5fc0f629f280731](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/57e480071aa44f40a5fc0f629f280731?workspace=benchflow) | Directionally accepted |
| CPU offload 256 GiB | `cpu-offloading-0de396` / `rhoai-cpu-kv-offload-256g` | `diadochos-hqxzk-gpu-h100-6kl5z` | [c40d143bd48d4784b74066788cba65df](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/c40d143bd48d4784b74066788cba65df?workspace=benchflow) | Directionally accepted |
| CPU + NVMe | `cpu-offloading-18209a` / `multi-tier-offloading-nvme` | `diadochos-hqxzk-gpu-h100-mt46x` | [084819f27b5a47d2b3441e5c7743c6fb](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/084819f27b5a47d2b3441e5c7743c6fb?workspace=benchflow) | Directionally accepted |
| CPU + CephFS | `cpu-offloading-6dea19` / `multi-tier-offloading-cephfs` | `diadochos-hqxzk-gpu-h100-6kl5z` | [d293ce35e62546c9ac63a2ad1bd6ee83](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d293ce35e62546c9ac63a2ad1bd6ee83?workspace=benchflow) | Directionally accepted |


Artifacts were downloaded on 2026-07-30. Results use `results/profile_export_aiperf.json`, `benchmark/profile_export.jsonl`, complete AIPerf/model logs, rendered manifests, Kubernetes state, and native Prometheus JSON. Prompt-source shares integrate uniform-cadence rate samples and normalize them across sources. Companion figures preserve the native 15-second source cadence without smoothing or downsampling.