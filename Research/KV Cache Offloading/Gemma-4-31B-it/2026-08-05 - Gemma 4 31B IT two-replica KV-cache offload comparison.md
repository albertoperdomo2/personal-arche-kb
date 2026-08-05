---
title: "2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison"
date: "2026-08-05"
type: "experiment-report"
topic: "KV Cache Offloading"
experiment: "MLflow experiment 318"
model: "google/gemma-4-31B-it"
runtime_image: "vllm/vllm-openai:v0.23.0"
vllm_version: "0.23.0"
dtype: "bfloat16"
quantization: "none"
kv_cache_dtype: "auto"
gpu_type: "H100"
gpu_count: 4
tensor_parallelism: 2
pipeline_parallelism: 1
replicas: 2
gpu_memory_utilization: 0.92
max_model_len: 131072
concurrency: 16
cpu_bytes_per_replica: 274877906944
shared_memory_per_replica: "300Gi"
random_seed: 42
duration_seconds: 1800
configuration_count: 4
baseline: "No offload"
status: "valid failure characterization; not valid for a storage-backend ranking"
---
# 2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison

## Benchmark overview

This experiment compares `google/gemma-4-31B-it` across four KV-cache configurations using **two TP2 replicas** (four H100 GPUs per configuration) under the stateful AgentX MVP workload. Stable parameters are vLLM 0.23.0, BF16 weights, automatic KV dtype, `gpu-memory-utilization=0.92`, 131,072-token context, AIPerf concurrency 16, seed 42, a 1,800-second send window, prefix caching, and 300 GiB shared memory per replica. Each offload replica receives a 256 GiB CPU tier; NVMe uses 64 read/64 write threads and CephFS uses 64 read/32 write threads.

All four configurations were deployed at the same time. Each of the two H100 nodes hosted one replica from every configuration, so each node ran four TP2 model servers and all eight GPUs were allocated. That makes the matrix controlled at the application level but not isolated at the host-memory, CPU, network, or storage level.

## Executive summary

**The no-offload baseline is the observed winner.** It completes 220 requests at 0.1208 req/s and 89.2 output tok/s. CPU-only is close but slightly worse: -7.5% request throughput and -5.9% output throughput. NVMe loses 45.1% request throughput and 50.0% output throughput. CephFS loses 81.6% and 86.7%. TTFT P95 rises from 99.6 seconds in the baseline to 127.8 seconds for CPU, 549.0 seconds for NVMe, and 671.7 seconds for CephFS.

**This is not evidence that the benchmark needs more concurrency.** The no-offload cell delivers 2.03× the August 3 single-replica throughput—almost ideal scaling—while the tiered cells already carry waiting queues. NVMe averages 8.47 waiting requests and CephFS 7.20 during the profiling window. AgentX subagents can also make server-side request pressure exceed the AIPerf parent-trajectory setting. Raising C16 is therefore more likely to deepen the backlog than to reveal a hidden offload benefit.

**The strongest explanation is multi-replica cache locality and promotion backpressure, not raw media saturation.** The EPP uses an *approximate* prefix-cache producer, with prefix score weight 3 versus queue score weight 2. The tiered runs show severe per-replica skew, active external restoration, and a progressive generation-throughput collapse. NVMe reads average 2.01 GiB/s across the two devices while mean device busy time remains 20.7% and 34.0% (peaks 51.5% and 73.5%). CephFS reads average 1.52 GiB/s and its PVC grows by 646.6 GiB during profiling. The tiers are active, but useful session progress is not keeping pace.

The likely causal chain is: approximate routing sends a continuation to a replica whose useful KV state is incomplete or still being promoted; the scheduler repeats asynchronous lookup while the secondary tier promotes data into CPU and then GPU; queue and replica imbalance grow; generation falls even though storage remains busy. vLLM's tiering design explicitly promotes secondary data through CPU before GPU and returns a temporary miss while an asynchronous promotion is in progress, causing the scheduler to retry ([vLLM tiering RFC](https://github.com/vllm-project/vllm/issues/38260)). Precise llm-d routing instead consumes actual per-pod KV block events and scores the exact resident fraction ([llm-d precise prefix-cache routing guide](https://github.com/llm-d/llm-d/blob/main/guides/precise-prefix-cache-aware/README.md)). This run did not use that precise event path.

