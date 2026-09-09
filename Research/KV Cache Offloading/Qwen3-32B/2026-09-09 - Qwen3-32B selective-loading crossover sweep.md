---
title: "Qwen3-32B selective-loading crossover sweep"
date: "2026-09-09"
type: "experiment-report"
topic: "KV Cache Offloading"
experiment: "MLflow experiments 408-413"
model: "Qwen/Qwen3-32B"
vllm_version: "0.27.0"
runtime_image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v1"
scheduler_image: "quay.io/rh-ee-aperdomo/llm-d-router-endpoint-picker:dev-0a55d5da"
tensor_parallelism: 2
replicas: 4
accelerator: "8x H100 per deployment node"
gpu_memory_utilization: "vLLM default; not explicitly set"
max_model_len: 32768
max_num_seqs: "vLLM default; not explicitly set"
concurrency: [16, 32, 64, 96]
cpu_bytes_per_replica: 274877906944
offload_spec: "TieringOffloadingSpec"
secondary_tier: "filesystem on local NVMe"
secondary_tier_threads:
  read: 64
  write: 64
shared_memory: "300Gi per replica"
block_size_tokens: 64
workload: "GuideLLM single-turn fixed-prefix-corpus crossover sweep"
random_seed: 20260909
stage_duration_seconds: 300
prewarm_rate: 96
prefix_corpus_tokens: 4194304
cache_cleaning: "hostPath cleanup between deployment cells; each cell then prewarmed"
status: "conditionally-valid-threshold-inconclusive"
---

# Qwen3-32B selective-loading crossover sweep

## Executive summary

This experiment asked where externally reusable KV becomes cheaper to load than to recompute for Qwen3-32B served by four TP2 replicas on one 8×H100 node with a 256 GiB CPU tier per replica and local-NVMe secondary tier. Six reusable-prefix sizes—512, 1,024, 2,048, 4,096, 8,192, and 16,384 tokens—were tested at concurrencies 16, 32, 64, and 96. Each point compared forced loading against forced recomputation, producing 48 five-minute measurements across 12 MLflow runs.

**The sweep does not identify a defensible static crossover threshold.** Forced loading was neutral around 512–1,024 tokens at C16–C32 and increasingly harmful at C64–C96. It nominally won at 4,096 tokens, but the paired recompute arm ran on a node with one consistently slow replica. Loading then lost again at 8,192 tokens on balanced replicas. That alternating sign cannot represent a genuine prefix-length crossover.

The experiment does establish two useful results. First, at C96, forced loading delivered 36.8% less request throughput than recomputation for a 1,024-token reusable prefix and 27.6% less for a 512-token prefix. Second, the restore path was under substantial pressure: at C96, mean asynchronous lookup grew from 0.53 seconds at 512 tokens to 6.55 seconds at 8,192 tokens, blocked loading work reached about 67 requests, the vLLM waiting queue approached 96 requests, and workload-node NVMe busy time reached 100%. The problem was therefore not insufficient loading pressure; it was insufficient useful external reuse under an already congested restore path.

## Validity verdict: conditionally valid

**Valid for demonstrating loading-path pressure and the value of skipping small restores; invalid / inconclusive for selecting a crossover threshold.** All twelve MLflow runs finished, all model pods remained running with zero restarts, and the forced-recompute arms reported zero external KV loads. However, one GPU pair or replica placement on node `gjfjh` was repeatedly 3–5× slower than its peers, directly affecting the 2K-load, 4K-recompute, and 16K-load arms. Each prefix's policies also ran on different nodes, the concurrency order was always ascending, the prewarm could leave background storage work, and the Prometheus source/transfer metrics use five-minute rolling windows equal to the five-minute benchmark stages.

## Main takeaways

