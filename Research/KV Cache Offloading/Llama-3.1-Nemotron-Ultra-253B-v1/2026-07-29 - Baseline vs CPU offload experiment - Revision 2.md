---
title: "2026-07-29 - Baseline vs CPU offload experiment - Revision 2"
date: "2026-07-29"
type: "experiment-report"
topic: "KV Cache Offloading"
experiment: "MLflow experiment 325"
report_revision: 2
model: "RedHatAI/Llama-3_1-Nemotron-Ultra-253B-v1"
model_revision: "unknown"
runtime_image: "vllm/vllm-openai:v0.23.0"
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
cpu_bytes: "0 for baseline; 68719476736 for CPU offload"
offload_spec: "none for baseline; TieringOffloadingSpec for CPU offload"
secondary_tier: "not applicable"
secondary_tier_threads: "not applicable"
shared_memory: "200Gi"
workload: "AIPerf AgentX inferencex-agentx-mvp"
random_seed: 42
duration_seconds: 1800
cache_cleaning_state: "new model pods; CPU mmap newly created; explicit workload-cache purge not recorded"
baseline: "No offload"
configuration_count: 2
---

# 2026-07-29 - Baseline vs CPU offload experiment - Revision 2

## Benchmark overview

This comparison tests `RedHatAI/Llama-3_1-Nemotron-Ultra-253B-v1` with No offload and a 64 GiB CPU KV tier. Both Revision 2 runs use one TP8 replica on eight H100 GPUs, vLLM `v0.23.0`, `gpu_memory_utilization=0.9`, a 131,072-token context limit, 200 GiB shared memory, prefix caching, seed 42, streaming AgentX, concurrency 16, and a 1,800-second profiling phase. Only `TieringOffloadingSpec` with `cpu_bytes_to_use=68719476736` varies.

Revision 2 follows [[2026-07-29 - Baseline vs CPU offload experiment|Revision 1 / the initial report]]. It is a lower-pressure follow-up, not a like-for-like repetition: concurrency changed from 32 to 16.

## Executive summary

**Revision 2 again shows no end-user benefit from the 64 GiB CPU tier.** CPU offload was **1.04% lower** in request throughput, **1.57% lower** in output-token throughput, **9.50% worse** in mean TTFT, and **7.55% worse** in mean end-to-end latency. Mean ITL moved in the opposite direction and was **14.67% better** with CPU offload. With one run per cell and mixed metric directions, there is no defensible winner and no statistical parity claim.

What differs from Revision 1 is the pressure level and the sign of several small effects. Revision 1 used concurrency 32; Revision 2 uses 16. Mean TTFT fell from 507.9 to 228.3 seconds for baseline and from 496.1 to 250.0 seconds for CPU offload. Mean waiting roughly halved, and completed root sessions increased from 2 to 3 in both configurations. Yet completed-request throughput stayed around 0.066 requests/s. Revision 1 had a nominal CPU lead of 0.49% in request throughput and 2.32% in mean TTFT; Revision 2 has nominal CPU regressions of 1.04% and 9.50%. The effect reversals reinforce that the original small deltas were not stable configuration effects.

The mechanism also reproduces Revision 1. The CPU tier was initialized, occupied about 65.3 GiB of extra model-container memory, and accepted **3,743.3 GiB** of GPU-to-CPU writes, but only **1.562 GiB** returned—a **2,396:1** store-to-load ratio. External-KV prompt attribution remained exactly **0%**, while **98.82%** of sampled prompt-token rate was local computation. Lowering concurrency did not turn the CPU tier into a materially reusable cache.

Both runs had zero request errors, model restarts, OOMs, tracebacks, or KV store-refusal warnings. Both nevertheless hit the fixed-duration and grace-period cutoffs, leaving 15 of 18 root sessions unfinished. The report therefore treats request and mechanism evidence as useful but session-level ranking as censored.

## Validity verdict

**Verdict: Conditionally valid.** The two Revision 2 cells are fingerprint-matched except for the intended CPU tier and ran concurrently, but they used different H100 nodes and have one repetition each. Both reached the grace-period cutoff with most root sessions unfinished. They are accepted for request-level and mechanism diagnosis; small deltas and completed-session performance remain inconclusive. Revision 1 and Revision 2 cannot be pooled as repetitions because concurrency changed from 32 to 16.

## Main conclusions