**Validity verdict:** valid as a failure-characterization experiment, but not valid for ranking NVMe against CephFS or for concluding that two-replica offload cannot scale. The matrix has one seed, all four configurations share the same two hosts, and tier retrieval latency is not directly captured. There are no request errors, OOMs, tracebacks, restarts, or memory failures; all cells hit the grace-period timeout.

## Headline results

| Configuration | MLflow | Req/s | Output tok/s | TTFT P50 (s) | TTFT P95 (s) | E2E P95 (s) | Sessions | Branches | External prompt | Generation first → last (tok/s) | Errors / cancelled |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| No offload | [e27348dff257466db48a3eee79f8182a](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/e27348dff257466db48a3eee79f8182a?workspace=benchflow) | 0.1208 | 89.2 | 29.36 | 99.57 | 426.56 | 6 / 21 | 31 / 34 | 0.0% | 92.7 → 57.4 | 0 / 6 |
| CPU offload 256 GiB | [0ff6322040384fe3b98ec8c8725d08ce](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/0ff6322040384fe3b98ec8c8725d08ce?workspace=benchflow) | 0.1118 | 83.9 | 24.83 | 127.83 | 401.97 | 5 / 20 | 29 / 33 | 12.1% | 85.3 → 65.0 | 0 / 15 |
| CPU + NVMe | [a293c9c7fefe4028a5d438bc3b5308ef](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/a293c9c7fefe4028a5d438bc3b5308ef?workspace=benchflow) | 0.0663 | 44.6 | 13.86 | 549.00 | 786.95 | 4 / 19 | 27 / 29 | 47.7% | 94.8 → 3.4 | 0 / 17 |
| CPU + CephFS | [a4176ee0e2db4309aee724ec9a86b168](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/a4176ee0e2db4309aee724ec9a86b168?workspace=benchflow) | 0.0222 | 11.9 | 23.18 | 671.69 | 1450.10 | 2 / 17 | 1 / 20 | 27.1% | 92.6 → 8.3 | 0 / 16 |

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":320,"title":"Figure 1 \u2014 Completed-request throughput","data":{"values":[{"configuration":"No offload","value":0.12080113534486867},{"configuration":"CPU offload 256 GiB","value":0.11177352926480702},{"configuration":"CPU + NVMe","value":0.0663469451117035},{"configuration":"CPU + CephFS","value":0.022170930231950305}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20}},"y":{"field":"value","type":"quantitative","title":"Requests per second"},"color":{"field":"configuration","type":"nominal","legend":null,"scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration"},{"field":"value","format":".3f"}]}}
```

Figure 1 is the end-to-end result. CPU-only remains near parity, but both filesystem tiers regress sharply.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":320,"title":"Figure 2 \u2014 Output-token throughput","data":{"values":[{"configuration":"No offload","value":89.16771076660555},{"configuration":"CPU offload 256 GiB","value":83.87607369577843},{"configuration":"CPU + NVMe","value":44.601025016459005},{"configuration":"CPU + CephFS","value":11.864773313628206}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20}},"y":{"field":"value","type":"quantitative","title":"Output tokens per second"},"color":{"field":"configuration","type":"nominal","legend":null,"scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration"},{"field":"value","format":".3f"}]}}
```