- **Measured:** At C96, forced load lost 27.6% request throughput at 512 reusable tokens and 36.8% at 1,024 tokens. This is direct evidence for a selective-load disable branch under high pressure.
- **Measured:** The apparent 4,096-token loading advantage ranged from +70.0% at C16 to +6.4% at C96, but its recompute control had a 20.7–38.3% per-replica throughput coefficient of variation because one replica was persistently slow. It cannot be used as crossover evidence.
- **Measured:** The balanced 8,192-token arms favored recomputation at every concurrency, by 20.7–36.4% request throughput. Loading showed 1.34–6.55 seconds of mean asynchronous lookup and increasing blocked work.
- **Measured:** Only 2.5–37.0% of prompt tokens in load-enabled cells came from external KV; most tokens were recomputed or found in HBM. At C96 the external share was only 4.1–11.2% across prefix sizes.
- **Measured:** Workload-node NVMe busy time was already 92–95% at C96 for 512–2,048 tokens and reached 100% for 4,096 tokens and above. The vLLM CPU-cache utilization metric remained at approximately 2% or less, pointing to the secondary-tier/promotion path rather than CPU-cache capacity as the observed bottleneck.
- **Inference:** A prefix-only threshold is pressure-dependent. At low concurrency, 512–1,024-token loading was roughly neutral; at C96 the same decisions were clearly harmful. A later dynamic policy should evaluate queued restore cost, but the initial static policy can still target one calibrated operating range.
- **Conclusion:** No value among 1,024, 2,048, or 4,096 tokens should be adopted from this batch. A same-node, lower-working-set focused repeat is required.

## Experiment design

The forced-load control used `multi-tier-offloading-nvme`, which leaves `kv_load_tiers` unchanged. The forced-recompute control used `rhoai-distributed-default-selective-loading-disabled`, whose EPP injects `kv_load_tiers: []`; offloading remained enabled in both directions so both controls continued to populate CPU/NVMe.

Each benchmark used one conversation turn, 256 non-prefix prompt tokens, 128 output tokens, and a fixed 4,194,304-token prefix corpus. Prefix count was inversely proportional to prefix length: 8,192 prefixes at 512 tokens, 4,096 at 1,024, 2,048 at 2,048, 1,024 at 4,096, 512 at 8,192, and 256 at 16,384. A full prefix-count prewarm ran at rate 96 before four ascending five-minute stages at C16, C32, C64, and C96. All profiles resolved to `--max-model-len=32768`.

The primary comparison is successful request throughput because every pair at a given prefix has the same request shape. Loading delta is:

$$
\Delta_{load}(L,C)=100\left(\frac{RPS_{load}(L,C)}{RPS_{recompute}(L,C)}-1\right).
$$

A usable static crossover would require this value to move from negative toward consistently positive as reusable prefix length $L$ grows at fixed concurrency $C$. Figure 1 shows that this condition was not met.

## Headline metrics

Forced recompute is the declared baseline. Positive values mean loading completed more requests per second; negative values mean recomputation won.

| Reusable prefix | C16 | C32 | C64 | C96 | Acceptance |
|---:|---:|---:|---:|---:|---|
| 512 | +0.3% | +0.3% | -4.9% | -27.6% | Useful small-prefix evidence |
| 1,024 | +5.0% | +0.2% | -19.7% | -36.8% | Useful pressure-dependence evidence |
| 2,048 | -33.3% | -37.1% | -38.5% | -52.2% | Reject: slow load replica on `gjfjh` |
| 4,096 | +70.0% | +52.3% | +22.6% | +6.4% | Reject: slow recompute replica on `gjfjh` |
| 8,192 | -36.4% | -28.3% | -25.9% | -20.7% | Valid balanced result; restore loses |
| 16,384 | -14.5% | -50.9% | -32.0% | -25.4% | Reject: slow load replica on `gjfjh` |

