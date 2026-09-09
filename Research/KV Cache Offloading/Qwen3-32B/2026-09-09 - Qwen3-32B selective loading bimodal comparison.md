---
title: "Qwen3-32B selective loading bimodal comparison"
date: "2026-09-09"
type: "experiment-report"
topic: "KV Cache Offloading"
model: "Qwen/Qwen3-32B"
status: "diagnostic-valid-performance-invalid"
experiment: "404"
runs:
  selective: "482c975ce11d4652a5fba183abe23a73"
  always_load: "eac09c9511f24914a8d4efdddc888fc1"
---

# Qwen3-32B selective loading bimodal comparison

## Executive verdict

**The runs are valid as an end-to-end mechanism check, but invalid as a causal performance comparison or threshold calibration.** The selective policy was configured and materially reduced external KV loading. However, the experiment had no clean operating region where external-tier reuse was both substantial and the deployment remained unsaturated. The two treatments also ran concurrently on the same four H100 nodes and shared the same host NVMe path, so their CPU and storage behavior was not independent.

The observed result must still be retained: selective loading slightly reduced mean TTFT at concurrency 32–128, while lowering throughput and increasing mean end-to-end latency at concurrency 64–128. At concurrency 512 it was decisively worse: total throughput fell 33.5%, mean TTFT rose 65.8%, and mean end-to-end latency rose 49.5% relative to always loading. This is consistent with short-prefix recomputation consuming GPU prefill capacity once the deployment is under heavy pressure. It is not evidence that the plugin is intrinsically harmful, because the saturated stage and shared-resource design prevent that conclusion.

The provisional 1,024-token threshold came from a different model and deployment. These runs do not calibrate that threshold for Qwen3-32B.

## Run registry

| Treatment | Deployment profile | MLflow run | Status | Window |
|---|---|---|---|---|
| Selective, threshold = 1,024 external reusable tokens | `rhoai-distributed-default-selective-loading` | [482c975ce11d4652a5fba183abe23a73](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/404/runs/482c975ce11d4652a5fba183abe23a73?workspace=benchflow) | Finished | 2026-09-08 22:20:51–23:00:51 UTC |
| Always load | `multi-tier-offloading-nvme` | [eac09c9511f24914a8d4efdddc888fc1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/404/runs/eac09c9511f24914a8d4efdddc888fc1?workspace=benchflow) | Finished | 2026-09-08 22:20:52–23:00:34 UTC |

Both deployments used Qwen3-32B, four replicas, TP2, H100 GPUs, vLLM `0.27.0`, image revision `v0.27.0-selective-load-v1`, 16,384 maximum model length, 64-token blocks, 256 GiB CPU offload per replica, and the same NVMe tier. Model startup reported 328,512 GPU KV-cache tokens per replica. All eight model pods remained running with zero restarts.

The EPP configurations matched except for the selective arm's experimental plugin:

```yaml
- name: selective-kv-policy
  type: selective-kv-policy
  parameters:
    loadPolicy: threshold
    minExternalReusableTokens: 1024
```

Both arms used the precise prefix-cache producer and the same prefix-affinity and token-load scheduling chain.

## Workload

GuideLLM ran five 300-second concurrent stages at streams 32, 64, 96, 128, and 512, after a 120-second pre-warmup at rate 64. Every stage used four-turn synthetic conversations, 256 new prompt tokens per turn, 128 output tokens, seed 42, and two equal-weight prefix buckets:

- 512 prefix tokens, intended to fall below the policy threshold;
- 8,192 prefix tokens, intended to remain load-enabled.

GuideLLM set each bucket's prefix count to twice the stage concurrency. Chat formatting and accumulated turns produced successful request prompt sizes of roughly 781–1,963 tokens for the short bucket and 8,461–9,643 for the long bucket. Request-level bucket analysis below classifies prompts at 4,096 tokens; it does not identify the actual cache source of an individual request.

## Headline results

All latency figures use successful requests from the per-stage GuideLLM JSON. Total throughput includes completed and incomplete token work, matching GuideLLM's server-throughput summary. External share is the mean of the collected 15-second Prometheus samples over each stage; the underlying query uses a five-minute `rate()` window.

