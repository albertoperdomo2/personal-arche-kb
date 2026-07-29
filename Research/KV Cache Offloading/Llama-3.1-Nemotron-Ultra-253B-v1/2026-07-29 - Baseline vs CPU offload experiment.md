---
title: "2026-07-29 - Baseline vs CPU offload experiment"
date: "2026-07-29"
type: "experiment-report"
status: "complete"
model: "RedHatAI/Llama-3_1-Nemotron-Ultra-253B-v1"
benchmark: "aiperf-agentx-inference"
experiment: 325
runs: 2
---

# 2026-07-29 - Baseline vs CPU offload experiment

## Benchmark overview

This experiment compares `RedHatAI/Llama-3_1-Nemotron-Ultra-253B-v1` with no KV offload against the same model with a 64 GiB CPU tier. Both runs used one replica, TP8 on eight H100 GPUs, vLLM `v0.23.0`, `--gpu-memory-utilization=0.9`, a 131,072-token maximum context, prefix caching, 32 concurrent AgentX sessions, seed 42, streaming, and a 1,800-second profiling window. The only intended runtime difference was `TieringOffloadingSpec` with `cpu_bytes_to_use=68719476736`.

## Executive summary

**Verdict: no demonstrated end-user benefit, and no statistically defensible winner.** CPU offload changed request throughput by only **+0.5%** and output-token throughput by **+2.7%**. It nominally reduced mean TTFT by **2.3%** and mean E2E latency by **0.8%**, but mean ITL became **19.6% worse**. With only one run per configuration, these small mixed changes should not be called parity.

The important finding is mechanistic. CPU offload was initialized and heavily exercised: the model allocated a 68.72 GB shared-memory region and wrote **4,514.3 GiB** of KV data from GPU to CPU. It loaded only **3.52 GiB** back, a store-to-load ratio of roughly **1,281:1**. Prometheus attributed **0.000%** of prompt tokens to external KV transfer and **98.94%** to local recomputation. In this workload, the CPU tier acted almost entirely as an eviction sink rather than a reusable cache.

Both runs remained capacity-bound: mean waiting depth was **26.7 vs 26.6**, both peaked at **54**, and the `capacity` series matched total waiting while `deferred` stayed zero. Only **2 of 32** root sessions completed in either run before the fixed-duration cutoff. The CPU tier therefore neither drained the queue nor improved session completion.

### Errors and validity

- AIPerf reported **0 request errors** in both runs; model logs show successful engine startup.
- Both runs reached the post-profile grace-period timeout. Baseline completed 122 requests and cancelled 33 in flight; CPU offload completed 123 and cancelled 32.
- Both spawned 33 branches and completed 30. Baseline truncated 3 branches; CPU offload truncated 2.
- The runs overlapped in time but their model pods ran on different H100 nodes (`diadochos-hqxzk-gpu-h100-gjfjh` and `diadochos-hqxzk-gpu-h100-mt46x`), leaving a residual node-level and shared-cluster confounder.
- **Validity: conditionally valid.** The runs are useful for request-level and cache-mechanism diagnosis, but not conclusive for completed-session performance or small configuration deltas.

## Configuration map

| Configuration | Child | Deployment | Model node | KV tier | MLflow |
|---|---|---|---|---|---|
| Baseline | `cpu-offloading-b643e1` | `cpu-kv-offload-distributed-default` | `diadochos-hqxzk-gpu-h100-gjfjh` | GPU only | [run `c2720531`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/c272053125704ddea1941213a8e84f5d?workspace=benchflow) |
| CPU offload 64 GiB | `cpu-offloading-4469b7` | `rhoai-cpu-kv-offload-64g` | `diadochos-hqxzk-gpu-h100-mt46x` | GPU + 64 GiB CPU | [run `98586342`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/9858634252fa410da396f69a3eaf6816?workspace=benchflow) |

## Headline results