Figure 1 plots every observed throughput delta. The alternating red/blue sequence around 2K–8K is the visual signature of a confounded crossover rather than a stable token-cost boundary.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Forced-load request-throughput delta versus recompute","width":720,"height":250,"data":{"values":[{"prefix":"512","concurrency":"16","delta":0.3},{"prefix":"512","concurrency":"32","delta":0.3},{"prefix":"512","concurrency":"64","delta":-4.9},{"prefix":"512","concurrency":"96","delta":-27.6},{"prefix":"1,024","concurrency":"16","delta":5.0},{"prefix":"1,024","concurrency":"32","delta":0.2},{"prefix":"1,024","concurrency":"64","delta":-19.7},{"prefix":"1,024","concurrency":"96","delta":-36.8},{"prefix":"2,048","concurrency":"16","delta":-33.3},{"prefix":"2,048","concurrency":"32","delta":-37.1},{"prefix":"2,048","concurrency":"64","delta":-38.5},{"prefix":"2,048","concurrency":"96","delta":-52.2},{"prefix":"4,096","concurrency":"16","delta":70.0},{"prefix":"4,096","concurrency":"32","delta":52.3},{"prefix":"4,096","concurrency":"64","delta":22.6},{"prefix":"4,096","concurrency":"96","delta":6.4},{"prefix":"8,192","concurrency":"16","delta":-36.4},{"prefix":"8,192","concurrency":"32","delta":-28.3},{"prefix":"8,192","concurrency":"64","delta":-25.9},{"prefix":"8,192","concurrency":"96","delta":-20.7},{"prefix":"16,384","concurrency":"16","delta":-14.5},{"prefix":"16,384","concurrency":"32","delta":-50.9},{"prefix":"16,384","concurrency":"64","delta":-32.0},{"prefix":"16,384","concurrency":"96","delta":-25.4}]},"mark":"rect","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","2,048","4,096","8,192","16,384"],"title":"Externally reusable prefix (tokens)"},"y":{"field":"concurrency","type":"ordinal","sort":["16","32","64","96"],"title":"Concurrent streams"},"color":{"field":"delta","type":"quantitative","title":"Load delta (%)","scale":{"scheme":"redblue","domain":[-70,70]}},"tooltip":[{"field":"prefix","type":"nominal","title":"Reusable prefix (tokens)"},{"field":"concurrency","type":"nominal","title":"Concurrent streams"},{"field":"delta","type":"quantitative","title":"Load delta (%)","format":"+.1f"}]}}
```

Figure 2 filters out the three C96 policy pairs confounded by the slow `gjfjh` replica and keeps only balanced comparisons. Recompute completed 38.1% more requests than load at 512 tokens, 58.2% more at 1,024 tokens, and 26.1% more at 8,192 tokens. This is the strongest result from the batch: under a saturated restore path, loading externally reusable KV was slower than recomputing it in every balanced C96 comparison, including the longest valid prefix.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Balanced C96 comparisons favor recomputation","width":720,"height":310,"data":{"values":[{"prefix":"512","policy":"Forced load","rps":24.610},{"prefix":"512","policy":"Forced recompute","rps":33.983},{"prefix":"1,024","policy":"Forced load","rps":18.050},{"prefix":"1,024","policy":"Forced recompute","rps":28.550},{"prefix":"8,192","policy":"Forced load","rps":6.177},{"prefix":"8,192","policy":"Forced recompute","rps":7.787}]},"mark":"bar","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","8,192"],"title":"Externally reusable prefix (tokens)"},"xOffset":{"field":"policy"},"y":{"field":"rps","type":"quantitative","title":"Successful requests/s","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"prefix","type":"nominal","title":"Reusable prefix (tokens)"},{"field":"policy","type":"nominal"},{"field":"rps","type":"quantitative","title":"Successful requests/s","format":".3f"}]}}
```