- **Winner:** No defensible winner. Baseline leads request throughput, output-token throughput, TTFT, and E2E; CPU offload leads ITL. All comparisons are single samples.
- **Loser:** CPU offload does not achieve the experiment objective: it does not improve throughput, completed sessions, or prompt reuse.
- **Parity:** Both completed 3 of 18 root sessions and 26 of 29 branches. This is observed equality in one pair, not statistical parity.
- **Issue:** Fifteen root sessions per configuration remained incomplete at the fixed-duration cutoff; separate model nodes remain a confounder.
- **Mechanism:** The CPU tier stores thousands of GiB but loads only 1.562 GiB, and external-KV prompt attribution remains zero.
- **Revision result:** Halving concurrency improves latency and queueing for both configurations but does not reveal a CPU-offload benefit.
- **Decision:** Do not claim a performance benefit for 64 GiB CPU offload on this workload. Validate exact-prefix restoration before further AgentX scaling.

## Configuration map

| Order | Configuration | What changes | Color |
|---:|---|---|---|
| 1 | No offload | GPU KV cache only | `#1f77b4` |
| 2 | CPU offload 64 GiB | `TieringOffloadingSpec`, 68,719,476,736 CPU bytes | `#ff7f0e` |

## Headline results

| Configuration | MLflow | Disposition | Requests/s | Δ vs baseline | Output tokens/s | Mean TTFT | Mean E2E | Mean ITL | Completed sessions | Error/cancel rate |
|---|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| No offload | [run `5e6ea113`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/5e6ea11348e5401086cfe71599306b37?workspace=benchflow) | Conditionally accepted | 0.06619 | baseline | 50.27 | 228.3 s | 266.4 s | 59.8 ms | 3 / 18 | 0 errors; 11.68% cancelled |
| CPU offload 64 GiB | [run `43521e84`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/43521e84d39b41cc8fe59e52c6b95598?workspace=benchflow) | Conditionally accepted | 0.06551 | -1.04% | 49.48 | 250.0 s | 286.5 s | 51.0 ms | 3 / 18 | 0 errors; 11.19% cancelled |

| Additional metric | No offload | CPU offload 64 GiB | CPU delta |
|---|---:|---:|---:|
| Completed requests | 121 | 119 | -1.65% |
| Total output tokens | 91,900 | 89,890 | -2.19% |
| Mean output tokens/request | 759.5 | 755.4 | -0.54% |
| Mean input tokens/request | 54,235 | 54,008 | -0.42% |
| TTFT P95 | 321.8 s | 348.9 s | +8.42% |
| E2E P95 | 372.9 s | 406.4 s | +8.97% |
| ITL P95 | 115.7 ms | 101.1 ms | -12.61% |

## Experiment and deployment fingerprint

| Dimension | Stable value or per-configuration difference |
|---|---|
| Model | `RedHatAI/Llama-3_1-Nemotron-Ultra-253B-v1`; revision not recorded |
| Runtime | `vllm/vllm-openai:v0.23.0`; RHOAI 3.5 EA2 |
| Topology | One replica, TP8, PP1, eight H100 GPUs |
| Cache settings | GPU cache 214,336 tokens; `gpu_memory_utilization=0.9`; prefix caching enabled |
| CPU tier | Baseline none; CPU run 64 GiB, 16,384 blocks, no secondary tier |
| Context and scheduling | `max_model_len=131072`; `max_num_seqs` not explicitly set |
| Workload | AgentX `inferencex-agentx-mvp`, concurrency 16, seed 42, duration 1,800 s, streaming |
| Dataset | `semianalysisai/cc-traces-weka-with-subagents-060826` |
| Warmup | 17 requests in each cell before profiling |
| Shared memory | 200 GiB; CPU mmap newly created at model startup |
| Placement | Baseline on `diadochos-hqxzk-gpu-h100-6kl5z`; CPU on `diadochos-hqxzk-gpu-h100-mt46x` |
| Restarts | Zero model and router restarts in both captured pod descriptions |

## Comparison evidence

Figure 1 compares completed-request throughput from `results/profile_export_aiperf.json`. Values cover each run's approximately 30-minute profiling result.

