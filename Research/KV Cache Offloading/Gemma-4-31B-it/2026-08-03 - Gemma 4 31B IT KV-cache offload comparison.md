---
title: "2026-08-03 - Gemma 4 31B IT KV-cache offload comparison"
date: "2026-08-03"
type: "experiment-report"
topic: "KV Cache Offloading"
experiment: "MLflow experiment 318"
model: "google/gemma-4-31B-it"
model_revision: "not recorded"
runtime_image: "vllm/vllm-openai:v0.23.0"
runtime_image_digest: "sha256:3a1e7f5904e1a1192a02aa0086ceaffc33985d7044c7bb25b3a43d61bdbe3ac0"
vllm_version: "0.23.0"
dtype: "bfloat16"
quantization: "none"
kv_cache_dtype: "auto"
gpu_type: "H100"
gpu_count: 2
tensor_parallelism: 2
pipeline_parallelism: 1
replicas: 1
gpu_memory_utilization: 0.92
max_model_len: 131072
concurrency: 8
cpu_bytes: 274877906944
shared_memory: "300Gi"
random_seed: 42
duration_seconds: 1800
configuration_count: 4
baseline: "No offload"
status: "directionally valid single-seed comparison; NVMe conditionally accepted"
---
# 2026-08-03 - Gemma 4 31B IT KV-cache offload comparison

## Benchmark overview

This experiment compares `google/gemma-4-31B-it` under the stateful AgentX MVP workload across four KV-cache configurations: no offload, a 256 GiB CPU tier, CPU plus local NVMe, and CPU plus CephFS. Stable parameters are one TP2 replica on two H100s, vLLM 0.23.0, BF16 weights, automatic KV dtype, `gpu-memory-utilization=0.92`, 131,072-token context, concurrency 8, seed 42, a 1,800-second send window, prefix caching, and 300 GiB shared memory.

The report uses the deployed artifacts as the source of truth. The earlier [[2026-07-25 - AgentX offload failure matrix|July 25 matrix]] described TP4/U0.85/FP8 in prose, but its artifact-backed deployment was TP2/U0.92/BF16. The current matrix is also TP2/U0.92/BF16; the controlled changes from that earlier batch are concurrency 32 → 8, CPU capacity 64 → 256 GiB, and 200 → 300 GiB shared memory, plus the tuned CephFS I/O-thread setting.

## Executive summary

**CephFS is the decisive winner in this single-seed matrix.** It completes 336 requests at 0.1844 req/s and produces 145.4 output tok/s: **3.10× request throughput** and **4.41× output-token throughput** versus no offload. Mean TTFT falls from 27.47 to 3.32 seconds (87.9% lower), mean E2E falls from 131.12 to 33.52 seconds (74.4% lower), and completed sessions rise from 4 to 11.

**CPU-only is a clean second-place result.** Its 256 GiB tier supplies 45.0% of prompt tokens externally, doubles request throughput (+101.9%), raises output throughput by 135.5%, and halves mean E2E latency. This proves that this Gemma workload does benefit from CPU KV offload once the tier is large enough and concurrency is calibrated.

**The NVMe cell is functionally active but not clean.** Its median TTFT is excellent at 1.29 seconds, yet P95 reaches 295.68 seconds and P99 502.47 seconds. It averages 2.12 requests deferred, emits 15 `cannot store blocks` warnings concentrated in two request IDs, and finishes only 5 sessions. Raw NVMe bandwidth is not the obvious constraint: reads average 1,588.6 MiB/s, device busy time averages 38.9%, and CephFS has similar mean pool read bandwidth. The strongest run-local confound is configuration asymmetry: CephFS uses 64 read and 32 write threads, while NVMe uses filesystem-tier defaults.

**Validity verdict:** directionally valid for the large CephFS/CPU/no-offload separation; **NVMe is conditionally accepted** and should not be used for a definitive NVMe-versus-CephFS ranking. All four model pods have zero restarts, tracebacks, CUDA OOMs, and request errors. The baseline, CPU, and NVMe AIPerf phases hit the grace cutoff; CephFS drains without a grace timeout. Ceph itself remains `HEALTH_WARN` for low MON disk space and recent daemon crashes, but the run has complete pool telemetry, zero model-side store refusals, and zero request errors.