Figures 3 and 4 show the absolute request rates at the lowest and highest tested pressure. At C16, 512–1,024-token loading is effectively neutral while the confounded 4K point appears favorable. At C96, recomputation wins every balanced comparison.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3. Successful request throughput at C16","width":720,"height":300,"data":{"values":[{"prefix":512,"policy":"Forced load","rps":8.18},{"prefix":512,"policy":"Forced recompute","rps":8.157},{"prefix":1024,"policy":"Forced load","rps":8.12},{"prefix":1024,"policy":"Forced recompute","rps":7.73},{"prefix":2048,"policy":"Forced load","rps":4.683},{"prefix":2048,"policy":"Forced recompute","rps":7.027},{"prefix":4096,"policy":"Forced load","rps":6.013},{"prefix":4096,"policy":"Forced recompute","rps":3.537},{"prefix":8192,"policy":"Forced load","rps":2.813},{"prefix":8192,"policy":"Forced recompute","rps":4.427},{"prefix":16384,"policy":"Forced load","rps":2.307},{"prefix":16384,"policy":"Forced recompute","rps":2.697}]},"mark":{"type":"line","point":true,"strokeWidth":2},"encoding":{"x":{"field":"prefix","type":"quantitative","title":"Externally reusable prefix (tokens)","scale":{"type":"log","base":2}},"y":{"field":"rps","type":"quantitative","title":"Successful requests/s","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"prefix","type":"quantitative","title":"Prefix tokens"},{"field":"policy","type":"nominal"},{"field":"rps","type":"quantitative","format":".3f","title":"Requests/s"}]}}
```

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4. Successful request throughput at C96","width":720,"height":300,"data":{"values":[{"prefix":512,"policy":"Forced load","rps":24.61},{"prefix":512,"policy":"Forced recompute","rps":33.983},{"prefix":1024,"policy":"Forced load","rps":18.05},{"prefix":1024,"policy":"Forced recompute","rps":28.55},{"prefix":2048,"policy":"Forced load","rps":9.91},{"prefix":2048,"policy":"Forced recompute","rps":20.733},{"prefix":4096,"policy":"Forced load","rps":8.353},{"prefix":4096,"policy":"Forced recompute","rps":7.853},{"prefix":8192,"policy":"Forced load","rps":6.177},{"prefix":8192,"policy":"Forced recompute","rps":7.787},{"prefix":16384,"policy":"Forced load","rps":2.78},{"prefix":16384,"policy":"Forced recompute","rps":3.727}]},"mark":{"type":"line","point":true,"strokeWidth":2},"encoding":{"x":{"field":"prefix","type":"quantitative","title":"Externally reusable prefix (tokens)","scale":{"type":"log","base":2}},"y":{"field":"rps","type":"quantitative","title":"Successful requests/s","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"prefix","type":"quantitative","title":"Prefix tokens"},{"field":"policy","type":"nominal"},{"field":"rps","type":"quantitative","format":".3f","title":"Requests/s"}]}}
```

