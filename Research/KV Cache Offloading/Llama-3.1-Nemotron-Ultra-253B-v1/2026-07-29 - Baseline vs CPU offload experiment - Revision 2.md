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
cpu_bytes: "0 for baseline; 68719476736 for CPU offload 64 GiB; 171798691840 for CPU offload 160 GiB"
offload_spec: "none for baseline; TieringOffloadingSpec for CPU offload"
secondary_tier: "not applicable"
secondary_tier_threads: "not applicable"
shared_memory: "200Gi"
workload: "AIPerf AgentX inferencex-agentx-mvp"
random_seed: 42
duration_seconds: 1800
cache_cleaning_state: "new model pods; CPU mmap newly created; explicit workload-cache purge not recorded"
baseline: "No offload"
configuration_count: 3
---

# 2026-07-29 - Baseline vs CPU offload experiment - Revision 2

## Benchmark overview

This comparison tests `RedHatAI/Llama-3_1-Nemotron-Ultra-253B-v1` with No offload, a 64 GiB CPU KV tier, and a 160 GiB CPU KV tier. All Revision 2 runs use one TP8 replica on eight H100 GPUs, vLLM `v0.23.0`, `gpu_memory_utilization=0.9`, a 131,072-token context limit, prefix caching, seed 42, streaming AgentX, concurrency 16, and a 1,800-second profiling phase. The CPU capacity is the intended mechanism variable: `cpu_bytes_to_use=68719476736` or `171798691840`.

Revision 2 follows [[2026-07-29 - Baseline vs CPU offload experiment|Revision 1 / the initial report]]. It is a lower-pressure follow-up, not a like-for-like repetition: concurrency changed from 32 to 16.

## Executive summary

**The new 160 GiB run proves that CPU capacity changes restoration behavior, but 160 GiB is still not enough to produce an end-user performance benefit.** Relative to No offload, CPU160 is **0.10% lower** in request throughput, **0.81% lower** in output-token throughput, **5.43% worse** in mean TTFT, **4.14% worse** in mean end-to-end latency, and **11.70% worse** in mean ITL. It completes the same 3 of 18 root sessions. P95 TTFT improves 4.74% and P95 ITL improves 5.42%, but one unpaired run cannot establish a tail-latency benefit.

The important result is mechanistic. CPU→GPU restoration rises from **1.562 GiB at 64 GiB to 77.586 GiB at 160 GiB**, a **49.7×** increase. External-KV prompt attribution rises from **0% to 2.06%**, and the store-to-load ratio improves from **2,396:1 to 47.8:1**. This demonstrates that the connector can restore AgentX prefixes and that the 64 GiB result was substantially capacity-limited.

Nevertheless, CPU160 still sends **3,711.1 GiB** down and retrieves only **77.6 GiB**; **96.76%** of integrated prompt processing remains local computation. Its retention-clock estimate is only **107.2 seconds** at the measured 1.493 GiB/s write rate, versus **240.7 seconds mean TTFT before client think time is added**. The tier crosses the first reuse threshold but remains below the workload's typical reuse gap.

All three runs have zero request errors, model restarts, OOMs, tracebacks, preemptions, or KV store-refusal warnings. All hit the fixed-duration and grace-period cutoffs, leaving 15 of 18 root sessions unfinished. The report therefore accepts the mechanism evidence conditionally but treats small request deltas and session-level ranking as inconclusive.

## Validity verdict

**Verdict: Conditionally valid.** The three Revision 2 cells are fingerprint-matched except for the intended CPU tier. Baseline and CPU64 ran concurrently on different H100 nodes; CPU160 ran later on the same node used by baseline. Each cell has one repetition, and all reached the grace-period cutoff with most root sessions unfinished. The runs are accepted for mechanism diagnosis; small performance deltas and completed-session comparisons remain inconclusive.

## Main conclusions

- **Winner:** No defensible performance winner; all configurations remain near 0.066 requests/s and complete 3 of 18 root sessions.
- **Measured mechanism:** CPU160 restores 77.6 GiB and sources 2.06% of prompt processing externally. CPU64 restores only 1.562 GiB and sources 0%.
- **Capacity conclusion:** 64 GiB is demonstrably too small. 160 GiB crosses into real reuse but remains insufficient: the store/load ratio is still 47.8:1 and local computation remains 96.76%.
- **Retention conclusion:** Estimated CPU retention grows from 42.5 seconds at 64 GiB to 107.2 seconds at 160 GiB, still well below the 240.7-second mean TTFT lower bound on the next-turn reuse gap.
- **GPU-utilization decision:** Keep `gpu_memory_utilization=0.9` for the next CPU-capacity cell. Do not lower it; offloading is already forced, and a smaller GPU cache would increase churn and queueing.
- **Next capacity:** Test 512 GiB for mean-gap coverage; use 640 GiB if the objective is closer to P95-gap coverage, subject to node-memory and `/dev/shm` capacity.