| Metric | Baseline | CPU offload 64 GiB | CPU delta | Interpretation |
|---|---:|---:|---:|---|
| Completed requests | 122 | 123 | +0.8% | No meaningful separation |
| Request throughput | 0.06737 req/s | 0.06770 req/s | +0.5% | Near-identical |
| Output-token throughput | 46.87 tok/s | 48.15 tok/s | +2.7% | Small nominal CPU lead |
| Mean TTFT | 507.9 s | 496.1 s | -2.3% | Small nominal improvement |
| P95 TTFT | 690.0 s | 656.4 s | -4.9% | CPU lower |
| Mean E2E latency | 543.4 s | 539.2 s | -0.8% | Near-identical |
| Mean ITL | 50.7 ms | 60.6 ms | +19.6% | CPU worse |
| P95 ITL | 97.9 ms | 137.8 ms | +40.7% | CPU worse |
| Mean input length | 55,309 tokens | 55,508 tokens | +0.4% | Closely matched |
| Total output tokens | 84,880 | 87,481 | +3.1% | Fixed-duration outcome |
| Root sessions completed | 2 / 32 | 2 / 32 | 0 | No session-level gain |
| Mean waiting requests | 26.7 | 26.6 | -0.1 | Same capacity pressure |

## Request-level performance

The request-rate bars are effectively the same. Output-token throughput is slightly higher with CPU offload, but the run also emitted 3.1% more output tokens and had one additional completed request, so a single fixed-duration sample cannot establish a throughput win.