At C96, loading also produced much higher mean TTFT in every comparison except the confounded 2K point. Figure 5 makes the latency cost of the restore queue visible; the 4K arm's slightly higher throughput still came with 5.60 seconds mean TTFT versus 0.76 seconds in its recompute control.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 5. Mean TTFT at C96","width":720,"height":310,"data":{"values":[{"prefix":"512","policy":"Forced load","ttft":1348},{"prefix":"512","policy":"Forced recompute","ttft":132},{"prefix":"1,024","policy":"Forced load","ttft":2402},{"prefix":"1,024","policy":"Forced recompute","ttft":161},{"prefix":"2,048","policy":"Forced load","ttft":999},{"prefix":"2,048","policy":"Forced recompute","ttft":272},{"prefix":"4,096","policy":"Forced load","ttft":5597},{"prefix":"4,096","policy":"Forced recompute","ttft":757},{"prefix":"8,192","policy":"Forced load","ttft":8278},{"prefix":"8,192","policy":"Forced recompute","ttft":1059},{"prefix":"16,384","policy":"Forced load","ttft":14249},{"prefix":"16,384","policy":"Forced recompute","ttft":5480}]},"mark":"bar","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","2,048","4,096","8,192","16,384"],"title":"Externally reusable prefix (tokens)"},"xOffset":{"field":"policy"},"y":{"field":"ttft","type":"quantitative","title":"Mean TTFT (ms)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"prefix","type":"nominal"},{"field":"policy","type":"nominal"},{"field":"ttft","type":"quantitative","title":"Mean TTFT (ms)","format":",.0f"}]}}
```

## Mechanism evidence

Figure 6 shows the C96 prompt-token source composition for forced-load cells. External reuse rose with prefix size but remained modest; even at 16K, only 11.2% of prompt tokens were restored externally, while 63.2% were recomputed. This limited the available compute saving while every external lookup still entered the shared restore path.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 6. Prompt-token source composition under forced load at C96","width":720,"height":310,"data":{"values":[{"prefix":"512","source":"External KV","percent":4.10},{"prefix":"512","source":"HBM hit","percent":2.55},{"prefix":"512","source":"Compute","percent":93.36},{"prefix":"1,024","source":"External KV","percent":5.37},{"prefix":"1,024","source":"HBM hit","percent":15.78},{"prefix":"1,024","source":"Compute","percent":78.85},{"prefix":"2,048","source":"External KV","percent":7.85},{"prefix":"2,048","source":"HBM hit","percent":19.96},{"prefix":"2,048","source":"Compute","percent":72.19},{"prefix":"4,096","source":"External KV","percent":10.47},{"prefix":"4,096","source":"HBM hit","percent":24.31},{"prefix":"4,096","source":"Compute","percent":65.21},{"prefix":"8,192","source":"External KV","percent":8.56},{"prefix":"8,192","source":"HBM hit","percent":25.98},{"prefix":"8,192","source":"Compute","percent":65.46},{"prefix":"16,384","source":"External KV","percent":11.17},{"prefix":"16,384","source":"HBM hit","percent":25.62},{"prefix":"16,384","source":"Compute","percent":63.21}]},"mark":"bar","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","2,048","4,096","8,192","16,384"],"title":"Externally reusable prefix (tokens)"},"y":{"field":"percent","type":"quantitative","stack":"normalize","title":"Prompt-token share (%)","axis":{"format":"%"}},"color":{"field":"source","type":"nominal","title":"Token source","scale":{"domain":["Compute","HBM hit","External KV"],"range":["#7f7f7f","#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"prefix","type":"nominal"},{"field":"source","type":"nominal"},{"field":"percent","type":"quantitative","title":"Measured share (%)","format":".2f"}]}}
```

Figure 7 relates prefix length to mean asynchronous lookup time at C96. The balanced 4K and 8K arms already spent 4.16 and 6.55 seconds in asynchronous lookup; the 16K value reached 12.8 seconds but is also affected by the slow replica. CPU-to-GPU bandwidth increased from 0.37 GiB/s at 512 tokens to 2.29 GiB/s at 16K, so raw copy bandwidth was not the useful decision metric—the queued lookup/promotion path dominated.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 7. Mean asynchronous KV lookup time under forced load at C96","width":720,"height":300,"data":{"values":[{"prefix":512,"seconds":0.532},{"prefix":1024,"seconds":1.314},{"prefix":2048,"seconds":0.305},{"prefix":4096,"seconds":4.158},{"prefix":8192,"seconds":6.551},{"prefix":16384,"seconds":12.800}]},"mark":{"type":"line","point":true,"strokeWidth":2,"color":"#ff7f0e"},"encoding":{"x":{"field":"prefix","type":"quantitative","title":"Externally reusable prefix (tokens)","scale":{"type":"log","base":2}},"y":{"field":"seconds","type":"quantitative","title":"Mean async lookup time (s)","scale":{"zero":true}},"tooltip":[{"field":"prefix","type":"quantitative","title":"Prefix tokens"},{"field":"seconds","type":"quantitative","title":"Async lookup (s)","format":".3f"}]}}
```

Figure 8 shows the corresponding blocked-loading estimate. The 4K and 8K C96 cells averaged approximately 64–67 blocked requests, aligned with mean waiting queues of 97 and 96 requests respectively. The 2K discontinuity is another symptom of its abnormal replica/node behavior rather than evidence of a cheaper 2K restore.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 8. Blocked KV-loading work under forced load at C96","width":720,"height":300,"data":{"values":[{"prefix":512,"blocked":24.90},{"prefix":1024,"blocked":44.15},{"prefix":2048,"blocked":5.40},{"prefix":4096,"blocked":63.74},{"prefix":8192,"blocked":67.25},{"prefix":16384,"blocked":49.59}]},"mark":{"type":"line","point":true,"strokeWidth":2,"color":"#d62728"},"encoding":{"x":{"field":"prefix","type":"quantitative","title":"Externally reusable prefix (tokens)","scale":{"type":"log","base":2}},"y":{"field":"blocked","type":"quantitative","title":"Mean blocked requests (requests)","scale":{"zero":true}},"tooltip":[{"field":"prefix","type":"quantitative","title":"Prefix tokens"},{"field":"blocked","type":"quantitative","title":"Blocked requests","format":".2f"}]}}
```