## Configuration map

| Order | Configuration | What changes | Color |
|---:|---|---|---|
| 1 | No offload | GPU KV cache only | `#1f77b4` |
| 2 | CPU offload 64 GiB | `TieringOffloadingSpec`, 68,719,476,736 CPU bytes | `#ff7f0e` |
| 3 | CPU offload 160 GiB | `TieringOffloadingSpec`, 171,798,691,840 CPU bytes | `#2ca02c` |

## Headline results

| Configuration | MLflow | Disposition | Requests/s | Δ vs baseline | Output tokens/s | Mean TTFT | Mean E2E | Mean ITL | Completed sessions | Error/cancel rate |
|---|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| No offload | [run `5e6ea113`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/5e6ea11348e5401086cfe71599306b37?workspace=benchflow) | Conditionally accepted | 0.06619 | baseline | 50.27 | 228.3 s | 266.4 s | 59.8 ms | 3 / 18 | 0 errors; 11.68% cancelled |
| CPU offload 64 GiB | [run `43521e84`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/43521e84d39b41cc8fe59e52c6b95598?workspace=benchflow) | Conditionally accepted | 0.06551 | -1.04% | 49.48 | 250.0 s | 286.5 s | 51.0 ms | 3 / 18 | 0 errors; 11.19% cancelled |
| CPU offload 160 GiB | [run `45f523ae`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/45f523ae6ebf45cf984282befaac0eeb?workspace=benchflow) | Conditionally accepted | 0.06612 | -0.10% | 49.87 | 240.7 s | 277.4 s | 66.8 ms | 3 / 18 | 0 errors; 11.11% cancelled |

| Additional metric | No offload | CPU64 | CPU64 Δ | CPU160 | CPU160 Δ |
|---|---:|---:|---:|---:|---:|
| Completed requests | 121 | 119 | -1.65% | 120 | -0.83% |
| Total output tokens | 91,900 | 89,890 | -2.19% | 90,498 | -1.53% |
| Mean output tokens/request | 759.5 | 755.4 | -0.54% | 754.1 | -0.70% |
| Mean input tokens/request | 54,235 | 54,008 | -0.42% | 54,278 | +0.08% |
| TTFT P95 | 321.8 s | 348.9 s | +8.42% | 306.5 s | -4.74% |
| E2E P95 | 372.9 s | 406.4 s | +8.97% | 379.9 s | +1.86% |
| ITL P95 | 115.7 ms | 101.1 ms | -12.61% | 109.4 ms | -5.42% |

## Experiment and deployment fingerprint

| Dimension | Stable value or per-configuration difference |
|---|---|
| Model | `RedHatAI/Llama-3_1-Nemotron-Ultra-253B-v1`; revision not recorded |
| Runtime | `vllm/vllm-openai:v0.23.0`; RHOAI 3.5 EA2 |
| Topology | One replica, TP8, PP1, eight H100 GPUs |
| Cache settings | GPU cache 214,336 tokens; `gpu_memory_utilization=0.9`; prefix caching enabled |
| CPU tier | Baseline none; CPU64 has 16,384 blocks / 262,144 tokens; CPU160 has 40,960 blocks / 655,360 tokens; no secondary tier |
| Context and scheduling | `max_model_len=131072`; `max_num_seqs` not explicitly set |
| Workload | AgentX `inferencex-agentx-mvp`, concurrency 16, seed 42, duration 1,800 s, streaming |
| Dataset | `semianalysisai/cc-traces-weka-with-subagents-060826` |
| Warmup | 17 requests in each cell before profiling |
| Shared memory | 200 GiB in all cells; CPU mmap newly created at model startup |
| Placement | Baseline and CPU160 on `diadochos-hqxzk-gpu-h100-6kl5z`; CPU64 on `diadochos-hqxzk-gpu-h100-mt46x` |
| Timing | Baseline and CPU64 ran concurrently; CPU160 ran approximately six hours later |
| Restarts | Zero model and router restarts in all captured pod descriptions |

## Comparison evidence

Figure 1 compares completed-request throughput from `results/profile_export_aiperf.json` across the 30-minute profiling runs.