| Streams | Total tok/s selective | Total tok/s control | Delta | Mean TTFT selective/control | Mean E2E selective/control | External-token share selective/control | Successful share of initiated requests selective/control |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 32 | 73,847 | 73,921 | -0.1% | 126 / 141 ms | 2.318 / 2.312 s | 0.11% / 0.60% | 99.23% / 99.26% |
| 64 | 114,350 | 118,435 | -3.4% | 165 / 188 ms | 2.982 / 2.877 s | 1.60% / 3.44% | 99.03% / 99.04% |
| 96 | 134,861 | 138,448 | -2.6% | 226 / 259 ms | 3.767 / 3.659 s | 1.49% / 3.36% | 98.75% / 98.79% |
| 128 | 147,730 | 155,421 | -4.9% | 279 / 319 ms | 4.626 / 4.405 s | 1.67% / 3.84% | 98.48% / 98.55% |
| 512 | 69,374 | 104,365 | -33.5% | 20,095 / 12,117 ms | 38.318 / 25.638 s | 7.44% / 25.62% | 87.86% / 91.72% |

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Total throughput by offered concurrency","width":720,"height":330,"data":{"values":[{"streams":32,"treatment":"Selective 1,024","tokens_per_second":73847},{"streams":64,"treatment":"Selective 1,024","tokens_per_second":114350},{"streams":96,"treatment":"Selective 1,024","tokens_per_second":134861},{"streams":128,"treatment":"Selective 1,024","tokens_per_second":147730},{"streams":512,"treatment":"Selective 1,024","tokens_per_second":69374},{"streams":32,"treatment":"Always load","tokens_per_second":73921},{"streams":64,"treatment":"Always load","tokens_per_second":118435},{"streams":96,"treatment":"Always load","tokens_per_second":138448},{"streams":128,"treatment":"Always load","tokens_per_second":155421},{"streams":512,"treatment":"Always load","tokens_per_second":104365}]},"mark":{"type":"line","point":true},"encoding":{"x":{"field":"streams","type":"quantitative","title":"Concurrent streams","scale":{"type":"log","base":2}},"y":{"field":"tokens_per_second","type":"quantitative","title":"Total tokens/s","scale":{"zero":true}},"color":{"field":"treatment","type":"nominal","title":"Treatment"},"tooltip":[{"field":"streams","type":"quantitative"},{"field":"treatment","type":"nominal"},{"field":"tokens_per_second","type":"quantitative","format":",.0f"}]}}
```

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Mean request latency","width":720,"height":330,"data":{"values":[{"streams":32,"treatment":"Selective 1,024","seconds":2.318},{"streams":64,"treatment":"Selective 1,024","seconds":2.982},{"streams":96,"treatment":"Selective 1,024","seconds":3.767},{"streams":128,"treatment":"Selective 1,024","seconds":4.626},{"streams":512,"treatment":"Selective 1,024","seconds":38.318},{"streams":32,"treatment":"Always load","seconds":2.312},{"streams":64,"treatment":"Always load","seconds":2.877},{"streams":96,"treatment":"Always load","seconds":3.659},{"streams":128,"treatment":"Always load","seconds":4.405},{"streams":512,"treatment":"Always load","seconds":25.638}]},"mark":{"type":"line","point":true},"encoding":{"x":{"field":"streams","type":"quantitative","title":"Concurrent streams","scale":{"type":"log","base":2}},"y":{"field":"seconds","type":"quantitative","title":"Mean end-to-end latency (s)","scale":{"zero":true}},"color":{"field":"treatment","type":"nominal","title":"Treatment"},"tooltip":[{"field":"streams","type":"quantitative"},{"field":"treatment","type":"nominal"},{"field":"seconds","type":"quantitative","format":".3f"}]}}
```

## Mechanism evidence