Figure 9 shows workload-node NVMe busy time at C96 after filtering out the node that hosted the EPP. It reached 92–95% even for 512–2,048-token cells and effectively 100% at 4K and above. This is a five-minute rolling device metric and therefore includes activity from the preceding stage, but it establishes that the secondary tier was not under-lightly-loaded conditions.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 9. Workload-node NVMe busy time under forced load at C96","width":720,"height":300,"data":{"values":[{"prefix":"512","percent":92.1},{"prefix":"1,024","percent":95.4},{"prefix":"2,048","percent":92.0},{"prefix":"4,096","percent":100.0},{"prefix":"8,192","percent":100.0},{"prefix":"16,384","percent":100.0}]},"mark":{"type":"bar","color":"#9467bd"},"encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","2,048","4,096","8,192","16,384"],"title":"Externally reusable prefix (tokens)"},"y":{"field":"percent","type":"quantitative","title":"NVMe busy time (%)","scale":{"domain":[0,100]}},"tooltip":[{"field":"prefix","type":"nominal"},{"field":"percent","type":"quantitative","title":"NVMe busy (%)","format":".1f"}]}}
```

## Validity and failure evidence

The nodes have the same declared 8×H100 configuration, but equivalent configuration did not produce equivalent realized performance. Figure 10 shows the coefficient of variation of per-pod request rate at C96. The three bars above 30% correspond exactly to treatments placed on `gjfjh`: forced load at 2K, forced recompute at 4K, and forced load at 16K.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 10. Per-replica request-rate imbalance at C96","width":720,"height":310,"data":{"values":[{"prefix":"512","policy":"Forced load","cv":3.5},{"prefix":"512","policy":"Forced recompute","cv":0.8},{"prefix":"1,024","policy":"Forced load","cv":2.3},{"prefix":"1,024","policy":"Forced recompute","cv":1.3},{"prefix":"2,048","policy":"Forced load","cv":33.5},{"prefix":"2,048","policy":"Forced recompute","cv":1.3},{"prefix":"4,096","policy":"Forced load","cv":10.9},{"prefix":"4,096","policy":"Forced recompute","cv":38.3},{"prefix":"8,192","policy":"Forced load","cv":5.4},{"prefix":"8,192","policy":"Forced recompute","cv":1.2},{"prefix":"16,384","policy":"Forced load","cv":42.6},{"prefix":"16,384","policy":"Forced recompute","cv":1.5}]},"mark":"bar","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["512","1,024","2,048","4,096","8,192","16,384"],"title":"Externally reusable prefix (tokens)"},"xOffset":{"field":"policy"},"y":{"field":"cv","type":"quantitative","title":"Per-pod request-rate CV (%)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"tooltip":[{"field":"prefix","type":"nominal"},{"field":"policy","type":"nominal"},{"field":"cv","type":"quantitative","title":"Coefficient of variation (%)","format":".1f"}]}}
```