Figure 2 shows that the same ranking holds in generated-token productivity, not only request counts.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":340,"title":"Figure 3 \u2014 Request latency","data":{"values":[{"configuration":"No offload","metric":"TTFT P50","seconds":29.361100772999997},{"configuration":"No offload","metric":"TTFT P95","seconds":99.56524457185},{"configuration":"No offload","metric":"E2E P50","seconds":93.95661709849999},{"configuration":"No offload","metric":"E2E P95","seconds":426.5628007365486},{"configuration":"CPU offload 256 GiB","metric":"TTFT P50","seconds":24.8287046925},{"configuration":"CPU offload 256 GiB","metric":"TTFT P95","seconds":127.83229358064997},{"configuration":"CPU offload 256 GiB","metric":"E2E P50","seconds":94.19197395049999},{"configuration":"CPU offload 256 GiB","metric":"E2E P95","seconds":401.9694527990994},{"configuration":"CPU + NVMe","metric":"TTFT P50","seconds":13.860459489999998},{"configuration":"CPU + NVMe","metric":"TTFT P95","seconds":548.9980993139999},{"configuration":"CPU + NVMe","metric":"E2E P50","seconds":43.218294900000004},{"configuration":"CPU + NVMe","metric":"E2E P95","seconds":786.9525039281999},{"configuration":"CPU + CephFS","metric":"TTFT P50","seconds":23.1823618445},{"configuration":"CPU + CephFS","metric":"TTFT P95","seconds":671.6918484634491},{"configuration":"CPU + CephFS","metric":"E2E P50","seconds":59.784891654},{"configuration":"CPU + CephFS","metric":"E2E P95","seconds":1450.1037641249493}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20},"title":"Configuration"},"xOffset":{"field":"metric","sort":["TTFT P50","TTFT P95","E2E P50","E2E P95"]},"y":{"field":"seconds","type":"quantitative","title":"Seconds"},"color":{"field":"metric","type":"nominal","sort":["TTFT P50","TTFT P95","E2E P50","E2E P95"]},"tooltip":[{"field":"configuration"},{"field":"metric"},{"field":"seconds","format":".3f"}]}}
```

Figure 3 shows a particularly important split: NVMe retains a fast TTFT median of 13.9 seconds but develops a 549.0-second P95; CephFS has a 23.2-second median and 671.7-second P95. The issue is a long-tail stall mode rather than uniformly slow service.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":340,"title":"Figure 4 \u2014 Stateful workload outcomes","data":{"values":[{"configuration":"No offload","metric":"Requests completed","count":220},{"configuration":"No offload","metric":"Requests cancelled","count":6},{"configuration":"No offload","metric":"Sessions completed","count":6},{"configuration":"No offload","metric":"Branches completed","count":31},{"configuration":"CPU offload 256 GiB","metric":"Requests completed","count":202},{"configuration":"CPU offload 256 GiB","metric":"Requests cancelled","count":15},{"configuration":"CPU offload 256 GiB","metric":"Sessions completed","count":5},{"configuration":"CPU offload 256 GiB","metric":"Branches completed","count":29},{"configuration":"CPU + NVMe","metric":"Requests completed","count":117},{"configuration":"CPU + NVMe","metric":"Requests cancelled","count":17},{"configuration":"CPU + NVMe","metric":"Sessions completed","count":4},{"configuration":"CPU + NVMe","metric":"Branches completed","count":27},{"configuration":"CPU + CephFS","metric":"Requests completed","count":40},{"configuration":"CPU + CephFS","metric":"Requests cancelled","count":16},{"configuration":"CPU + CephFS","metric":"Sessions completed","count":2},{"configuration":"CPU + CephFS","metric":"Branches completed","count":1}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20},"title":"Configuration"},"xOffset":{"field":"metric","sort":["Requests completed","Requests cancelled","Sessions completed","Branches completed"]},"y":{"field":"count","type":"quantitative","title":"Count"},"color":{"field":"metric","type":"nominal","sort":["Requests completed","Requests cancelled","Sessions completed","Branches completed"]},"tooltip":[{"field":"configuration"},{"field":"metric"},{"field":"count","format":".3f"}]}}
```

Only 2 of 17 CephFS sessions and 4 of 19 NVMe sessions complete. CephFS finishes only 1 of 20 spawned branches. These stateful outcomes make the throughput loss operationally meaningful.