## Headline results

| Configuration | MLflow | Req/s | Output tok/s | Mean TTFT (s) | TTFT P95 (s) | Mean E2E (s) | Sessions | Branches | External prompt | KV mean / peak | Errors / cancelled |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| No offload | [3fca95c95ebf4607a3de6347c18eb0b2](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/3fca95c95ebf4607a3de6347c18eb0b2?workspace=benchflow) | 0.0595 | 33.0 | 27.47 | 70.25 | 131.12 | 4 / 12 | 8 / 10 | 0.0% | 59.9% / 93.8% | 0 / 9 |
| CPU offload 256 GiB | [0c12c93eb595463a922c9d492a8c4285](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/0c12c93eb595463a922c9d492a8c4285?workspace=benchflow) | 0.1202 | 77.7 | 11.93 | 48.68 | 65.09 | 8 / 16 | 19 / 21 | 45.0% | 54.9% / 91.2% | 0 / 3 |
| CPU + NVMe | [684447b51491409a9f0aedd2991c1909](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/684447b51491409a9f0aedd2991c1909?workspace=benchflow) | 0.0840 | 49.6 | 37.06 | 295.68 | 84.26 | 5 / 13 | 14 / 16 | 44.5% | 42.2% / 83.7% | 0 / 4 |
| CPU + CephFS | [1e8948753d4b40b29f8c963b8703c27c](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/1e8948753d4b40b29f8c963b8703c27c?workspace=benchflow) | 0.1844 | 145.4 | 3.31 | 13.36 | 33.52 | 11 / 19 | 27 / 28 | 48.8% | 42.7% / 88.1% | 0 / 0 |

Request throughput does not stand alone: AgentX branches as it progresses, so faster cells complete a different downstream mix. Output throughput, latency, prompt-source attribution, session/branch completion, scheduler state, and transfer telemetry all support the same broad result.

## Performance comparison