At C96, the slow 2K-load replica completed about 2.02 req/s while its peers completed 5.70–5.80 req/s. The slow 4K-recompute replica completed 1.16 req/s while its peers completed 4.09–4.36 req/s. The slow 16K-load replica completed 0.31 req/s while its peers completed 1.45–1.49 req/s. The affected pod identity changed between deployments, suggesting a persistent GPU-pair, NUMA, PCIe, CPU-memory, or local-device locality problem rather than one broken Kubernetes pod. This inference is not a root-cause diagnosis because GPU utilization and PCIe telemetry were empty.

All twelve runs had `FINISHED` status, all model pods showed zero restarts, and no sustained request error condition appeared. A small number of C16 startup-stage errors occurred in several arms: 18 total in forced-load runs and 30 total in forced-recompute runs across the entire batch. Each fixed-duration stage ended with roughly one concurrency window of incomplete in-flight requests; these are deadline cutoffs rather than server failures, but they make tail completion share unsuitable as a policy-ranking metric.

Additional limitations:

- The forced policies for a given prefix did not run on the same node. The matrix rotated treatments over `6kl5z`, `fx7c8`, `gjfjh`, and `mt46x`, but it did not create a same-node pair.
- Four cells ran concurrently in each wave on disjoint model nodes. Local NVMe was isolated by node, while network and cluster services remained shared.
- The prewarm ran immediately before measurement at rate 96. Background offload writes may have carried into C16, especially for large prefixes.
- The concurrency order was always 16 → 32 → 64 → 96, so cache state, page cache, storage backlog, and thermal state are inseparable from load level.
- Prompt-source, transfer, and device metrics were collected at 15-second cadence using five-minute `rate()` windows. Because each stage also lasted five minutes, reported stage means contain neighboring-phase history.
- GPU utilization, GPU memory, and PCIe telemetry were empty. CPU and vLLM queue metrics were present. No request-level join connects an EPP decision to the eventual vLLM token source.

## Run registry

Measured windows are UTC on 2026-09-09. All runs used GuideLLM 0.7.3 and finished successfully.

| Prefix | Policy | Workload node | Measured window | MLflow run |
|---:|---|---|---|---|
| 512 | Forced load | `6kl5z` | 10:52:54–11:14:09 | [5f7b650cf85b4ceab01cfa3878ef3f0d](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/408/runs/5f7b650cf85b4ceab01cfa3878ef3f0d?workspace=benchflow) |
| 512 | Forced recompute | `mt46x` | 11:39:21–12:00:37 | [c1affdf3bc4e431f8dd668577ab78212](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/408/runs/c1affdf3bc4e431f8dd668577ab78212?workspace=benchflow) |
| 1,024 | Forced load | `fx7c8` | 10:50:58–11:12:11 | [f2140ae669bb449ebd24ed03aa7a5c80](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/409/runs/f2140ae669bb449ebd24ed03aa7a5c80?workspace=benchflow) |
| 1,024 | Forced recompute | `6kl5z` | 11:38:29–11:59:44 | [7287712fe61a43cd8a868d83f22d584d](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/409/runs/7287712fe61a43cd8a868d83f22d584d?workspace=benchflow) |
| 2,048 | Forced load | `gjfjh` | 10:51:57–11:13:06 | [fa36e5c09a1240c78285bd2ddd98d5ae](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/410/runs/fa36e5c09a1240c78285bd2ddd98d5ae?workspace=benchflow) |
| 2,048 | Forced recompute | `fx7c8` | 12:20:36–12:41:46 | [bf622254ce954ae99dd1cf889ef0d524](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/410/runs/bf622254ce954ae99dd1cf889ef0d524?workspace=benchflow) |
| 4,096 | Forced load | `mt46x` | 10:50:49–11:11:57 | [063db1199c054b4f8bf92301f47c274d](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/411/runs/063db1199c054b4f8bf92301f47c274d?workspace=benchflow) |
| 4,096 | Forced recompute | `gjfjh` | 12:18:54–12:39:59 | [f7276862bc574357a1bd193b240896ac](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/411/runs/f7276862bc574357a1bd193b240896ac?workspace=benchflow) |
| 8,192 | Forced load | `fx7c8` | 11:35:17–11:56:26 | [7c7c3043f80841a8afddf310b94727df](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/412/runs/7c7c3043f80841a8afddf310b94727df?workspace=benchflow) |
| 8,192 | Forced recompute | `6kl5z` | 12:24:10–12:45:16 | [6df507e1623f4cbcb35d9acfcfea3b93](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/412/runs/6df507e1623f4cbcb35d9acfcfea3b93?workspace=benchflow) |
| 16,384 | Forced load | `gjfjh` | 11:35:52–11:56:57 | [70ffbb7450ee48038aad869b05c3c29c](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/413/runs/70ffbb7450ee48038aad869b05c3c29c?workspace=benchflow) |
| 16,384 | Forced recompute | `mt46x` | 12:23:01–12:44:07 | [28d0f81a246e48848d531260c21ac53a](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/413/runs/28d0f81a246e48848d531260c21ac53a?workspace=benchflow) |