```vega-lite
{"width":700,"height":300,"title":"Completed request throughput","data":{"values":[{"configuration":"Baseline","requests_s":0.06737300023660386},{"configuration":"CPU offload 64 GiB","requests_s":0.0677015207879933}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":null,"sort":["Baseline","CPU offload 64 GiB"]},"y":{"field":"requests_s","type":"quantitative","title":"Requests per second","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["Baseline","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"requests_s","type":"quantitative","format":".5f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

```vega-lite
{"width":700,"height":300,"title":"Output-token throughput","data":{"values":[{"configuration":"Baseline","tokens_s":46.873936558056855},{"configuration":"CPU offload 64 GiB","tokens_s":48.1511930085727}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":null,"sort":["Baseline","CPU offload 64 GiB"]},"y":{"field":"tokens_s","type":"quantitative","title":"Output tokens per second","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["Baseline","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"tokens_s","type":"quantitative","format":".2f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

TTFT and E2E latency move modestly in favor of CPU offload, especially at P50, while decode pacing moves in the opposite direction. This mixed pattern is consistent with a tier that can relieve some GPU eviction pressure but adds transfer/management work without delivering substantial cache reuse.

```vega-lite
{"width":720,"height":320,"title":"TTFT and end-to-end latency","data":{"values":[{"configuration":"Baseline","measure":"TTFT Mean","seconds":507.87012737223773},{"configuration":"Baseline","measure":"TTFT P50","seconds":576.2507098135},{"configuration":"Baseline","measure":"TTFT P95","seconds":689.9655665701999},{"configuration":"Baseline","measure":"E2E request Mean","seconds":543.4351945017622},{"configuration":"Baseline","measure":"E2E request P50","seconds":608.25790648},{"configuration":"Baseline","measure":"E2E request P95","seconds":773.2123165463499},{"configuration":"CPU offload 64 GiB","measure":"TTFT Mean","seconds":496.09361169918697},{"configuration":"CPU offload 64 GiB","measure":"TTFT P50","seconds":526.61831376},{"configuration":"CPU offload 64 GiB","measure":"TTFT P95","seconds":656.3664599198},{"configuration":"CPU offload 64 GiB","measure":"E2E request Mean","seconds":539.242933772187},{"configuration":"CPU offload 64 GiB","measure":"E2E request P50","seconds":564.445998839},{"configuration":"CPU offload 64 GiB","measure":"E2E request P95","seconds":754.7939651639999}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"measure","type":"nominal","title":"Metric and statistic","sort":["TTFT Mean","TTFT P50","TTFT P95","E2E request Mean","E2E request P50","E2E request P95"]},"xOffset":{"field":"configuration","sort":["Baseline","CPU offload 64 GiB"]},"y":{"field":"seconds","type":"quantitative","title":"Seconds","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["Baseline","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"measure","type":"nominal"},{"field":"seconds","type":"quantitative","format":".3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

```vega-lite
{"width":720,"height":320,"title":"Inter-token latency","data":{"values":[{"configuration":"Baseline","statistic":"Mean","milliseconds":50.67031369452795},{"configuration":"Baseline","statistic":"P50","milliseconds":39.49510054764245},{"configuration":"Baseline","statistic":"P95","milliseconds":97.94721973673418},{"configuration":"CPU offload 64 GiB","statistic":"Mean","milliseconds":60.61041904757159},{"configuration":"CPU offload 64 GiB","statistic":"P50","milliseconds":50.25169449475262},{"configuration":"CPU offload 64 GiB","statistic":"P95","milliseconds":137.79991653981924}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"statistic","type":"nominal","title":"Statistic","sort":["Mean","P50","P95"]},"xOffset":{"field":"configuration","sort":["Baseline","CPU offload 64 GiB"]},"y":{"field":"milliseconds","type":"quantitative","title":"Milliseconds","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["Baseline","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"statistic","type":"nominal"},{"field":"milliseconds","type":"quantitative","format":".3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

## Session- and branch-level outcomes

Both runs launched 32 root sessions but only two completed. The final load-generator log reports two session cancellations; the other 28 root sessions did not reach terminal completion before the run ended. Because nearly all sessions are censored by the fixed-duration cutoff, there is no valid completed-session latency distribution to compare. Across the completed request records, each root session produced a median of 3 completed requests; the maximum was 24 in both runs.

```vega-lite
{"width":720,"height":320,"title":"Request and root-session outcomes at cutoff","data":{"values":[{"configuration":"Baseline","outcome":"Requests completed","count":122},{"configuration":"Baseline","outcome":"Requests cancelled at cutoff","count":33},{"configuration":"Baseline","outcome":"Root sessions completed","count":2},{"configuration":"Baseline","outcome":"Root sessions not completed","count":30},{"configuration":"CPU offload 64 GiB","outcome":"Requests completed","count":123},{"configuration":"CPU offload 64 GiB","outcome":"Requests cancelled at cutoff","count":32},{"configuration":"CPU offload 64 GiB","outcome":"Root sessions completed","count":2},{"configuration":"CPU offload 64 GiB","outcome":"Root sessions not completed","count":30}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"outcome","type":"nominal","title":"Outcome","sort":["Requests completed","Requests cancelled at cutoff","Root sessions completed","Root sessions not completed"]},"xOffset":{"field":"configuration","sort":["Baseline","CPU offload 64 GiB"]},"y":{"field":"count","type":"quantitative","title":"Count","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["Baseline","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"outcome","type":"nominal"},{"field":"count","type":"quantitative","format":".3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

## Why the CPU tier did not help

### 1. Almost no evicted KV returned to the GPU

The asymmetric transfer counters are the clearest explanation. CPU offload stored more than four tebibytes but loaded only 3.52 GiB back. The CPU-to-GPU rate was zero throughout the sampled profiling interval; the small cumulative load counter was already present when that series appeared. This is not a bandwidth-shortage signature. It is a reuse/lookup signature: blocks were evicted, but subsequent requests did not retrieve them.

```vega-lite
{"width":700,"height":300,"title":"CPU-offload cumulative transfer volume","data":{"values":[{"direction":"GPU \u2192 CPU stores","gib":4514.2890625},{"direction":"CPU \u2192 GPU loads","gib":3.5234375}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"direction","type":"nominal","title":null,"sort":["GPU \u2192 CPU stores","CPU \u2192 GPU loads"]},"y":{"field":"gib","type":"quantitative","title":"GiB (log scale)","scale":{"type":"log"}},"color":{"field":"direction","type":"nominal","legend":null,"scale":{"domain":["GPU \u2192 CPU stores","CPU \u2192 GPU loads"],"range":["#ff7f0e","#9467bd"]}},"tooltip":[{"field":"direction","type":"nominal"},{"field":"gib","type":"quantitative","format":",.2f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

### 2. Prompt processing remained recompute-dominated

The baseline served only 0.028% of sampled prompt-token rate from local HBM hits. CPU offload increased local-hit share to 1.06%, but external KV transfer remained 0.000%. Consequently, 98.94% of CPU-run prompt processing was still local compute. The tier changed where evicted blocks were written without making them materially reusable.

```vega-lite
{"width":720,"height":300,"title":"Prompt-token processing share by source","data":{"values":[{"configuration":"Baseline","source":"External KV transfer","share_percent":0.0},{"configuration":"Baseline","source":"Local HBM hit","share_percent":0.027614483396167535},{"configuration":"Baseline","source":"Local compute","share_percent":99.97238551660384},{"configuration":"CPU offload 64 GiB","source":"External KV transfer","share_percent":0.0},{"configuration":"CPU offload 64 GiB","source":"Local HBM hit","share_percent":1.061572311117983},{"configuration":"CPU offload 64 GiB","source":"Local compute","share_percent":98.93842768888203}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":null,"sort":["Baseline","CPU offload 64 GiB"]},"y":{"field":"share_percent","type":"quantitative","title":"Share of sampled prompt-token rate (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"Source","scale":{"domain":["External KV transfer","Local HBM hit","Local compute"],"range":["#9467bd","#2ca02c","#d62728"]}},"order":{"field":"source","sort":["External KV transfer","Local HBM hit","Local compute"]},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"source","type":"nominal"},{"field":"share_percent","type":"quantitative","format":".4f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

### 3. The tier was small relative to long-context concurrency

vLLM reported 214,336 GPU-cache tokens. The 64 GiB CPU tier contained 16,384 blocks × 16 tokens, or 262,144 tokens, for a nominal combined capacity of **476,480 tokens**. The CPU run's mean input was 55,508 tokens; 32 such contexts represent about **1,776,246 tokens**, roughly **3.7×** the combined nominal capacity before output growth. This is a demand-scale comparison, not an exact residency calculation, but it explains why reuse distance can exceed the CPU tier and why old blocks may be evicted before the next turn needs them.

### 4. Capacity pressure did not change

Mean and peak waiting depth were essentially identical. The `capacity` series matched total waiting while `deferred` stayed zero. CPU offload increased the maximum number of simultaneously running requests from 9 to 18 at some instants, but it did not improve mean queue depth or completed sessions.

```vega-lite
{"width":720,"height":320,"title":"Scheduler pressure","data":{"values":[{"configuration":"Baseline","measure":"Mean running","requests":2.6},{"configuration":"Baseline","measure":"Peak running","requests":9.0},{"configuration":"Baseline","measure":"Mean waiting","requests":26.685714285714287},{"configuration":"Baseline","measure":"Peak waiting","requests":54.0},{"configuration":"CPU offload 64 GiB","measure":"Mean running","requests":2.9622641509433962},{"configuration":"CPU offload 64 GiB","measure":"Peak running","requests":18.0},{"configuration":"CPU offload 64 GiB","measure":"Mean waiting","requests":26.61320754716981},{"configuration":"CPU offload 64 GiB","measure":"Peak waiting","requests":54.0}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"measure","type":"nominal","title":"Measure","sort":["Mean running","Peak running","Mean waiting","Peak waiting"]},"xOffset":{"field":"configuration","sort":["Baseline","CPU offload 64 GiB"]},"y":{"field":"requests","type":"quantitative","title":"Requests","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["Baseline","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"measure","type":"nominal"},{"field":"requests","type":"quantitative","format":".3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

```vega-lite
{"width":720,"height":320,"title":"GPU KV-cache utilization","data":{"values":[{"configuration":"Baseline","measure":"Mean","percent":66.40942782488135},{"configuration":"Baseline","measure":"Peak","percent":99.60432997387085},{"configuration":"CPU offload 64 GiB","measure":"Mean","percent":68.1439145837295},{"configuration":"CPU offload 64 GiB","measure":"Peak","percent":96.24486748786862}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"measure","type":"nominal","title":"Statistic","sort":["Mean","Peak"]},"xOffset":{"field":"configuration","sort":["Baseline","CPU offload 64 GiB"]},"y":{"field":"percent","type":"quantitative","title":"KV-cache utilization (%)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["Baseline","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"measure","type":"nominal"},{"field":"percent","type":"quantitative","format":".3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

### 5. Allocation was real, not silently ignored

The CPU run's model-container working set was about 65.3 GiB higher than baseline, matching the 64 GiB shared-memory allocation plus overhead. Engine logs show `TieringOffloadingManager` with 16,384 primary-tier blocks and all TP workers opening the same 68.72 GB mmap. The missing benefit is therefore not explained by a missing CPU allocation.

```vega-lite
{"width":700,"height":300,"title":"Model-container memory working set","data":{"values":[{"configuration":"Baseline","gib":506.4528313228062},{"configuration":"CPU offload 64 GiB","gib":571.7654113589592}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":null,"sort":["Baseline","CPU offload 64 GiB"]},"y":{"field":"gib","type":"quantitative","title":"GiB","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["Baseline","CPU offload 64 GiB"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"gib","type":"quantitative","format":".1f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}
```

## Interpretation

The current evidence supports two explanations that the next experiment must distinguish:

1. **Workload locality/capacity mismatch:** AgentX revisits a session after enough intervening long-context work that its CPU-resident blocks have already been overwritten. Increasing CPU capacity or reducing concurrency should produce CPU-to-GPU loads if this is the cause.
2. **Load-path or attribution defect:** the connector stores blocks but fails to find/restore them for later turns, or the prompt-source metric does not attribute those loads. A controlled exact-prefix replay with known reuse should produce a nonzero `CPU_to_GPU` counter and external-KV prompt tokens; failure would isolate the implementation/observability path.

The ITL regression deserves follow-up but is not yet causal evidence of PCIe contention. The CPU run's mean ITL was 19.6% worse and P95 was 40.7% worse, while GPU-to-CPU writes averaged about 1.41 GiB/s. A replicated test should correlate transfer intervals with ITL before assigning causality.

## Detailed time-series articles

- [[2026-07-29 - Baseline vs CPU offload experiment/01 - Prompt-token sources - Baseline|Prompt-token sources — Baseline]]
- [[2026-07-29 - Baseline vs CPU offload experiment/02 - Prompt-token sources - CPU offload|Prompt-token sources — CPU offload]]
- [[2026-07-29 - Baseline vs CPU offload experiment/03 - KV-cache utilization|KV-cache utilization]]
- [[2026-07-29 - Baseline vs CPU offload experiment/04 - Running requests|Running requests]]
- [[2026-07-29 - Baseline vs CPU offload experiment/05 - Waiting requests|Waiting requests]]
- [[2026-07-29 - Baseline vs CPU offload experiment/06 - CPU transfer|CPU transfer]]
- [[2026-07-29 - Baseline vs CPU offload experiment/07 - Model memory|Model memory]]

The time-series articles retain the native 15-second Prometheus cadence. Duplicate `kserve_vllm:*` and `vllm:*` scrape streams were identical; the charts use the `kserve_vllm:*` family once to avoid double-counting.

## Recommended next experiment

Run a paired, replicated mechanism sweep:

1. Add a controlled two-request prefix-replay smoke test before AgentX. Require nonzero CPU-to-GPU bytes and nonzero external-KV prompt-token attribution.
2. Run baseline and CPU offload at concurrency 8, 16, and 32 with three repetitions each, randomized across the same H100 node set.
3. Sweep CPU capacity through 64, 128, and 256 GiB where node memory permits.
4. Treat CPU offload as effective only if it increases CPU-to-GPU load volume, external-KV token share, completed sessions, or request throughput without materially worsening ITL.
5. Capture block-level store/load hit, miss, eviction, and lookup-failure counters. The current aggregate counters cannot distinguish capacity churn from a lookup failure.

## Provenance and reproducibility

- Experiment: MLflow experiment `325`, workspace `benchflow`
- Benchmark: `aiperf-agentx-inference`, AIPerf `0.8.0`
- Dataset: `semianalysisai/cc-traces-weka-with-subagents-060826`
- Workload: 32 concurrent sessions, seed 42, 1,800-second profiling duration, streaming
- Runtime: vLLM `v0.23.0`, RHOAI 3.5 EA2, one replica, TP8, H100, 200 GiB `/dev/shm`
- Common flags: `--trust-remote-code --no-enable-log-requests --enable-prefix-caching --kv-cache-metrics --kv-cache-metrics-sample=0.01 --gpu-memory-utilization=0.9 --max-model-len=131072`
- CPU-only delta: `--kv-transfer-config={"kv_connector":"OffloadingConnector","kv_role":"kv_both","kv_connector_extra_config":{"spec_name":"TieringOffloadingSpec","cpu_bytes_to_use":68719476736}}`
- Sources: MLflow `results/profile_export_aiperf.json`, `benchmark/profile_export.jsonl`, model and load-generator logs, rendered manifests, and Prometheus JSON artifacts
- Metric window: approximately 2026-07-29 11:47–12:41 UTC at 15-second cadence

## Run registry

| Run | Configuration | MLflow | Artifact use |
|---|---|---|---|
| `c272053125704ddea1941213a8e84f5d` | Baseline | [open run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/c272053125704ddea1941213a8e84f5d?workspace=benchflow) | AIPerf, logs, manifests, Prometheus |
| `9858634252fa410da396f69a3eaf6816` | CPU offload 64 GiB | [open run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/9858634252fa410da396f69a3eaf6816?workspace=benchflow) | AIPerf, logs, manifests, Prometheus |