```vega-lite
{"width":740,"height":300,"title":"Figure 1 \u2014 Completed-request throughput","data":{"values":[{"configuration":"No offload","value":0.05953056706711495},{"configuration":"CPU offload 256 GiB","value":0.1202050896437268},{"configuration":"CPU + NVMe","value":0.0839805293475775},{"configuration":"CPU + CephFS","value":0.18436665521883885}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"value","type":"quantitative","title":"Requests per second","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"value","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Figure 1 shows the end-to-end productivity result. CPU-only reaches 2.02× baseline, NVMe 1.41×, and CephFS 3.10×. NVMe's average is depressed by its long stalls rather than a uniformly slow request distribution.

```vega-lite
{"width":740,"height":300,"title":"Figure 2 \u2014 Output-token throughput","data":{"values":[{"configuration":"No offload","value":32.97603963210253},{"configuration":"CPU offload 256 GiB","value":77.67444317736235},{"configuration":"CPU + NVMe","value":49.56398241258213},{"configuration":"CPU + CephFS","value":145.35609762126936}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"value","type":"quantitative","title":"Output tokens per second","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"value","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Figure 2 strengthens the conclusion in generated-token terms. CephFS produces 145.4 tok/s, CPU 77.7, NVMe 49.6, and baseline 33.0.

```vega-lite
{"width":760,"height":320,"title":"Figure 3 \u2014 Request latency","data":{"values":[{"configuration":"No offload","metric":"TTFT mean","seconds":27.47159783513084},{"configuration":"No offload","metric":"TTFT P50","seconds":20.188388691},{"configuration":"No offload","metric":"TTFT P95","seconds":70.25225036909994},{"configuration":"No offload","metric":"E2E mean","seconds":131.12142693395327},{"configuration":"No offload","metric":"E2E P95","seconds":409.2747672934999},{"configuration":"CPU offload 256 GiB","metric":"TTFT mean","seconds":11.93441241816895},{"configuration":"CPU offload 256 GiB","metric":"TTFT P50","seconds":3.1579555069999996},{"configuration":"CPU offload 256 GiB","metric":"TTFT P95","seconds":48.683224185399986},{"configuration":"CPU offload 256 GiB","metric":"E2E mean","seconds":65.09220337949772},{"configuration":"CPU offload 256 GiB","metric":"E2E P95","seconds":212.88844399889996},{"configuration":"CPU + NVMe","metric":"TTFT mean","seconds":37.061421067085526},{"configuration":"CPU + NVMe","metric":"TTFT P50","seconds":1.2893563044999998},{"configuration":"CPU + NVMe","metric":"TTFT P95","seconds":295.6770104106998},{"configuration":"CPU + NVMe","metric":"E2E mean","seconds":84.26433165508553},{"configuration":"CPU + NVMe","metric":"E2E P95","seconds":471.7232181947993},{"configuration":"CPU + CephFS","metric":"TTFT mean","seconds":3.3145574515238097},{"configuration":"CPU + CephFS","metric":"TTFT P50","seconds":0.9569405419999999},{"configuration":"CPU + CephFS","metric":"TTFT P95","seconds":13.364549921},{"configuration":"CPU + CephFS","metric":"E2E mean","seconds":33.52222525134523},{"configuration":"CPU + CephFS","metric":"E2E P95","seconds":117.48105828674998}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["TTFT mean","TTFT P50","TTFT P95","E2E mean","E2E P95"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"seconds","type":"quantitative","title":"Seconds","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"seconds","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Figure 3 exposes the NVMe bimodality. Its P50 is excellent while its P95 is by far the worst. CephFS is both fast and comparatively stable: TTFT P50/P95 are 0.96/13.37 seconds and E2E P50/P95 are 18.31/117.48 seconds.

```vega-lite
{"width":760,"height":320,"title":"Figure 11 \u2014 Tail inflation","data":{"values":[{"configuration":"No offload","metric":"TTFT P95 / P50","ratio":3.479834445649367},{"configuration":"No offload","metric":"E2E P95 / P50","ratio":4.564808725863351},{"configuration":"CPU offload 256 GiB","metric":"TTFT P95 / P50","ratio":15.416057660561584},{"configuration":"CPU offload 256 GiB","metric":"E2E P95 / P50","ratio":5.1835687468647125},{"configuration":"CPU + NVMe","metric":"TTFT P95 / P50","ratio":229.32141362224968},{"configuration":"CPU + NVMe","metric":"E2E P95 / P50","ratio":18.848016624532132},{"configuration":"CPU + CephFS","metric":"TTFT P95 / P50","ratio":13.965914635686948},{"configuration":"CPU + CephFS","metric":"E2E P95 / P50","ratio":6.417179557858319}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["TTFT P95 / P50","E2E P95 / P50"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"ratio","type":"quantitative","title":"P95 / P50 ratio","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"ratio","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Figure 11 makes the tail problem explicit. NVMe TTFT P95 is 229× its P50; CephFS is 14×, CPU 15×, and baseline 3.5×. A fast NVMe median therefore must not be summarized as healthy service.

```vega-lite
{"width":760,"height":320,"title":"Figure 4 \u2014 Request, session, and branch outcomes","data":{"values":[{"configuration":"No offload","metric":"Requests completed","count":107},{"configuration":"No offload","metric":"Requests cancelled","count":9},{"configuration":"No offload","metric":"Sessions completed","count":4},{"configuration":"No offload","metric":"Branches completed","count":8},{"configuration":"CPU offload 256 GiB","metric":"Requests completed","count":219},{"configuration":"CPU offload 256 GiB","metric":"Requests cancelled","count":3},{"configuration":"CPU offload 256 GiB","metric":"Sessions completed","count":8},{"configuration":"CPU offload 256 GiB","metric":"Branches completed","count":19},{"configuration":"CPU + NVMe","metric":"Requests completed","count":152},{"configuration":"CPU + NVMe","metric":"Requests cancelled","count":4},{"configuration":"CPU + NVMe","metric":"Sessions completed","count":5},{"configuration":"CPU + NVMe","metric":"Branches completed","count":14},{"configuration":"CPU + CephFS","metric":"Requests completed","count":336},{"configuration":"CPU + CephFS","metric":"Requests cancelled","count":0},{"configuration":"CPU + CephFS","metric":"Sessions completed","count":11},{"configuration":"CPU + CephFS","metric":"Branches completed","count":27}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Requests completed","Requests cancelled","Sessions completed","Branches completed"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"count","type":"quantitative","title":"Count","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"count","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Figure 4 captures stateful progress. CephFS completes 336 requests, 11 sessions, and 27 branches with no cancellations. CPU completes 219/8/19. NVMe completes 152/5/14. Baseline completes 107/4/8.

## Offload mechanism

```vega-lite
{"width":760,"height":320,"title":"Figure 5 \u2014 Prompt tokens by source","data":{"values":[{"configuration":"No offload","source":"External KV transfer","share":0.0},{"configuration":"No offload","source":"Local HBM hit","share":1.355173373032105},{"configuration":"No offload","source":"Local compute","share":98.64482662696788},{"configuration":"CPU offload 256 GiB","source":"External KV transfer","share":45.00472669054006},{"configuration":"CPU offload 256 GiB","source":"Local HBM hit","share":16.5935957925751},{"configuration":"CPU offload 256 GiB","source":"Local compute","share":38.40167751688484},{"configuration":"CPU + NVMe","source":"External KV transfer","share":44.50720021926748},{"configuration":"CPU + NVMe","source":"Local HBM hit","share":35.84206149841672},{"configuration":"CPU + NVMe","source":"Local compute","share":19.650738282315807},{"configuration":"CPU + CephFS","source":"External KV transfer","share":48.83175566962187},{"configuration":"CPU + CephFS","source":"Local HBM hit","share":41.59083006444287},{"configuration":"CPU + CephFS","source":"Local compute","share":9.577414265935253}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"share","type":"quantitative","title":"Integrated prompt-token rate share (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"Prompt source","scale":{"domain":["External KV transfer","Local HBM hit","Local compute"],"range":["#9467bd","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration"},{"field":"source"},{"field":"share","format":".2f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Prompt-source telemetry proves that offload is on the critical path. External restoration supplies 45.0% of prompt tokens for CPU, 44.5% for NVMe, and 48.8% for CephFS. Local recomputation falls from 98.6% in the baseline to 38.4%, 19.7%, and 9.6% respectively. CephFS wins because it combines the lowest recomputation share with a clean storage path; NVMe's even lower-than-CPU compute share does not translate into stable latency because requests are deferred or retried.

```vega-lite
{"width":760,"height":320,"title":"Figure 6 \u2014 HBM KV-cache utilization","data":{"values":[{"configuration":"No offload","metric":"Mean","percent":59.91229407738529},{"configuration":"No offload","metric":"Peak","percent":93.78927419622968},{"configuration":"CPU offload 256 GiB","metric":"Mean","percent":54.884961931043655},{"configuration":"CPU offload 256 GiB","metric":"Peak","percent":91.17265684742495},{"configuration":"CPU + NVMe","metric":"Mean","percent":42.19670661394375},{"configuration":"CPU + NVMe","metric":"Peak","percent":83.68853276590086},{"configuration":"CPU + CephFS","metric":"Mean","percent":42.70160292286834},{"configuration":"CPU + CephFS","metric":"Peak","percent":88.08391794394387}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Mean","Peak"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"percent","type":"quantitative","title":"KV-cache utilization (%)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"percent","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Peak HBM KV utilization is 83.7–93.8%, so U0.92/C8 creates meaningful pressure without persistent 100% saturation. Mean occupancy is lower because startup, workload waves, and drain are included. There are zero preemptions in all four cells. This is a much healthier sizing point than the earlier C32 batch, which sat near 100% and failed to make session progress in the secondary tiers.

```vega-lite
{"width":760,"height":320,"title":"Figure 7 \u2014 Mean scheduler state","data":{"values":[{"configuration":"No offload","metric":"Running","requests":7.2},{"configuration":"No offload","metric":"Waiting","requests":1.103448275862069},{"configuration":"No offload","metric":"Deferred","requests":0.0},{"configuration":"CPU offload 256 GiB","metric":"Running","requests":6.402777777777778},{"configuration":"CPU offload 256 GiB","metric":"Waiting","requests":0.7638888888888888},{"configuration":"CPU offload 256 GiB","metric":"Deferred","requests":0.041666666666666664},{"configuration":"CPU + NVMe","metric":"Running","requests":4.958620689655173},{"configuration":"CPU + NVMe","metric":"Waiting","requests":2.537931034482759},{"configuration":"CPU + NVMe","metric":"Deferred","requests":2.1241379310344826},{"configuration":"CPU + CephFS","metric":"Running","requests":5.638888888888889},{"configuration":"CPU + CephFS","metric":"Waiting","requests":0.5833333333333334},{"configuration":"CPU + CephFS","metric":"Deferred","requests":0.25}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Running","Waiting","Deferred"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"requests","type":"quantitative","title":"Requests","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"requests","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Baseline and CPU have about 1.10 and 0.76 mean waiting requests. CephFS averages only 0.58 waiting. NVMe averages 2.54, of which 2.12 are explicitly deferred; that is the clearest scheduler symptom behind its extreme tail.

```vega-lite
{"width":760,"height":320,"title":"Figure 8 \u2014 Cumulative KV transfer volume","data":{"values":[{"configuration":"CPU offload 256 GiB","direction":"GPU \u2192 CPU","gib":9311.3232421875},{"configuration":"CPU offload 256 GiB","direction":"CPU \u2192 GPU","gib":1142.8271484375},{"configuration":"CPU + NVMe","direction":"GPU \u2192 CPU","gib":2834.82421875},{"configuration":"CPU + NVMe","direction":"CPU \u2192 GPU","gib":854.833984375},{"configuration":"CPU + CephFS","direction":"GPU \u2192 CPU","gib":3393.2177734375},{"configuration":"CPU + CephFS","direction":"CPU \u2192 GPU","gib":1855.1220703125}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"direction","type":"nominal","title":"Metric","sort":["GPU \u2192 CPU","CPU \u2192 GPU"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"gib","type":"quantitative","title":"GiB","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"direction","type":"nominal"},{"field":"gib","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU restores 1,142.8 GiB and stores 9,311.3 GiB. NVMe restores 854.8 GiB and stores 2,834.8 GiB. CephFS restores 1,855.1 GiB and stores 3,393.2 GiB. These are direct vLLM CPU↔GPU counters; secondary-storage pool/device traffic is shown separately and should not be conflated with PCIe movement.

## NVMe performance

```vega-lite
{"width":760,"height":320,"title":"Figure 9 \u2014 Secondary-tier mean throughput","data":{"values":[{"configuration":"CPU + NVMe","metric":"Read","value":1588.5637931034482},{"configuration":"CPU + NVMe","metric":"Write","value":531.9926544540231},{"configuration":"CPU + CephFS","metric":"Read","value":1571.7636022021197},{"configuration":"CPU + CephFS","metric":"Write","value":793.4059480128}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Read","Write"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"value","type":"quantitative","title":"MiB/s","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"value","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

```vega-lite
{"width":760,"height":320,"title":"Figure 10 \u2014 Secondary-tier mean IOPS","data":{"values":[{"configuration":"CPU + NVMe","metric":"Read","value":1270.913103448276},{"configuration":"CPU + NVMe","metric":"Write","value":473.3185696040868},{"configuration":"CPU + CephFS","metric":"Read","value":628.715984405458},{"configuration":"CPU + CephFS","metric":"Write","value":941.8324317738791}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Read","Write"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"value","type":"quantitative","title":"IOPS","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"value","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

The NVMe device averages 1,588.6 MiB/s reads and 532.0 MiB/s writes, peaking at 2,981.0 and 1,800.2 MiB/s. It averages 1,270.9 read IOPS and 473.3 write IOPS; `nvme0n1` is 38.9% busy on average and 75.0% at peak. Filesystem usage rises from 22.7% before workload activity to 38.8% at the end and peak; the run plan requests cleanup of `/var/mnt/benchflow-nvme/benchflow-kv-cache`.

These numbers do **not** prove media saturation. The cell's direct CPU→GPU restoration averages only 439.9 MiB/s versus 979.3 MiB/s for CephFS, and the model log records two requests repeatedly failing to store blocks (six and nine warnings). The primary hypothesis is the filesystem-tier execution path—especially default I/O-thread counts or block-store coordination—not raw sequential read bandwidth.

## CephFS performance

CephFS pool telemetry covers the full 35-minute captured benchmark interval, including warm-up and profiling. Pool `kvcache-fs-data0` reads average 1,571.8 MiB/s and peak at 4,640.1 MiB/s; writes average 793.4 and peak at 1,979.6 MiB/s. Read/write IOPS average 628.7/941.8 and peak at 1,856.0/2,339.4. The fresh 3 TiB PVC grows from 0 to 1,674.0 GiB (54.49%).

The deployment uses a single-replica NVMe data pool, two active MDS daemons, 12 GiB OSD limits, host networking, and explicit `n_read_threads=64`, `n_write_threads=32`. Application logs have zero `cannot store blocks`, unavailable-block storms, tracebacks, or OOMs. Cluster status at collection is `HEALTH_WARN` due to low MON disk space and three recent daemon crashes; one MON liveness restart is present in cluster events. This is an infrastructure caveat, not an observed application failure in this cell.

## What changed from the failed July matrix

| Dimension | July 25 batch | August 3 batch | Consequence |
|---|---:|---:|---|
| Deployed topology | TP2 / U0.92 / BF16 | TP2 / U0.92 / BF16 | Same artifact-backed model regime |
| Concurrency | 32 | 8 | Avoids persistent 100% KV saturation |
| CPU tier | 64 GiB | 256 GiB | Much deeper reusable-history shelf |
| Shared memory | 200 GiB | 300 GiB | Matches larger host tier |
| NVMe store refusals | 37 across 8 requests | 15 across 2 requests | Improved, not eliminated |
| CephFS store refusals | 273 across 12 requests | 0 | Failure mode eliminated |
| NVMe completed sessions | 0 | 5 | Functionally active but tail-degraded |
| CephFS completed sessions | 0 | 11 | Cleanest and fastest cell |

This comparison is explanatory, not a controlled attribution: concurrency, CPU capacity, shared memory, and CephFS tuning all change together. It establishes that the new operating point works; it does not assign a percentage of improvement to any single change.

## GPU memory utilization and tensor parallelism

**Keep `gpu-memory-utilization=0.92` and TP2 for the confirmation run.** Startup reports 30.38 GiB of model weights and 36.72 GiB of GPU KV capacity per GPU, for 564,730 aggregate GPU KV tokens and 4.31× theoretical full-context concurrency. Observed peak KV occupancy of 84–94% at C8 is exactly the desired pressure band: enough to force offload, below persistent saturation, and zero preemptions.

Do not lower U merely to create more offload in this workload. The current configuration already sources roughly half of prompt tokens externally. Lowering U would shrink the HBM shelf, likely increasing churn and tail latency without addressing NVMe's store/deferred-path defect. TP1 would roughly double per-GPU weight pressure and change compute parallelism; TP4 would add aggregate HBM and weaken pressure. Either would confound the storage comparison. Change TP only in a separate topology study.

## Recommended next experiment

1. Repeat the four cells unchanged at seeds 42, 123, and 456; preserve U0.92, TP2, C8, CPU256, and the same image digest.
2. Give NVMe the same `n_read_threads=64` and `n_write_threads=32` as CephFS and require a fresh, empty path. Compare it with the current-default NVMe cell in an A/B test.
3. Reject any cell with `cannot store blocks`, request errors, model/router restarts, or non-zero preemptions. Record grace-timeout cancellations separately from server errors.
4. Require Ceph `HEALTH_OK` before the confirmation run, or explicitly document an approved warning set. Preserve the full pool/PVC/health window.
5. If all three seeds stay below 80% peak KV use, raise concurrency to 10; if any cell exceeds 97% or preempts, return to C8. Do not change U and concurrency in the same batch.

## AIPerf command

The command is identical across all four configurations; BenchFlow substitutes only the deployment-specific service URL and artifact directory.

```bash
aiperf profile --model 'google/gemma-4-31B-it' \
  --url 'https://cpu-offloading-m1-2931d9a25c-kserve-workload-svc.benchflow.svc.cluster.local:8000' \
  --artifact-dir '/benchmark-results/remote-jobs/benchflow-benchmark-cpu-offloading-7744/benchmark' \
  --ui None --benchmark-duration 1800 --concurrency 8 \
  --endpoint '/v1/chat/completions' --endpoint-type 'chat' \
  --hf-weka-repo 'semianalysisai/cc-traces-weka-with-subagents-060826' \
  --max-context-length 131072 --public-dataset 'weka_hf' --random-seed 42 \
  --scenario 'inferencex-agentx-mvp' --streaming \
  --tokenizer-trust-remote-code --use-server-token-count \
  --tokenizer 'google/gemma-4-31B-it'
```

## Native-cadence figures

- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/01 - Prompt-token sources - No offload|Prompt-token sources — no offload]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/02 - Prompt-token sources - CPU offload|Prompt-token sources — CPU offload]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/03 - Prompt-token sources - NVMe|Prompt-token sources — NVMe]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/04 - Prompt-token sources - CephFS|Prompt-token sources — CephFS]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/05 - GPU KV-cache utilization|GPU KV-cache utilization]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/06 - Scheduler pressure - Baseline and CPU|Scheduler pressure — baseline and CPU]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/07 - Scheduler pressure - Secondary tiers|Scheduler pressure — secondary tiers]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/08 - CPU KV transfer bandwidth|CPU KV transfer bandwidth]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/09 - Secondary-tier KV transfer bandwidth|Secondary-tier KV transfer bandwidth]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/10 - Generation throughput|Generation throughput]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/11 - TTFT P90|TTFT P90]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/12 - NVMe throughput|NVMe throughput]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/13 - NVMe IOPS|NVMe IOPS]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/14 - NVMe busy time|NVMe busy time]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/15 - NVMe filesystem usage|NVMe filesystem usage]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/16 - CephFS throughput|CephFS throughput]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/17 - CephFS IOPS|CephFS IOPS]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/18 - CephFS capacity|CephFS capacity]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison/19 - CephFS health|CephFS health]]

## Validity and provenance

- All four cells match on model, image digest, vLLM version, TP2, U0.92, BF16, context, concurrency, seed, workload, duration, prefix-caching settings, and shared memory.
- Model and router pods report zero restarts. AIPerf reports zero request errors in every cell; model logs report zero tracebacks, CUDA OOMs, and unavailable-block errors.
- Baseline, CPU, and NVMe reach the 30-second grace timeout with 9, 3, and 4 cancelled requests. CephFS drains cleanly with zero cancellations. Boundary cancellations are not counted as server errors, but they limit exact request-mix parity.
- NVMe has 15 store refusals across two request IDs; its latency and throughput remain useful as a degraded observation, not a clean backend estimate.
- This is one seed. The effect size and matching mechanism telemetry justify a directional conclusion, not a confidence interval.
- The model repository revision is not pinned. The OCI image digest is pinned in the captured pod status.
- Results come from AIPerf profile JSON/JSONL, complete AIPerf and model logs, run plans, rendered manifests, Kubernetes/Ceph state, and native Prometheus JSON. Rate shares integrate uniform 15-second samples without smoothing or downsampling.

## Run registry

| Configuration | Execution / deployment | Node | MLflow | Disposition |
|---|---|---|---|---|
| No offload | `cpu-offloading-1b87a2` / `cpu-kv-offload-distributed-default` | `diadochos-hqxzk-gpu-h100-fx7c8` | [3fca95c95ebf4607a3de6347c18eb0b2](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/3fca95c95ebf4607a3de6347c18eb0b2?workspace=benchflow) | Directionally accepted |
| CPU offload 256 GiB | `cpu-offloading-d0326b` / `rhoai-cpu-kv-offload-256g` | `diadochos-hqxzk-gpu-h100-fx7c8` | [0c12c93eb595463a922c9d492a8c4285](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/0c12c93eb595463a922c9d492a8c4285?workspace=benchflow) | Directionally accepted |
| CPU + NVMe | `cpu-offloading-0d5f53` / `multi-tier-offloading-nvme` | `diadochos-hqxzk-gpu-h100-mt46x` | [684447b51491409a9f0aedd2991c1909](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/684447b51491409a9f0aedd2991c1909?workspace=benchflow) | Conditionally accepted (15 store refusals; extreme tail) |
| CPU + CephFS | `cpu-offloading-c23500` / `multi-tier-offloading-cephfs` | `diadochos-hqxzk-gpu-h100-mt46x` | [1e8948753d4b40b29f8c963b8703c27c](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/1e8948753d4b40b29f8c963b8703c27c?workspace=benchflow) | Directionally accepted |

Artifacts were downloaded and analyzed on 2026-08-03.