## Conclusion and next experiment

For this deployment and 4,194,304-token corpus, the data do not show a prefix length where restore reliably beats recomputation. At C64–C96 the balanced evidence instead shows that entering the external restore path can be harmful even at 8,192 reusable tokens because queued lookup/promotion time overwhelms saved prefill. The current 1,024-token threshold is therefore too permissive for this pressure regime, but choosing a larger finite threshold from this batch would be unjustified.

The smallest experiment that resolves the uncertainty is:

1. Exclude `gjfjh` with node affinity until it passes a separate replica-balance diagnostic. Pin every calibration treatment to one known-balanced node such as `mt46x` and run paired policies sequentially on that same node.
2. Test 1K, 2K, 4K, and 8K prefixes at C16, C32, and C64.
3. Reduce the corpus to 2,097,152 prefix tokens—still intended to exceed aggregate HBM KV capacity while reducing immediate NVMe spill—and lower prewarm rate to 16 or 32.
4. Run every prefix/concurrency/policy cell once; prior GuideLLM validation established sufficient within-run significance for this environment.
5. Accept a point only when external prompt-token share is at least 20%, per-pod request-rate CV is at most 10%, no replica restarts, and the restore queue is stable rather than monotonically growing.
6. Collect counter deltas at stage boundaries or use a shorter Prometheus rate window so adjacent concurrency stages do not bleed together.

If that repeat establishes, for example, that 4K loading wins while 512–1K loading loses, the showcase should use a 50/50 512/4K workload and compare always load, always recompute, and the selective plugin at the calibrated threshold. If the preferred action still changes with concurrency, the longer-term rule should be pressure-aware:

$$
\text{load iff }T_{lookup+queue+promotion}(L,P)<L\,t_{prefill}(C),
$$

where $P$ represents current promotion-path pressure. That would retain reusable-prefix length as the value signal while allowing queued restore cost to veto loading under secondary-tier saturation.

## Provenance

Data were retrieved with the MLflow CLI from experiments 408–413. The analysis used all 48 native GuideLLM benchmark summaries, resolved run plans, pod-placement snapshots, and full-resolution Prometheus artifacts. Request outcomes are exact per five-minute GuideLLM stage. Mechanism metrics are means of the available 15-second samples within each stage; rate-based metrics use five-minute Prometheus windows and are explicitly treated as approximate. NVMe busy time was filtered to the model workload node because the resolved query also returned the node hosting the EPP. No samples were downsampled for analysis; figures contain the exact categorical point summaries described above.

Related records: [[2026-09-09 - Qwen3-32B selective loading bimodal comparison]], [[../01 - Calibration Protocol|KV Cache Offloading calibration protocol]], [[../03 - KV Transfer Metrics and PromQL|KV transfer metrics and PromQL]], and [[Engineering/Projects/Selective KV Loading and Offloading/07 - Selective loading calibration test plan|Selective loading calibration test plan]].