```vega-lite
{"width":720,"height":300,"title":"Figure 1 \u2014 Completed-request throughput","data":{"values":[{"configuration":"No offload","requests_s":0.06619194632166268},{"configuration":"CPU offload 64 GiB","requests_s":0.06550533822973867}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 64 GiB"]},"y":{"field":"requests_s","type":"quantitative","title":"Request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"requests_s","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

The 1.04% CPU deficit is smaller than the uncertainty implied by one run per cell. It does not establish a baseline advantage.

Figure 2 compares output-token throughput from the same AIPerf exports.

```vega-lite
{"width":720,"height":300,"title":"Figure 2 \u2014 Output-token throughput","data":{"values":[{"configuration":"No offload","output_tokens_s":50.27305675174215},{"configuration":"CPU offload 64 GiB","output_tokens_s":49.48130128967402}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 64 GiB"]},"y":{"field":"output_tokens_s","type":"quantitative","title":"Output-token throughput (tokens/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"output_tokens_s","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU offload is 1.57% lower. The mean output length differs by only 0.54%, so output-shape drift is small but cannot be completely removed from a fixed-duration trace replay.

Figure 3 compares mean and P95 TTFT and end-to-end request latency from native completed-request samples.

```vega-lite
{"width":760,"height":320,"title":"Figure 3 \u2014 Request latency","data":{"values":[{"configuration":"No offload","metric":"TTFT mean","seconds":228.28792417577682},{"configuration":"No offload","metric":"TTFT P95","seconds":321.786914992},{"configuration":"No offload","metric":"E2E mean","seconds":266.3814219308595},{"configuration":"No offload","metric":"E2E P95","seconds":372.939898392},{"configuration":"CPU offload 64 GiB","metric":"TTFT mean","seconds":249.96758578315965},{"configuration":"CPU offload 64 GiB","metric":"TTFT P95","seconds":348.89249515599994},{"configuration":"CPU offload 64 GiB","metric":"E2E mean","seconds":286.5045160955126},{"configuration":"CPU offload 64 GiB","metric":"E2E P95","seconds":406.39487997769993}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["TTFT mean","TTFT P95","E2E mean","E2E P95"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB"]},"y":{"field":"seconds","type":"quantitative","title":"Latency (seconds)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"seconds","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU offload is worse for every plotted request-latency statistic: mean TTFT +9.50%, P95 TTFT +8.42%, mean E2E +7.55%, and P95 E2E +8.97%. One paired sample does not establish causality, but there is no latency benefit to preserve here.

Figure 4 compares ITL from the completed-request samples.

```vega-lite
{"width":760,"height":320,"title":"Figure 4 \u2014 Inter-token latency","data":{"values":[{"configuration":"No offload","metric":"Mean","milliseconds":59.764674268385804},{"configuration":"No offload","metric":"P50","milliseconds":47.783617940133034},{"configuration":"No offload","metric":"P95","milliseconds":115.69527901886792},{"configuration":"CPU offload 64 GiB","metric":"Mean","milliseconds":50.994061800721354},{"configuration":"CPU offload 64 GiB","metric":"P50","milliseconds":43.89015036572622},{"configuration":"CPU offload 64 GiB","metric":"P95","milliseconds":101.10995814742004}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Mean","P50","P95"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB"]},"y":{"field":"milliseconds","type":"quantitative","title":"Inter-token latency (milliseconds)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"milliseconds","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

ITL moves the other way: CPU is 14.67% lower at the mean and 12.61% lower at P95. This mixed result argues against compressing the experiment into a single winner label.

## Request-, session-, and branch-level results

Figure 5 combines final AIPerf request, root-session, and branch counts.

```vega-lite
{"width":760,"height":320,"title":"Figure 5 \u2014 Request, session, and branch outcomes","data":{"values":[{"configuration":"No offload","outcome":"Requests completed","count":121},{"configuration":"No offload","outcome":"Requests cancelled","count":16},{"configuration":"No offload","outcome":"Sessions completed","count":3},{"configuration":"No offload","outcome":"Sessions not completed","count":15},{"configuration":"No offload","outcome":"Branches completed","count":26},{"configuration":"CPU offload 64 GiB","outcome":"Requests completed","count":119},{"configuration":"CPU offload 64 GiB","outcome":"Requests cancelled","count":15},{"configuration":"CPU offload 64 GiB","outcome":"Sessions completed","count":3},{"configuration":"CPU offload 64 GiB","outcome":"Sessions not completed","count":15},{"configuration":"CPU offload 64 GiB","outcome":"Branches completed","count":26}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"outcome","type":"nominal","title":"Metric","sort":["Requests completed","Requests cancelled","Sessions completed","Sessions not completed","Branches completed"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB"]},"y":{"field":"count","type":"quantitative","title":"Count","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"outcome","type":"nominal"},{"field":"count","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Both configurations complete 3 of 18 root sessions and 26 of 29 branches. Baseline completes two more individual requests. The 15 unfinished sessions per cell are fixed-duration censoring evidence, not request errors, but they prevent a reliable completed-session latency comparison.

The native request-level distributions are retained in [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/10 - TTFT distribution|the TTFT distribution]], [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/11 - End-to-end latency distribution|the end-to-end latency distribution]], and [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/12 - Output-token samples|output-token samples]].

## Why CPU offload still does not help

Figure 6 integrates all native 15-second `prompt_tokens_rate_by_source` samples by source, then normalizes each configuration to 100%.

```vega-lite
{"width":720,"height":300,"title":"Figure 6 \u2014 Prompt-token processing share by source","data":{"values":[{"configuration":"No offload","source":"External KV transfer","share_percent":0.0},{"configuration":"No offload","source":"Local HBM hit","share_percent":0.028439629850111593},{"configuration":"No offload","source":"Local compute","share_percent":99.97156037014989},{"configuration":"CPU offload 64 GiB","source":"External KV transfer","share_percent":0.0},{"configuration":"CPU offload 64 GiB","source":"Local HBM hit","share_percent":1.176726226982799},{"configuration":"CPU offload 64 GiB","source":"Local compute","share_percent":98.82327377301719}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 64 GiB"]},"y":{"field":"share_percent","type":"quantitative","title":"Integrated prompt-token rate share (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"Prompt source","scale":{"domain":["External KV transfer","Local HBM hit","Local compute"],"range":["#9467bd","#2ca02c","#d62728"]}},"order":{"field":"source","sort":["External KV transfer","Local HBM hit","Local compute"]},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"source","type":"nominal"},{"field":"share_percent","type":"quantitative","format":".4f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

External-KV transfer contributes 0.000% in both cells. CPU offload raises local HBM-hit share from 0.028% to 1.177%, but 98.823% remains local computation. The exact prompt-source time series are in [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/01 - Prompt-token sources - Baseline|Baseline prompt sources]] and [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/02 - Prompt-token sources - CPU offload|CPU prompt sources]].

Figure 7 summarizes native 15-second scheduler series.

```vega-lite
{"width":760,"height":320,"title":"Figure 7 \u2014 Scheduler pressure","data":{"values":[{"configuration":"No offload","metric":"Mean running","requests":2.9526627218934913},{"configuration":"No offload","metric":"Peak running","requests":17.0},{"configuration":"No offload","metric":"Mean waiting","requests":13.366863905325443},{"configuration":"No offload","metric":"Peak waiting","requests":36.0},{"configuration":"CPU offload 64 GiB","metric":"Mean running","requests":2.75},{"configuration":"CPU offload 64 GiB","metric":"Peak running","requests":11.0},{"configuration":"CPU offload 64 GiB","metric":"Mean waiting","requests":14.05952380952381},{"configuration":"CPU offload 64 GiB","metric":"Peak waiting","requests":36.0}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Mean running","Peak running","Mean waiting","Peak waiting"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB"]},"y":{"field":"requests","type":"quantitative","title":"Requests (count)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"requests","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU offload has slightly fewer running requests and 0.69 more waiting requests on average. All waiting is attributed to `capacity`; `deferred` remains zero. Lower concurrency reduces pressure relative to Revision 1, but CPU offload does not reduce it relative to the Revision 2 baseline. Native views: [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/04 - Running requests|running requests]] and [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/05 - Waiting requests|waiting requests]].

Figure 8 summarizes native 15-second GPU KV-cache occupancy.

```vega-lite
{"width":760,"height":320,"title":"Figure 8 \u2014 GPU KV-cache utilization","data":{"values":[{"configuration":"No offload","metric":"Mean","percent":70.39613385724162},{"configuration":"No offload","metric":"Peak","percent":99.79096677864875},{"configuration":"CPU offload 64 GiB","metric":"Mean","percent":68.05177838212552},{"configuration":"CPU offload 64 GiB","metric":"Peak","percent":99.94027622247107}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Mean","Peak"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB"]},"y":{"field":"percent","type":"quantitative","title":"GPU KV-cache utilization (%)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"percent","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Both cells repeatedly approach 100% occupancy. CPU offload lowers mean occupancy from 70.40% to 68.05%, but that does not translate into higher throughput or more completed sessions. See [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/03 - GPU KV-cache utilization|the full utilization time series]].

Figure 9 shows the final cumulative CPU-tier transfer counters. A point plot with a logarithmic axis is used because the directions differ by more than three orders of magnitude.

```vega-lite
{"width":700,"height":300,"title":"Figure 9 \u2014 CPU-offload cumulative transfer volume","data":{"values":[{"direction":"GPU \u2192 CPU stores","gib":3743.28125},{"direction":"CPU \u2192 GPU loads","gib":1.5625}]},"mark":{"type":"point","filled":true,"size":170,"tooltip":true},"encoding":{"x":{"field":"direction","type":"nominal","title":"Transfer direction","sort":["GPU \u2192 CPU stores","CPU \u2192 GPU loads"]},"y":{"field":"gib","type":"quantitative","title":"Cumulative transfer volume (GiB, logarithmic)","scale":{"type":"log"}},"color":{"field":"direction","type":"nominal","legend":null,"scale":{"domain":["GPU \u2192 CPU stores","CPU \u2192 GPU loads"],"range":["#ff7f0e","#9467bd"]}},"tooltip":[{"field":"direction","type":"nominal"},{"field":"gib","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

The tier stores 3,743.3 GiB and loads 1.562 GiB, a 2,396:1 ratio. Revision 2 captures a tiny nonzero CPU-to-GPU rate, unlike Revision 1's zero sampled rate, but the cumulative load volume is even smaller than Revision 1's 3.523 GiB. This is still an eviction-sink signature. See [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/06 - CPU offload transfers|the transfer time series]].

The CPU model-container working set averages 571.5 GiB versus 506.3 GiB at baseline, confirming that the 64 GiB allocation was real. See [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/07 - Model-container memory|model memory]].

## What Revision 2 changes from Revision 1

| Measure | Revision 1 · C32 baseline | Revision 1 · C32 CPU | Revision 2 · C16 baseline | Revision 2 · C16 CPU |
|---|---:|---:|---:|---:|
| Requests/s | 0.06737 | 0.06770 | 0.06619 | 0.06551 |
| Mean TTFT | 507.9 s | 496.1 s | 228.3 s | 250.0 s |
| Mean E2E | 543.4 s | 539.2 s | 266.4 s | 286.5 s |
| Mean ITL | 50.7 ms | 60.6 ms | 59.8 ms | 51.0 ms |
| Mean waiting | 26.69 | 26.61 | 13.37 | 14.06 |
| Completed sessions | 2 / 32 | 2 / 32 | 3 / 18 | 3 / 18 |
| CPU stores | N/A | 4,514.3 GiB | N/A | 3,743.3 GiB |
| CPU loads | N/A | 3.523 GiB | N/A | 1.562 GiB |
| External-KV prompt share | 0% | 0% | 0% | 0% |

Figure 10 plots the CPU-versus-baseline effect inside each revision. Positive latency deltas are regressions; positive throughput deltas are improvements.

```vega-lite
{"width":760,"height":330,"title":"Figure 10 \u2014 CPU-offload effect changed sign between revisions","data":{"values":[{"revision":"Revision 1 \u00b7 C32","metric":"Request throughput","cpu_delta_percent":0.4876145492047623},{"revision":"Revision 1 \u00b7 C32","metric":"Mean TTFT","cpu_delta_percent":-2.318804560131038},{"revision":"Revision 1 \u00b7 C32","metric":"Mean E2E","cpu_delta_percent":-0.7714371045509427},{"revision":"Revision 1 \u00b7 C32","metric":"Mean ITL","cpu_delta_percent":19.61721692304641},{"revision":"Revision 2 \u00b7 C16","metric":"Request throughput","cpu_delta_percent":-1.0372985386884026},{"revision":"Revision 2 \u00b7 C16","metric":"Mean TTFT","cpu_delta_percent":9.496630925904759},{"revision":"Revision 2 \u00b7 C16","metric":"Mean E2E","cpu_delta_percent":7.554240839616866},{"revision":"Revision 2 \u00b7 C16","metric":"Mean ITL","cpu_delta_percent":-14.675245159503714}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Request throughput","Mean TTFT","Mean E2E","Mean ITL"]},"xOffset":{"field":"revision","sort":["Revision 1 \u00b7 C32","Revision 2 \u00b7 C16"]},"y":{"field":"cpu_delta_percent","type":"quantitative","title":"CPU-offload delta versus same-revision baseline (%)","scale":{"zero":true}},"color":{"field":"revision","type":"nominal","title":"Revision","scale":{"domain":["Revision 1 \u00b7 C32","Revision 2 \u00b7 C16"],"range":["#7f7f7f","#17becf"]}},"tooltip":[{"field":"revision","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"cpu_delta_percent","type":"quantitative","format":"+.2f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Request throughput, TTFT, and ITL effects reverse sign between revisions. This is evidence that the small Revision 1 deltas were sensitive to workload pressure, node/run variation, or both. The durable observation is the one that reproduces: the CPU tier is allocated and heavily written, yet it produces negligible restoration and no external-KV prompt attribution.

## Additional time-series evidence

- [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/08 - Generation throughput|Generation throughput at native 15-second cadence]]
- [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/09 - TTFT P90|Server-side TTFT P90 at native 15-second cadence]]

## Saturation, errors, and validity evidence

- Both runs report zero AIPerf request errors.
- Both model pods and both router pods have restart count zero.
- Searches of model logs found no traceback, CUDA OOM, store-refusal, or unavailable-block storm.
- Both AIPerf runs reached the 1,800-second sending timeout and 30-second grace timeout.
- Baseline cancelled 16 of 137 sent requests; CPU cancelled 15 of 134.
- Both had one abandoned pending branch join during cleanup. This follows the fixed-duration cutoff and does not indicate a model-serving error.
- Capacity is the only nonzero waiting reason. Deferred waiting remains zero.
- The runs are on separate H100 nodes, and one repetition cannot separate CPU-tier effect from run-to-run variation.

## Conclusions

### Established by the data

- At concurrency 16, CPU offload does not improve request throughput, output-token throughput, TTFT, E2E latency, queueing, or completed sessions.
- The CPU tier is definitely enabled: it creates a 68.72 GB mmap, initializes 16,384 blocks, and raises working-set memory by about 65.3 GiB.
- The tier writes 3,743.3 GiB but loads only 1.562 GiB. External-KV prompt attribution remains zero.
- Halving concurrency materially reduces latency and waiting versus Revision 1 for both configurations, but it does not change the CPU-tier conclusion.

### Uncertainty and limitations

- There is one run per configuration and different node placement.
- Revision 1 and Revision 2 use different concurrency and are not statistical repetitions.
- Most root sessions are censored by the fixed-duration cutoff.
- Aggregate counters cannot distinguish capacity eviction, key/lookup mismatch, restoration rejection, or a prompt-attribution defect.
- CPU memory bandwidth, NUMA placement, per-block hit/miss/eviction, and lookup-failure telemetry are unavailable.

### Decision

Do not recommend 64 GiB CPU offload for this AgentX workload based on either revision. Treat the next step as a functional restoration test, not another broad performance run.

## Next experiment

1. Run an exact-prefix two-request smoke test and require nonzero CPU-to-GPU bytes plus nonzero externally sourced prompt tokens.
2. Capture block-level store, load-hit, load-miss, eviction, lookup-failure, and restoration-rejection counters.
3. If the smoke test passes, repeat baseline and CPU offload at C8 and C16 with at least three paired, randomized repetitions on the same node.
4. Sweep CPU capacity through 64, 128, and 256 GiB only after the restoration path is verified.
5. Extend duration or reduce workload breadth enough to obtain an uncensored session-latency distribution.
6. Reject a run on request errors, model restart/OOM, configuration drift, or a failed restoration smoke test.

## Run registry and provenance

| Configuration | Child/deployment | MLflow run | Artifact sources | Disposition |
|---|---|---|---|---|
| No offload | `cpu-offloading-c0c856` / `cpu-kv-offload-distributed-default` | [5e6ea11348e5401086cfe71599306b37](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/5e6ea11348e5401086cfe71599306b37?workspace=benchflow) | AIPerf, model/load-generator logs, manifests, Prometheus | Conditionally accepted |
| CPU offload 64 GiB | `cpu-offloading-739e0a` / `rhoai-cpu-kv-offload-64g` | [43521e84d39b41cc8fe59e52c6b95598](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/43521e84d39b41cc8fe59e52c6b95598?workspace=benchflow) | AIPerf, model/load-generator logs, manifests, Prometheus | Conditionally accepted |

Artifacts were downloaded on 2026-07-29. Tables use `results/profile_export_aiperf.json`, `benchmark/profile_export.jsonl`, final AIPerf phase logs, rendered manifests, model logs, and Prometheus JSON. Prometheus companion charts retain the native 15-second samples; no smoothing or downsampling was applied. Prompt-source shares integrate native rate samples and normalize by the sum across sources. The available Prometheus window includes startup, warmup, profiling, grace, and capture rather than only the 1,800-second profiling phase.