## Scaling and the concurrency hypothesis

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":320,"title":"Figure 5 \u2014 Two-replica scaling efficiency versus the August 3 single-replica result","data":{"values":[{"configuration":"No offload","value":101.46137897557614},{"configuration":"CPU offload 256 GiB","value":46.492878526187354},{"configuration":"CPU + NVMe","value":18.73297299382885},{"configuration":"CPU + CephFS","value":6.012716546873981}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20}},"y":{"field":"value","type":"quantitative","title":"Observed / ideal 2\u00d7 request throughput (%)"},"color":{"field":"configuration","type":"nominal","legend":null,"scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration"},{"field":"value","format":".3f"}]}}
```

The prior August 3 report used one TP2 replica at C8. Ideal two-replica scaling would double those throughputs. The new baseline reaches 101.5% of that ideal, CPU 46.5%, NVMe 18.7%, and CephFS 6.0%. This historical comparison is directional because the runs occurred on different days and the new matrix is co-tenant, but it decisively rejects “C16 does not drive two replicas” as the primary explanation.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":340,"title":"Figure 7 \u2014 Mean scheduler pressure","data":{"values":[{"configuration":"No offload","metric":"Running","requests":13.9},{"configuration":"No offload","metric":"Waiting","requests":3.2666666666666666},{"configuration":"CPU offload 256 GiB","metric":"Running","requests":12.25},{"configuration":"CPU offload 256 GiB","metric":"Waiting","requests":4.15},{"configuration":"CPU + NVMe","metric":"Running","requests":9.625},{"configuration":"CPU + NVMe","metric":"Waiting","requests":8.466666666666667},{"configuration":"CPU + CephFS","metric":"Running","requests":6.833333333333333},{"configuration":"CPU + CephFS","metric":"Waiting","requests":7.2}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20},"title":"Configuration"},"xOffset":{"field":"metric","sort":["Running","Waiting"]},"y":{"field":"requests","type":"quantitative","title":"Requests"},"color":{"field":"metric","type":"nominal","sort":["Running","Waiting"]},"tooltip":[{"field":"configuration"},{"field":"metric"},{"field":"requests","format":".3f"}]}}
```

Mean running/waiting counts are 13.90/3.27 for baseline, 12.25/4.15 for CPU, 9.63/8.47 for NVMe, and 6.83/7.20 for CephFS. The storage tiers have *more* waiting and *less* useful running work. A concurrency increase cannot repair a promotion or routing bottleneck; it feeds it.

Recommended concurrency work is a **response-curve experiment**, not a blind increase: C8, C12, C16, and C24, one storage configuration at a time, with fixed replica routing. Plot throughput and TTFT P95 against offered concurrency and select the knee before waiting rises sharply. Based on these data, C16 is already beyond the knee for NVMe and CephFS.