The selective arm clearly changed load behavior. External prompt-token share was approximately halved at streams 64–128 and reduced by 71% at streams 512. Mean CPU-to-GPU load traffic showed the same direction: approximately 656 versus 1,487 MiB/s at streams 64, 799 versus 1,845 MiB/s at 96, 986 versus 2,369 MiB/s at 128, and 2,920 versus 19,997 MiB/s at 512.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"External KV transfer share of prompt tokens","width":720,"height":330,"data":{"values":[{"streams":32,"treatment":"Selective 1,024","percent":0.11},{"streams":64,"treatment":"Selective 1,024","percent":1.60},{"streams":96,"treatment":"Selective 1,024","percent":1.49},{"streams":128,"treatment":"Selective 1,024","percent":1.67},{"streams":512,"treatment":"Selective 1,024","percent":7.44},{"streams":32,"treatment":"Always load","percent":0.60},{"streams":64,"treatment":"Always load","percent":3.44},{"streams":96,"treatment":"Always load","percent":3.36},{"streams":128,"treatment":"Always load","percent":3.84},{"streams":512,"treatment":"Always load","percent":25.62}]},"mark":{"type":"line","point":true},"encoding":{"x":{"field":"streams","type":"quantitative","title":"Concurrent streams","scale":{"type":"log","base":2}},"y":{"field":"percent","type":"quantitative","title":"External prompt-token share (%)","scale":{"zero":true}},"color":{"field":"treatment","type":"nominal","title":"Treatment"},"tooltip":[{"field":"streams","type":"quantitative"},{"field":"treatment","type":"nominal"},{"field":"percent","type":"quantitative","format":".2f"}]}}
```

At streams 32–128, most prompt reuse remained HBM-local. The control's external share never exceeded 3.84%, so those stages provide little leverage for assessing an external-load gate. At streams 512, external reuse became material, but both deployments entered pathological pressure. The mean vLLM waiting queue was about 63 requests per engine in the selective arm and 67 in the control, with maxima of 112 and 126. Both ended with 512 incomplete requests, and successful throughput collapsed from the streams-128 knee.

Splitting successful requests by prompt length shows the same pressure transition:

| Streams | Short mean TTFT selective/control | Long mean TTFT selective/control |
|---:|---:|---:|
| 32 | 94 / 111 ms | 159 / 170 ms |
| 64 | 120 / 155 ms | 210 / 223 ms |
| 96 | 170 / 218 ms | 285 / 302 ms |
| 128 | 226 / 289 ms | 333 / 350 ms |
| 512 | 19,587 / 11,933 ms | 20,614 / 12,308 ms |

The selective arm reduced TTFT for both prompt groups at streams 32–128, possibly by reducing transfer-path contention, but its mean TPOT and end-to-end latency were worse from streams 64 onward. The likely mechanism is a resource trade: recomputing short prefixes can reduce the admission wait for individual prefills while consuming GPU work that lowers sustained decoding capacity. At streams 512 that additional compute amplifies the queue and hurts both prefix groups. This is an inference from the observed source, latency, and queue data, not a per-request causal trace.

## Why the performance comparison is invalid

1. **The treatments were not isolated.** Both matrix children ran at the same time. One model pod from each treatment was placed on each of the same four H100 nodes. Their 300-second stages overlapped by about 281–286 seconds.
2. **They shared the same secondary-tier path.** Both mounted `/var/mnt/benchflow-nvme/benchflow-kv-cache` and configured `/mnt/nvme-kv-cache` as the FS tier. They therefore shared device bandwidth, filesystem cache, and potentially content-addressed KV files. The always-load child started each stage 14–19 seconds earlier, so it may also have warmed storage for the selective child.
3. **External reuse and stable operation did not overlap.** Streams 32–128 were dominated by HBM-local hits. Streams 512 produced meaningful external reuse but severe queueing, throughput collapse, and incomplete work.
4. **There was one repetition and a fixed ascending stage order.** Cache population, filesystem page cache, thermal state, and storage carry-over are inseparable from concurrency.
5. **The collected source metrics are five-minute rolling rates.** Each benchmark stage also lasted five minutes, so neighboring stages bleed into the reported stage averages.
6. **There is no request-level decision-to-source join.** The artifact proves aggregate behavior, but cannot show that every short external match was disabled and every long external match was preserved. EPP decision logs were not collected at a level that permits this join.
7. **GPU utilization telemetry was empty.** Queue and request metrics establish saturation, but the GPU-side explanation cannot be completed from this artifact set.

These issues do not invalidate the plugin's end-to-end activation: the expected configuration was loaded, all pods were healthy, and aggregate source and transfer metrics moved in the expected direction. They invalidate the numerical policy ranking and any attempt to infer a Qwen3-32B crossover threshold.

## Corrected experiment proposal

### Phase 1: isolated mechanism cells

Run the following cells sequentially, not as concurrent matrix children:

| Prefix workload | Always load | Threshold 1,024 | Always recompute |
|---|---:|---:|---:|
| Short-only, 512 tokens | Required | Required | Diagnostic |
| Long-only, 8,192 tokens | Required | Required | Diagnostic |

Use at least three repetitions and alternate treatment order. Always recompute is a boundary check; the decisive expectations are that threshold matches always recompute for externally resident short prefixes and matches always load for externally resident long prefixes.

Make cache pressure independent of active request concurrency. With 328,512 HBM KV tokens per replica, use a fixed working set larger than HBM at streams 64–128. A practical starting point for four replicas is 2,048 short prefixes and 512 long prefixes in the later mixed test; this is about 1.31 million prefix tokens per replica before suffixes and should force both CPU and NVMe participation while fitting the configured aggregate offload capacity. Confirm the actual distribution from token-source counters before accepting the run.

Pre-warm until every intended prefix has completed at least one seed cycle and external evidence is visible; do not stop warmup after an arbitrary 120 seconds. Use a unique NVMe subdirectory per treatment and repetition, or clean and verify the exact directory between sequential cells.

### Phase 2: stable-pressure sweep

Start with streams 64, 96, and 128. Add intermediate points around the measured knee; do not reuse streams 512 until a lower stress step completes at least 95% of initiated requests without persistent queue growth. Keep model image, GPU allocation, CPU allocation, node placement, EPP chain, seed, and offload policy identical.

Validity gates for every cell:

- control external prompt-token share at least 20%;
- expected short or long prefix remains discoverable externally;
- threshold short-only external transfer is near zero and local compute rises by the expected block-aligned amount;
- threshold long-only source mix agrees with always load within 5%;
- at least 95% of initiated requests complete;
- no pod restarts, allocation failures, or monotonic queue growth;
- disjoint node/storage resources, or strictly sequential execution;
- counter deltas captured at cell boundaries rather than only five-minute rolling rates.

### Phase 3: mixed policy value

After Phase 1 validates both decisions, run the 50/50 mixed workload with the fixed pressure-making working set. Compare always load, threshold, and always recompute at each accepted concurrency. The primary decision metric is successful request throughput subject to a p95 TTFT or end-to-end SLO; report TTFT and TPOT separately so a prefill gain cannot hide a decode-capacity loss.

The threshold is acceptable for this deployment only if it improves the mixed workload over always load in repeated isolated runs, preserves the long-prefix load path, and does not move the sustainable-concurrency knee left. If the preferred action changes between low and representative pressure, a single reusable-token threshold is insufficient and the next policy should add a pressure veto or pressure-adjusted crossover.

## Conclusions

- The router-to-vLLM selective-load path activated successfully under a real Qwen3-32B, four-replica TP2 deployment.
- The policy reduced external KV loading, but the current run does not demonstrate an end-user performance benefit.
- The 1,024-token threshold cannot be transferred from the earlier Nemotron calibration to Qwen3-32B based on these data.
- The most important result is the crossover's pressure dependence: recomputing short prefixes can improve TTFT while reducing overall throughput, and becomes strongly harmful after saturation.
- The next experiment must create external-tier pressure at streams 64–128, isolate storage and nodes, and validate short and long decisions separately before evaluating the mixed policy.

## Provenance and analysis notes

Data were retrieved with the MLflow CLI from experiment 404. The analysis used run metadata, all five GuideLLM JSON artifacts per run, model and EPP logs, rendered and collected Kubernetes manifests, pod descriptions, platform-state snapshots, and Prometheus range-query artifacts. Request-level means and prompt-length splits were recomputed from successful request records. Mechanism means sum per-pod rates at each 15-second timestamp and average timestamps within the corresponding GuideLLM stage.

Related records: [[Engineering/Projects/Selective KV Loading and Offloading/07 - Selective loading calibration test plan|Selective loading calibration test plan]], [[Engineering/Projects/Selective KV Loading and Offloading/09 - 2026-09-08 Diadochos router threshold live validation|Diadochos router threshold live validation]], [[Research/KV Cache Offloading/01 - Calibration Protocol|KV Cache Offloading calibration protocol]], and [[Research/KV Cache Offloading/03 - KV Transfer Metrics and PromQL|KV transfer metrics and PromQL]].