```vega-lite
{"width":720,"height":300,"title":"Figure 1 \u2014 Completed-request throughput","data":{"values":[{"configuration":"No offload","requests_s":0.06619194632166268},{"configuration":"CPU offload 64 GiB","requests_s":0.06550533822973867},{"configuration":"CPU offload 160 GiB","requests_s":0.0661248947697594}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"]},"y":{"field":"requests_s","type":"quantitative","title":"Request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"],"range":["#1f77b4","#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"requests_s","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU64 is 1.04% below baseline and CPU160 is 0.10% below baseline. These differences are smaller than the uncertainty implied by one run per cell and do not establish a throughput ranking.

Figure 2 compares output-token throughput from the same AIPerf exports.

```vega-lite
{"width":720,"height":300,"title":"Figure 2 \u2014 Output-token throughput","data":{"values":[{"configuration":"No offload","output_tokens_s":50.27305675174215},{"configuration":"CPU offload 64 GiB","output_tokens_s":49.48130128967402},{"configuration":"CPU offload 160 GiB","output_tokens_s":49.868089390614045}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"]},"y":{"field":"output_tokens_s","type":"quantitative","title":"Output-token throughput (tokens/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"],"range":["#1f77b4","#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"output_tokens_s","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU64 is 1.57% lower and CPU160 is 0.81% lower than baseline. Mean output length differs by less than 0.71% across cells, so output-shape drift is small but cannot be completely removed from a fixed-duration trace replay.

Figure 3 compares mean and P95 TTFT and end-to-end request latency from native completed-request samples.

```vega-lite
{"width":760,"height":320,"title":"Figure 3 \u2014 Request latency","data":{"values":[{"configuration":"No offload","metric":"TTFT mean","seconds":228.28792417577682},{"configuration":"No offload","metric":"TTFT P95","seconds":321.786914992},{"configuration":"No offload","metric":"E2E mean","seconds":266.3814219308595},{"configuration":"No offload","metric":"E2E P95","seconds":372.939898392},{"configuration":"CPU offload 64 GiB","metric":"TTFT mean","seconds":249.96758578315965},{"configuration":"CPU offload 64 GiB","metric":"TTFT P95","seconds":348.89249515599994},{"configuration":"CPU offload 64 GiB","metric":"E2E mean","seconds":286.5045160955126},{"configuration":"CPU offload 64 GiB","metric":"E2E P95","seconds":406.39487997769993},{"configuration":"CPU offload 160 GiB","metric":"TTFT mean","seconds":240.68260271566663},{"configuration":"CPU offload 160 GiB","metric":"TTFT P95","seconds":306.51878655459996},{"configuration":"CPU offload 160 GiB","metric":"E2E mean","seconds":277.419361434875},{"configuration":"CPU offload 160 GiB","metric":"E2E P95","seconds":379.86337687155}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["TTFT mean","TTFT P95","E2E mean","E2E P95"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"]},"y":{"field":"seconds","type":"quantitative","title":"Latency (seconds)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"],"range":["#1f77b4","#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"seconds","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU64 is worse for every plotted request-latency statistic. CPU160 narrows the mean regressions to +5.43% TTFT and +4.14% E2E and improves P95 TTFT by 4.74%, but its P95 E2E remains 1.86% worse. One unpaired sample does not establish a tail-latency benefit.

Figure 4 compares ITL from the completed-request samples.

```vega-lite
{"width":760,"height":320,"title":"Figure 4 \u2014 Inter-token latency","data":{"values":[{"configuration":"No offload","metric":"Mean","milliseconds":59.764674268385804},{"configuration":"No offload","metric":"P50","milliseconds":47.783617940133034},{"configuration":"No offload","metric":"P95","milliseconds":115.69527901886792},{"configuration":"CPU offload 64 GiB","metric":"Mean","milliseconds":50.994061800721354},{"configuration":"CPU offload 64 GiB","metric":"P50","milliseconds":43.89015036572622},{"configuration":"CPU offload 64 GiB","metric":"P95","milliseconds":101.10995814742004},{"configuration":"CPU offload 160 GiB","metric":"Mean","milliseconds":66.75472296334947},{"configuration":"CPU offload 160 GiB","metric":"P50","milliseconds":45.82790095948327},{"configuration":"CPU offload 160 GiB","metric":"P95","milliseconds":109.42861297384056}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Mean","P50","P95"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"]},"y":{"field":"milliseconds","type":"quantitative","title":"Inter-token latency (milliseconds)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"],"range":["#1f77b4","#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"milliseconds","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

ITL remains mixed: CPU64 is 14.67% better at the mean, whereas CPU160 is 11.70% worse; both improve P95 by 12.61% and 5.42%, respectively. These sign changes reinforce that the small deltas are not stable configuration effects.

## Request-, session-, and branch-level results

Figure 5 combines final AIPerf request, root-session, and branch counts.

```vega-lite
{"width":760,"height":320,"title":"Figure 5 \u2014 Request, session, and branch outcomes","data":{"values":[{"configuration":"No offload","outcome":"Requests completed","count":121},{"configuration":"No offload","outcome":"Requests cancelled","count":16},{"configuration":"No offload","outcome":"Sessions completed","count":3},{"configuration":"No offload","outcome":"Sessions not completed","count":15},{"configuration":"No offload","outcome":"Branches completed","count":26},{"configuration":"CPU offload 64 GiB","outcome":"Requests completed","count":119},{"configuration":"CPU offload 64 GiB","outcome":"Requests cancelled","count":15},{"configuration":"CPU offload 64 GiB","outcome":"Sessions completed","count":3},{"configuration":"CPU offload 64 GiB","outcome":"Sessions not completed","count":15},{"configuration":"CPU offload 64 GiB","outcome":"Branches completed","count":26},{"configuration":"CPU offload 160 GiB","outcome":"Requests completed","count":120},{"configuration":"CPU offload 160 GiB","outcome":"Requests cancelled","count":15},{"configuration":"CPU offload 160 GiB","outcome":"Sessions completed","count":3},{"configuration":"CPU offload 160 GiB","outcome":"Sessions not completed","count":15},{"configuration":"CPU offload 160 GiB","outcome":"Branches completed","count":26}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"outcome","type":"nominal","title":"Metric","sort":["Requests completed","Requests cancelled","Sessions completed","Sessions not completed","Branches completed"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"]},"y":{"field":"count","type":"quantitative","title":"Count","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"],"range":["#1f77b4","#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"outcome","type":"nominal"},{"field":"count","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

All three configurations complete 3 of 18 root sessions and 26 of 29 branches. Completed-request counts differ by at most two. The 15 unfinished sessions per cell are fixed-duration censoring evidence, not request errors, but they prevent a reliable completed-session latency comparison.

The native request-level distributions are retained in [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/10 - TTFT distribution|the TTFT distribution]], [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/11 - End-to-end latency distribution|the end-to-end latency distribution]], and [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/12 - Output-token samples|output-token samples]].

## Why CPU offload still does not help

Figure 6 integrates all native 15-second `prompt_tokens_rate_by_source` samples by source, then normalizes each configuration to 100%.

```vega-lite
{"width":720,"height":300,"title":"Figure 6 \u2014 Prompt-token processing share by source","data":{"values":[{"configuration":"No offload","source":"External KV transfer","share_percent":0.0},{"configuration":"No offload","source":"Local HBM hit","share_percent":0.028439629850111593},{"configuration":"No offload","source":"Local compute","share_percent":99.97156037014989},{"configuration":"CPU offload 64 GiB","source":"External KV transfer","share_percent":0.0},{"configuration":"CPU offload 64 GiB","source":"Local HBM hit","share_percent":1.176726226982799},{"configuration":"CPU offload 64 GiB","source":"Local compute","share_percent":98.82327377301719},{"configuration":"CPU offload 160 GiB","source":"External KV transfer","share_percent":2.0597625093174536},{"configuration":"CPU offload 160 GiB","source":"Local HBM hit","share_percent":1.1756566305376162},{"configuration":"CPU offload 160 GiB","source":"Local compute","share_percent":96.76458086014492}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"]},"y":{"field":"share_percent","type":"quantitative","title":"Integrated prompt-token rate share (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"Prompt source","scale":{"domain":["External KV transfer","Local HBM hit","Local compute"],"range":["#9467bd","#2ca02c","#d62728"]}},"order":{"field":"source","sort":["External KV transfer","Local HBM hit","Local compute"]},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"source","type":"nominal"},{"field":"share_percent","type":"quantitative","format":".4f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

External-KV transfer contributes 0.000% in baseline and CPU64, then rises to 2.06% in CPU160. Local computation falls from 98.82% at CPU64 to 96.76% at CPU160. This is the clearest evidence that larger capacity enables real restoration, but the external share remains small relative to the workload's approximately 93–94% theoretical within-play reuse. Exact time series: [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/01 - Prompt-token sources - Baseline|Baseline]], [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/02 - Prompt-token sources - CPU offload|CPU64]], and [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/13 - Prompt-token sources - CPU offload 160 GiB|CPU160]].

Figure 7 summarizes native 15-second scheduler series.

```vega-lite
{"width":760,"height":320,"title":"Figure 7 \u2014 Scheduler pressure","data":{"values":[{"configuration":"No offload","metric":"Mean running","requests":2.9526627218934913},{"configuration":"No offload","metric":"Peak running","requests":17.0},{"configuration":"No offload","metric":"Mean waiting","requests":13.366863905325443},{"configuration":"No offload","metric":"Peak waiting","requests":36.0},{"configuration":"CPU offload 64 GiB","metric":"Mean running","requests":2.75},{"configuration":"CPU offload 64 GiB","metric":"Peak running","requests":11.0},{"configuration":"CPU offload 64 GiB","metric":"Mean waiting","requests":14.05952380952381},{"configuration":"CPU offload 64 GiB","metric":"Peak waiting","requests":36.0},{"configuration":"CPU offload 160 GiB","metric":"Mean running","requests":2.761904761904762},{"configuration":"CPU offload 160 GiB","metric":"Peak running","requests":19.0},{"configuration":"CPU offload 160 GiB","metric":"Mean waiting","requests":13.345238095238095},{"configuration":"CPU offload 160 GiB","metric":"Peak waiting","requests":36.0}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Mean running","Peak running","Mean waiting","Peak waiting"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"]},"y":{"field":"requests","type":"quantitative","title":"Requests (count)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"],"range":["#1f77b4","#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"requests","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU160 averages 2.76 running and 13.35 waiting requests, essentially matching baseline's 2.95 and 13.37. All waiting is attributed to `capacity`; `deferred` remains zero. The larger tier enables restoration but has not yet drained the queue. Native views: [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/04 - Running requests|running requests]] and [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/05 - Waiting requests|waiting requests]].

Figure 8 summarizes native 15-second GPU KV-cache occupancy.

```vega-lite
{"width":760,"height":320,"title":"Figure 8 \u2014 GPU KV-cache utilization","data":{"values":[{"configuration":"No offload","metric":"Mean","percent":70.39613385724162},{"configuration":"No offload","metric":"Peak","percent":99.79096677864875},{"configuration":"CPU offload 64 GiB","metric":"Mean","percent":68.05177838212552},{"configuration":"CPU offload 64 GiB","metric":"Peak","percent":99.94027622247107},{"configuration":"CPU offload 160 GiB","metric":"Mean","percent":67.53799392097264},{"configuration":"CPU offload 160 GiB","metric":"Peak","percent":99.08921239268383}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Mean","Peak"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"]},"y":{"field":"percent","type":"quantitative","title":"GPU KV-cache utilization (%)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"],"range":["#1f77b4","#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"percent","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

All cells repeatedly approach full GPU KV occupancy. CPU160 averages 67.54% and peaks at 99.09%, while baseline averages 70.40% and peaks at 99.79%. GPU spill is already being forced at `gpu_memory_utilization=0.9`; lowering the setting is unnecessary and would increase CPU churn. See [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/03 - GPU KV-cache utilization|the full utilization time series]].

Figure 9 shows the final cumulative CPU-tier transfer counters. A point plot with a logarithmic axis is used because the directions differ by more than three orders of magnitude.

```vega-lite
{"width":740,"height":300,"title":"Figure 9 \u2014 CPU-tier cumulative transfer volume","data":{"values":[{"series":"64 GiB \u00b7 GPU \u2192 CPU stores","configuration":"CPU offload 64 GiB","gib":3743.28125},{"series":"64 GiB \u00b7 CPU \u2192 GPU loads","configuration":"CPU offload 64 GiB","gib":1.5625},{"series":"160 GiB \u00b7 GPU \u2192 CPU stores","configuration":"CPU offload 160 GiB","gib":3711.109375},{"series":"160 GiB \u00b7 CPU \u2192 GPU loads","configuration":"CPU offload 160 GiB","gib":77.5859375}]},"mark":{"type":"point","filled":true,"size":170,"tooltip":true},"encoding":{"x":{"field":"series","type":"nominal","title":"CPU capacity and transfer direction","sort":["64 GiB \u00b7 GPU \u2192 CPU stores","64 GiB \u00b7 CPU \u2192 GPU loads","160 GiB \u00b7 GPU \u2192 CPU stores","160 GiB \u00b7 CPU \u2192 GPU loads"]},"y":{"field":"gib","type":"quantitative","title":"Cumulative transfer volume (GiB, logarithmic)","scale":{"type":"log"}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["CPU offload 64 GiB","CPU offload 160 GiB"],"range":["#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"series","type":"nominal"},{"field":"gib","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU64 stores 3,743.3 GiB and loads 1.562 GiB, a 2,396:1 ratio. CPU160 stores 3,711.1 GiB and loads 77.6 GiB, improving to 47.8:1. The roughly 49.7× read-back increase proves capacity sensitivity, but CPU160 remains strongly write-dominated. Native transfer series: [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/06 - CPU offload transfers|CPU64]] and [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/16 - CPU160 offload transfers|CPU160]].

Model-container working set averages 506.3 GiB at baseline, 571.5 GiB at CPU64, and 669.5 GiB at CPU160, confirming both allocations. See [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/07 - Model-container memory|model memory]].

## Retention-clock analysis

The CPU tier acts as an LRU cache under continuous writes. Its approximate characteristic retention time is:

$$
t_{retain} = \frac{B_{CPU}}{\dot B_{GPU\rightarrow CPU}}
$$

Figure 10 compares the shelf-derived retention estimate with completed-request TTFT. TTFT is already a lower bound on the next-turn reuse gap because AgentX client think time is additional.

```vega-lite
{"width":760,"height":320,"title":"Figure 10 \u2014 CPU retention estimate versus TTFT lower bound","data":{"values":[{"configuration":"CPU offload 64 GiB","metric":"Estimated retention","seconds":42.485109450072464},{"configuration":"CPU offload 64 GiB","metric":"Mean TTFT","seconds":249.96758578315965},{"configuration":"CPU offload 64 GiB","metric":"P95 TTFT","seconds":348.89249515599994},{"configuration":"CPU offload 160 GiB","metric":"Estimated retention","seconds":107.19452188343719},{"configuration":"CPU offload 160 GiB","metric":"Mean TTFT","seconds":240.68260271566663},{"configuration":"CPU offload 160 GiB","metric":"P95 TTFT","seconds":306.51878655459996}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Estimated retention","Mean TTFT","P95 TTFT"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"]},"y":{"field":"seconds","type":"quantitative","title":"Time (seconds)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 64 GiB","CPU offload 160 GiB"],"range":["#1f77b4","#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"seconds","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

CPU160 extends estimated retention from 42.5 to 107.2 seconds, which is enough for some reuse but remains less than half of its 240.7-second mean TTFT. At the observed write rate, covering mean TTFT alone requires about 359 GiB; adding the workload's maximum 60-second think gap raises that to 449 GiB. P95 TTFT plus 60 seconds implies about 547 GiB. The direct `kv_block_reuse_gap` histograms are unavailable—all samples are NaN—so these are sizing estimates, not measured reuse-gap quantiles. Eviction/lifetime telemetry: [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/14 - Retention and eviction telemetry - CPU64|CPU64]] and [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/15 - Retention and eviction telemetry - CPU160|CPU160]].

## What Revision 2 changes from Revision 1

| Measure | Revision 1 · C32 baseline | Revision 1 · C32 CPU64 | Revision 2 · C16 baseline | Revision 2 · C16 CPU64 | Revision 2 · C16 CPU160 |
|---|---:|---:|---:|---:|---:|
| Requests/s | 0.06737 | 0.06770 | 0.06619 | 0.06551 | 0.06612 |
| Mean TTFT | 507.9 s | 496.1 s | 228.3 s | 250.0 s | 240.7 s |
| Mean E2E | 543.4 s | 539.2 s | 266.4 s | 286.5 s | 277.4 s |
| Mean ITL | 50.7 ms | 60.6 ms | 59.8 ms | 51.0 ms | 66.8 ms |
| Mean waiting | 26.69 | 26.61 | 13.37 | 14.06 | 13.35 |
| Completed sessions | 2 / 32 | 2 / 32 | 3 / 18 | 3 / 18 | 3 / 18 |
| CPU stores | N/A | 4,514.3 GiB | N/A | 3,743.3 GiB | 3,711.1 GiB |
| CPU loads | N/A | 3.523 GiB | N/A | 1.562 GiB | 77.6 GiB |
| Store/load ratio | N/A | 1,281:1 | N/A | 2,396:1 | 47.8:1 |
| External-KV prompt share | 0% | 0% | 0% | 0% | 2.06% |

Figure 11 plots the CPU-versus-baseline effect inside each revision and for both Revision 2 CPU capacities. Positive latency deltas are regressions; positive throughput deltas are improvements.

```vega-lite
{"width":760,"height":330,"title":"Figure 11 \u2014 CPU-offload effect by revision and capacity","data":{"values":[{"comparison":"Revision 1 \u00b7 C32 \u00b7 CPU64","metric":"Request throughput","cpu_delta_percent":0.4876145492047623},{"comparison":"Revision 1 \u00b7 C32 \u00b7 CPU64","metric":"Mean TTFT","cpu_delta_percent":-2.318804560131038},{"comparison":"Revision 1 \u00b7 C32 \u00b7 CPU64","metric":"Mean E2E","cpu_delta_percent":-0.7714371045509427},{"comparison":"Revision 1 \u00b7 C32 \u00b7 CPU64","metric":"Mean ITL","cpu_delta_percent":19.61721692304641},{"comparison":"Revision 2 \u00b7 C16 \u00b7 CPU64","metric":"Request throughput","cpu_delta_percent":-1.0372985386884026},{"comparison":"Revision 2 \u00b7 C16 \u00b7 CPU64","metric":"Mean TTFT","cpu_delta_percent":9.496630925904759},{"comparison":"Revision 2 \u00b7 C16 \u00b7 CPU64","metric":"Mean E2E","cpu_delta_percent":7.554240839616866},{"comparison":"Revision 2 \u00b7 C16 \u00b7 CPU64","metric":"Mean ITL","cpu_delta_percent":-14.675245159503714},{"comparison":"Revision 2 \u00b7 C16 \u00b7 CPU160","metric":"Request throughput","cpu_delta_percent":-0.10129865584771469},{"comparison":"Revision 2 \u00b7 C16 \u00b7 CPU160","metric":"Mean TTFT","cpu_delta_percent":5.429406125900105},{"comparison":"Revision 2 \u00b7 C16 \u00b7 CPU160","metric":"Mean E2E","cpu_delta_percent":4.143659653142184},{"comparison":"Revision 2 \u00b7 C16 \u00b7 CPU160","metric":"Mean ITL","cpu_delta_percent":11.695953806379645}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Request throughput","Mean TTFT","Mean E2E","Mean ITL"]},"xOffset":{"field":"comparison","sort":["Revision 1 \u00b7 C32 \u00b7 CPU64","Revision 2 \u00b7 C16 \u00b7 CPU64","Revision 2 \u00b7 C16 \u00b7 CPU160"]},"y":{"field":"cpu_delta_percent","type":"quantitative","title":"CPU-offload delta versus same-revision baseline (%)","scale":{"zero":true}},"color":{"field":"comparison","type":"nominal","title":"Comparison","scale":{"domain":["Revision 1 \u00b7 C32 \u00b7 CPU64","Revision 2 \u00b7 C16 \u00b7 CPU64","Revision 2 \u00b7 C16 \u00b7 CPU160"],"range":["#7f7f7f","#ff7f0e","#2ca02c"]}},"tooltip":[{"field":"comparison","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"cpu_delta_percent","type":"quantitative","format":"+.2f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

Request throughput remains effectively flat across Revision 2 capacities. CPU160 narrows CPU64's mean-TTFT regression and produces real external reuse, but it does not improve aggregate throughput or completed sessions. The durable new observation is a capacity response in the restoration mechanism, not an end-user performance win.

## Additional time-series evidence

- [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/08 - Generation throughput|Generation throughput at native 15-second cadence]]
- [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/09 - TTFT P90|Server-side TTFT P90 at native 15-second cadence]]
- [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/13 - Prompt-token sources - CPU offload 160 GiB|CPU160 prompt-token sources at native 15-second cadence]]
- [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/14 - Retention and eviction telemetry - CPU64|CPU64 eviction telemetry]]
- [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/15 - Retention and eviction telemetry - CPU160|CPU160 eviction telemetry]]
- [[2026-07-29 - Baseline vs CPU offload experiment - Revision 2/16 - CPU160 offload transfers|CPU160 transfer bandwidth]]

## Saturation, errors, and validity evidence

- All three runs report zero AIPerf request errors.
- All model and router pods have restart count zero.
- Searches of model logs found no traceback, CUDA OOM, store-refusal, or unavailable-block storm.
- All AIPerf runs reached the 1,800-second sending timeout and 30-second grace timeout.
- Baseline cancelled 16 of 137 sent requests; CPU64 cancelled 15 of 134; CPU160 cancelled 15 of 135.
- Each run had one abandoned pending branch join during cleanup. This follows the fixed-duration cutoff and does not indicate a model-serving error.
- Capacity is the only nonzero waiting reason. Deferred waiting remains zero.
- CPU160 ran later than the original pair. One repetition per cell cannot separate capacity effect from run-to-run variation for small outcome deltas, although the 49.7× load-volume change is strong mechanism evidence.

## Conclusions

### Established by the data

- At concurrency 16, neither 64 GiB nor 160 GiB improves request throughput, output-token throughput, mean latency, queueing, or completed sessions.
- Both CPU tiers are definitely enabled. CPU64 initializes 16,384 blocks and CPU160 initializes 40,960 blocks; model-container memory rises in proportion to allocation.
- CPU160 changes the mechanism: CPU→GPU loads rise from 1.562 to 77.6 GiB, and external-KV prompt share rises from 0% to 2.06%.
- The capacity response proves that restoration works and that CPU64 was too small, but CPU160 remains below the retention requirement and yields no end-user benefit in this sample.

### Uncertainty and limitations

- There is one run per configuration and different node placement.
- CPU160 was not run concurrently with baseline or CPU64.
- Revision 1 and Revision 2 use different concurrency and are not statistical repetitions.
- Most root sessions are censored by the fixed-duration cutoff.
- `kv_block_reuse_gap` telemetry is present as a query but all samples are NaN; direct reuse-gap quantiles remain unavailable.
- CPU memory bandwidth, NUMA placement, per-block load misses, and lookup-failure telemetry remain unavailable.

### Decision

Do not recommend 64 GiB or 160 GiB CPU offload for this AgentX workload as performance configurations. CPU160 proves the restoration path, so the next decisive test is a retention-sized tier: 512 GiB at `gpu_memory_utilization=0.9`, or 640 GiB when targeting P95 TTFT plus the possible 60-second think gap.

## Next experiment

1. Keep `gpu_memory_utilization=0.9` and test CPU512 at C16. Increase `/dev/shm` from 200 GiB to at least 600 GiB and verify node-memory headroom first.
2. Pair No offload, CPU160, and CPU512 on the same node/time block with at least three randomized repetitions.
3. Require the retention estimate to exceed mean next-turn gap, CPU→GPU loads to move toward the same order of magnitude as stores, external prompt share to rise materially, and local-compute share to fall.
4. If tail retention is the objective, add CPU640; the current P95-TTFT-plus-60-second estimate is approximately 547 GiB.
5. Only after the capacity step, calibrate `gpu_memory_utilization` upward at 0.90, 0.92, and 0.94 while holding CPU capacity fixed. Reconcile server-reported GPU blocks and reject OOM/configuration drift.
6. Do not lower GPU utilization: it would shrink the 214,336-token GPU shelf, increase GPU→CPU churn, and shorten effective CPU retention.
7. Add a direct next-turn reuse-gap metric or derive it from request traces; the current histogram query returns only NaN.

## Run registry and provenance

| Configuration | Child/deployment | MLflow run | Artifact sources | Disposition |
|---|---|---|---|---|
| No offload | `cpu-offloading-c0c856` / `cpu-kv-offload-distributed-default` | [5e6ea11348e5401086cfe71599306b37](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/5e6ea11348e5401086cfe71599306b37?workspace=benchflow) | AIPerf, model/load-generator logs, manifests, Prometheus | Conditionally accepted |
| CPU offload 64 GiB | `cpu-offloading-739e0a` / `rhoai-cpu-kv-offload-64g` | [43521e84d39b41cc8fe59e52c6b95598](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/43521e84d39b41cc8fe59e52c6b95598?workspace=benchflow) | AIPerf, model/load-generator logs, manifests, Prometheus | Conditionally accepted |
| CPU offload 160 GiB | `cpu-offloading-4ae138` / `rhoai-cpu-kv-offload-160g` | [45f523ae6ebf45cf984282befaac0eeb](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/45f523ae6ebf45cf984282befaac0eeb?workspace=benchflow) | AIPerf, model/load-generator logs, manifests, Prometheus | Conditionally accepted |

Artifacts were downloaded on 2026-07-29. Tables use `results/profile_export_aiperf.json`, `benchmark/profile_export.jsonl`, final AIPerf phase logs, rendered manifests, model logs, and Prometheus JSON. Prometheus companion charts retain the native 15-second samples; no smoothing or downsampling was applied. Prompt-source shares integrate native rate samples and normalize by the sum across sources. The available Prometheus windows include startup, warmup, profiling, grace, and capture rather than only the 1,800-second profiling phase.