## Offload behavior and replica locality

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":330,"title":"Figure 6 \u2014 Prompt tokens by source during profiling","data":{"values":[{"configuration":"No offload","source":"External KV transfer","share":0.0},{"configuration":"No offload","source":"Local HBM","share":6.344089616205428},{"configuration":"No offload","source":"Local recomputation","share":93.65591038379458},{"configuration":"CPU offload 256 GiB","source":"External KV transfer","share":12.09490339554548},{"configuration":"CPU offload 256 GiB","source":"Local HBM","share":13.054969431912115},{"configuration":"CPU offload 256 GiB","source":"Local recomputation","share":74.85012717254241},{"configuration":"CPU + NVMe","source":"External KV transfer","share":47.70459197555799},{"configuration":"CPU + NVMe","source":"Local HBM","share":17.997866937341943},{"configuration":"CPU + NVMe","source":"Local recomputation","share":34.297541087100065},{"configuration":"CPU + CephFS","source":"External KV transfer","share":27.082110101429283},{"configuration":"CPU + CephFS","source":"Local HBM","share":32.59915817077129},{"configuration":"CPU + CephFS","source":"Local recomputation","share":40.318731727799424}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20}},"y":{"field":"share","type":"quantitative","title":"Integrated rate share (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","scale":{"domain":["External KV transfer","Local HBM","Local recomputation"],"range":["#9467bd","#17becf","#7f7f7f"]},"sort":["External KV transfer","Local HBM","Local recomputation"]},"tooltip":[{"field":"configuration"},{"field":"source"},{"field":"share","format":".2f"}]}}
```

The tiers are being used. External KV supplies 12.1% of CPU prompt tokens, 47.7% of NVMe prompt tokens, and 27.1% of CephFS prompt tokens. NVMe and CephFS therefore do not lose because offload is disabled. CPU's external share collapses from 45.0% in the prior single-replica C8 run to 12.1% here, which is consistent with reuse being split or misdirected across replicas.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":340,"title":"Figure 8 \u2014 Mean external-prefix hit rate by replica","data":{"values":[{"configuration":"No offload","metric":"Replica A","percent":0.0},{"configuration":"No offload","metric":"Replica B","percent":0.0},{"configuration":"CPU offload 256 GiB","metric":"Replica A","percent":25.731807663327835},{"configuration":"CPU offload 256 GiB","metric":"Replica B","percent":5.338306722406281},{"configuration":"CPU + NVMe","metric":"Replica A","percent":32.99038346567535},{"configuration":"CPU + NVMe","metric":"Replica B","percent":38.6678040522347},{"configuration":"CPU + CephFS","metric":"Replica A","percent":24.898547431049426},{"configuration":"CPU + CephFS","metric":"Replica B","percent":31.05945155444309}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20},"title":"Configuration"},"xOffset":{"field":"metric","sort":["Replica A","Replica B"]},"y":{"field":"percent","type":"quantitative","title":"Hit rate (%)"},"color":{"field":"metric","type":"nominal","sort":["Replica A","Replica B"]},"tooltip":[{"field":"configuration"},{"field":"metric"},{"field":"percent","format":".3f"}]}}
```

Mean external-prefix hit rates are highly uneven by replica: CPU averages 25.7% versus 5.3%; NVMe 33.0% versus 38.7%; CephFS 24.9% versus 31.1%, with long stretches at zero and late spikes near 99%. Per-replica scheduler data are also skewed. CPU KV occupancy averages 86.7% on one replica and 14.2% on the other; CephFS averages 0.8 running requests on one replica and 6.0 on the other. Baseline generation remains balanced at 44.8 versus 47.2 tok/s.

The service-level scrape adds evidence compatible with repeated lookup: one sampled NVMe endpoint reports 160.4 million local prefix-query tokens for only 0.685 million local hit tokens; the sampled CephFS endpoint reports 37.2 million queries and zero local hits, while both still report external hits. Because this export is not pod-labelled and may observe only one service-selected backend, it is a diagnostic clue rather than a cluster-wide counter.

## KV retrieval time: what is and is not available

The MLflow artifacts do **not** contain a valid secondary-tier retrieval-latency series. The five configured Prometheus queries for transfer duration and tiering lookup P50/P90/P99 all return zero series. The query pack expects older or tier-specific metric names, while vLLM 0.23.0 exposes generic `vllm:kv_offload_total_time`, `vllm:kv_offload_total_bytes`, and `vllm:kv_offload_size` metrics at the service endpoint.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":340,"title":"Figure 9 \u2014 Connector operation time from the AIPerf service scrape","data":{"values":[{"configuration":"CPU offload 256 GiB","metric":"CPU \u2192 GPU","milliseconds":62.92110981260026},{"configuration":"CPU offload 256 GiB","metric":"GPU \u2192 CPU","milliseconds":65.90083936582661},{"configuration":"CPU + NVMe","metric":"CPU \u2192 GPU","milliseconds":60.400927690359254},{"configuration":"CPU + NVMe","metric":"GPU \u2192 CPU","milliseconds":57.63912740265951},{"configuration":"CPU + CephFS","metric":"CPU \u2192 GPU","milliseconds":66.05898257664272},{"configuration":"CPU + CephFS","metric":"GPU \u2192 CPU","milliseconds":53.566704994001846}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20},"title":"Configuration"},"xOffset":{"field":"metric","sort":["CPU \u2192 GPU","GPU \u2192 CPU"]},"y":{"field":"milliseconds","type":"quantitative","title":"Milliseconds per operation"},"color":{"field":"metric","type":"nominal","sort":["CPU \u2192 GPU","GPU \u2192 CPU"]},"tooltip":[{"field":"configuration"},{"field":"metric"},{"field":"milliseconds","format":".3f"}]}}
```

| Configuration | Connector direction | Operations | Mean operation size (GiB) | Mean time (ms) | Effective GiB/s |
|---|---|---:|---:|---:|---:|
| CPU offload 256 GiB | CPU → GPU | 28 | 2.92 | 62.92 | 46.43 |
| CPU offload 256 GiB | GPU → CPU | 698 | 2.81 | 65.90 | 42.68 |
| CPU + NVMe | CPU → GPU | 26 | 2.85 | 60.40 | 47.10 |
| CPU + NVMe | GPU → CPU | 320 | 2.63 | 57.64 | 45.60 |
| CPU + CephFS | CPU → GPU | 14 | 3.18 | 66.06 | 48.06 |
| CPU + CephFS | GPU → CPU | 170 | 2.30 | 53.57 | 42.90 |

These values are the nearest trustworthy timing signal, but **CPU → GPU is not NVMe/CephFS retrieval time**. A secondary hit is promoted storage → CPU → GPU; the chart measures only the connector's aggregate direction and the service scrape is not pod-labelled. It cannot separate filesystem lookup, filesystem read, CPU placement, queue delay, or final PCIe transfer. The 60–66 ms CPU→GPU means are similar across tiers, so the final GPU copy is not the obvious discriminator.

For the next run, update the telemetry pack to scrape the exact 0.23 metric names and labels from both pods, or add spans around: secondary lookup, pending-promotion retry count, storage read, storage→CPU completion, CPU→GPU completion, and total request wait attributable to KV promotion.

## NVMe and CephFS performance

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":760,"height":340,"title":"Figure 10 \u2014 Secondary-tier throughput during profiling","data":{"values":[{"configuration":"CPU + NVMe","metric":"Read","mib_s":2056.7215277777777},{"configuration":"CPU + NVMe","metric":"Write","mib_s":924.2208370707948},{"configuration":"CPU + CephFS","metric":"Read","mib_s":1560.1706876770374},{"configuration":"CPU + CephFS","metric":"Write","mib_s":370.42534208241955}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"axis":{"labelAngle":-20},"title":"Configuration"},"xOffset":{"field":"metric","sort":["Read","Write"]},"y":{"field":"mib_s","type":"quantitative","title":"MiB/s"},"color":{"field":"metric","type":"nominal","sort":["Read","Write"]},"tooltip":[{"field":"configuration"},{"field":"metric"},{"field":"mib_s","format":".3f"}]}}
```

NVMe is asymmetric by node: `fx7c8` averages 674 MiB/s reads and 410 MiB/s writes, while `mt46x` averages 1,382 MiB/s reads and 514 MiB/s writes. That 2.05× read difference mirrors replica imbalance but neither device is continuously saturated. NVMe generation starts at 94.8 tok/s and ends at 3.4 tok/s.

CephFS pool `kvcache-fs-data0` averages 1,560 MiB/s reads and 370 MiB/s writes during profiling, with P95 read/write rates of 3,758/2,462 MiB/s. Read/write IOPS average 624/463. The PVC grows from 386.3 to 1,033.0 GiB during profiling. CephFS generation starts at 92.6 tok/s and ends at 8.3 tok/s. Pool bandwidth proves active storage traffic, not successful timely reuse.

CephFS logs contain nine `cannot store blocks` warnings across the two replicas; NVMe and CPU contain none. Nine warnings do not explain an 86.7% output-throughput loss by themselves, but they confirm secondary-tier backpressure in the worst cell.

## Host pressure and co-tenancy

| Configuration | Mean model-pod CPU | Mean model-pod working set | Memory-failure rate peak |
|---|---:|---:|---:|
| No offload | 358.1% | 12.4 GiB | 0 |
| CPU offload 256 GiB | 404.8% | 270.6 GiB | 0 |
| CPU + NVMe | 214.4% | 271.4 GiB | 0 |
| CPU + CephFS | 161.1% | 329.6 GiB | 0 |

The offload model pods hold roughly 271 GiB per replica for CPU/NVMe and 330 GiB per replica for CephFS. Across the three offload configurations, each node therefore carries about 871 GiB of model-pod working set in addition to the baseline and platform services. There are no memory failures and no collected CPU-throttling series, but shared memory bandwidth, NUMA locality, page-cache pressure, and Ceph client/network contention are unmeasured. Running all four configurations concurrently is a major confound and should not be repeated for the diagnosis matrix.

## Root-cause assessment

| Hypothesis | Assessment | Evidence |
|---|---|---|
| AIPerf concurrency is too low | **Rejected as primary cause** | Baseline scales 2.03×; tiered cells already have persistent waiting queues; AgentX branches add pressure beyond parent C16. |
| Replica-local ownership / approximate routing | **Leading hypothesis** | Approximate prefix producer, prefix weight 3 > queue weight 2, large per-replica KV/queue/generation skew, reduced CPU external reuse. |
| Async promotion / lookup retry amplification | **Leading mechanism** | Progressive throughput collapse, huge sampled prefix-query counters, active tiers without media saturation; matches the vLLM tiering retry design. |
| NVMe media saturation | **Unlikely** | Peak busy remains 51.5%/73.5%; throughput is substantial; final CPU→GPU copy time is comparable. |
| Ceph raw bandwidth limit | **Unlikely as sole cause** | 1.52 GiB/s mean reads and substantial IOPS/PVC growth, yet useful throughput collapses. |
| Ceph shared-root coordination | **Plausible, unproven** | Two independent engine IDs use the same `/mnt/kv_cache`; artifacts do not prove cross-engine namespace/index coordination. Test separate roots. |
| Host co-tenancy | **Material confound** | Four TP2 model replicas per node plus three large offload working sets; no memory failures, but bandwidth/NUMA pressure is unmeasured. |

## Next experiment: isolate ownership before tuning capacity

1. **Run one configuration at a time.** Reserve both nodes for that cell so CPU memory bandwidth and storage telemetry are attributable.
2. **Prove fixed ownership.** Repeat NVMe and CephFS with one client shard pinned to each replica or with session-sticky routing. Start at total C16 (C8 per replica). If throughput returns near 2× the single-replica result, routing/locality is the cause.
3. **Compare router modes.** Test queue-only, the current approximate prefix scorer, and precise KV-event routing. Keep all workload and cache parameters identical. Record per-request chosen pod and prefix score.
4. **Namespace Ceph per replica.** Use `/mnt/kv_cache/replica-a` and `/mnt/kv_cache/replica-b` unless the implementation explicitly documents safe shared metadata/index coordination. This separates pool performance from cross-engine namespace behavior.
5. **Only then sweep concurrency.** C8/C12/C16/C24 with stable ownership. Stop increasing once waiting rises or TTFT P95 inflects.
6. **Capture promotion telemetry.** Require per-pod and per-tier lookup count, pending retry count, promotion latency P50/P95/P99, bytes promoted, and miss/recompute outcome. Without it, storage bandwidth cannot explain request stalls.
7. **Repeat at three seeds.** The current single-seed failure is large enough to diagnose, but confirmation and backend ranking need repetitions.

Do **not** change TP2 or U0.92 in the first diagnostic rerun. Each replica already reports 36.72 GiB of GPU KV memory per GPU and 564,730 aggregate GPU KV tokens, and the tiered runs demonstrably retrieve external KV. Changing tensor parallelism or HBM allocation would alter cache pressure and compute parallelism before the multi-replica ownership problem is isolated.

## AIPerf command

The command is identical across configurations; BenchFlow substitutes the deployment-specific service URL and artifact directory.

```bash
aiperf profile --model 'google/gemma-4-31B-it' \
  --url '<deployment-service-url>' \
  --artifact-dir '<benchmark-artifact-dir>' \
  --ui None --benchmark-duration 1800 --concurrency 16 \
  --endpoint '/v1/chat/completions' --endpoint-type 'chat' \
  --hf-weka-repo 'semianalysisai/cc-traces-weka-with-subagents-060826' \
  --max-context-length 131072 --public-dataset 'weka_hf' --random-seed 42 \
  --scenario 'inferencex-agentx-mvp' --streaming \
  --tokenizer-trust-remote-code --use-server-token-count \
  --tokenizer 'google/gemma-4-31B-it'
```

## Native-cadence figures

- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/01 - Generation throughput|Generation throughput]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/02 - Prompt-token sources - No offload|Prompt-token sources — no offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/03 - Prompt-token sources - CPU offload|Prompt-token sources — CPU offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/04 - Prompt-token sources - NVMe|Prompt-token sources — NVMe]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/05 - Prompt-token sources - CephFS|Prompt-token sources — CephFS]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/06 - Per-replica generation - No offload|Per-replica generation — no offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/07 - Per-replica generation - CPU offload|Per-replica generation — CPU offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/08 - Per-replica generation - NVMe|Per-replica generation — NVMe]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/09 - Per-replica generation - CephFS|Per-replica generation — CephFS]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/10 - Running requests - No offload|Running requests — no offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/11 - Waiting requests - No offload|Waiting requests — no offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/12 - Running requests - CPU offload|Running requests — CPU offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/13 - Waiting requests - CPU offload|Waiting requests — CPU offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/14 - Running requests - NVMe|Running requests — NVMe]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/15 - Waiting requests - NVMe|Waiting requests — NVMe]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/16 - Running requests - CephFS|Running requests — CephFS]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/17 - Waiting requests - CephFS|Waiting requests — CephFS]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/18 - HBM KV-cache - No offload|HBM KV-cache — no offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/19 - HBM KV-cache - CPU offload|HBM KV-cache — CPU offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/20 - HBM KV-cache - NVMe|HBM KV-cache — NVMe]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/21 - HBM KV-cache - CephFS|HBM KV-cache — CephFS]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/22 - External-prefix hit rate - CPU offload|External-prefix hit rate — CPU offload]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/23 - External-prefix hit rate - NVMe|External-prefix hit rate — NVMe]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/24 - External-prefix hit rate - CephFS|External-prefix hit rate — CephFS]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/25 - NVMe throughput - fx7c8|NVMe throughput — fx7c8]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/26 - NVMe busy time - fx7c8|NVMe busy time — fx7c8]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/27 - NVMe throughput - mt46x|NVMe throughput — mt46x]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/28 - NVMe busy time - mt46x|NVMe busy time — mt46x]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/29 - CephFS throughput|CephFS throughput]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/30 - CephFS IOPS|CephFS IOPS]]
- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison/31 - CephFS capacity|CephFS capacity]]

## Validity and provenance

- All cells match on model, image, vLLM version, TP2, two replicas, U0.92, BF16, context, concurrency, seed, workload, duration, prefix caching, and shared memory.
- Model and router pods report zero restarts. AIPerf reports zero request errors; model logs report zero tracebacks, CUDA OOMs, and unavailable-block errors. All cells hit the grace-period cutoff.
- CephFS records nine store refusals; other cells record none.
- GPU utilization queries returned no series, and CPU-throttling queries returned no series. These are explicit observability gaps.
- Prometheus performance figures use the exact profiling send window and native 15-second samples. Storage appendices use the full captured window; the exported throughput and IOPS series are five-minute rate calculations.
- The August 3 single-replica comparison is used only for directional scaling context, not as a controlled paired baseline.
- Artifacts were downloaded and analyzed on 2026-08-05.

## Run registry

| Configuration | Deployment | Replicas / TP | MLflow | Disposition |
|---|---|---:|---|---|
| No offload | `cpu-offloading-m1-be633f3f64` | 2 / 2 | [e27348dff257466db48a3eee79f8182a](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/e27348dff257466db48a3eee79f8182a?workspace=benchflow) | Accepted failure characterization |
| CPU offload 256 GiB | `cpu-offloading-m2-be633f3f64` | 2 / 2 | [0ff6322040384fe3b98ec8c8725d08ce](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/0ff6322040384fe3b98ec8c8725d08ce?workspace=benchflow) | Accepted failure characterization |
| CPU + NVMe | `cpu-offloading-m3-be633f3f64` | 2 / 2 | [a293c9c7fefe4028a5d438bc3b5308ef](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/a293c9c7fefe4028a5d438bc3b5308ef?workspace=benchflow) | Accepted failure characterization |
| CPU + CephFS | `cpu-offloading-m4-be633f3f64` | 2 / 2 | [a4176ee0e2db4309aee724ec9a86b168](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/318/runs/a4176ee0e2db4309aee724ec9a86b168?workspace=benchflow) | Accepted failure characterization |