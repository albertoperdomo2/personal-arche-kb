---
title: "Qwen3-32B focused selective-loading crossover calibration"
date: "2026-09-10"
type: "experiment-report"
topic: "KV Cache Offloading"
experiment: "llm-d-selective-loading-focused-crossover; MLflow experiments 414-425"
model: "Qwen/Qwen3-32B"
vllm_version: "0.27.0 selective-load patch"
runtime_image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v1"
scheduler_image: "quay.io/rh-ee-aperdomo/llm-d-router-endpoint-picker:dev-0a55d5da"
tensor_parallelism: 2
replicas: 4
accelerator: "8x H100 on one pinned node"
node: "diadochos-hqxzk-gpu-h100-mt46x"
gpu_memory_utilization: "vLLM default; not explicitly set"
max_model_len: 32768
max_num_seqs: "vLLM default; not explicitly set"
concurrency: [16, 32, 64]
cpu_bytes_per_replica: 274877906944
offload_spec: "TieringOffloadingSpec"
secondary_tier: "filesystem on local NVMe"
secondary_tier_threads: {read: 64, write: 64}
shared_memory: "300Gi per replica"
block_size_tokens: 64
workload: "GuideLLM single-turn fixed-prefix-corpus focused crossover"
random_seed: 20260909
stage_duration_seconds: 300
prewarm_rate: 32
prefix_corpus_tokens: 2097152
cache_cleaning: "cleanup: true on hostPath between deployment cells; each cell prewarmed"
status: "conditionally-valid-pressure-boundary-established"
---

# Qwen3-32B focused selective-loading crossover calibration

This is an ongoing investigation into when llm-d should allow vLLM to restore reusable KV from CPU/NVMe and when it should opt out and recompute. The immediate goal is a defensible static policy for this exact Qwen3-32B deployment; the longer-term goal is a dynamic gate that combines the value of reuse with current restore-path pressure. These measurements are calibration evidence, not a universal threshold.

## Executive summary

This focused matrix repeated the earlier crossover study on one known-balanced 8×H100 node. It compared forced external loading with forced recomputation for nominal reusable prefixes of 1,024, 2,048, 4,096, and 8,192 tokens at concurrencies 16, 32, and 64. The deployment used four TP2 Qwen3-32B replicas, a 256 GiB CPU tier per replica, and a local-NVMe secondary tier.

The result is clean but it is **not a single minimum-prefix crossover**. At C16, loading won all four throughput comparisons by 2.9–8.5%. At C32, it won from 1K through 4K by 7.2–10.7% but lost at 8K by 15.8%. At C64, it won only at 1K (+9.7%) and lost at 2K, 4K, and 8K by 7.0%, 16.2%, and 21.5%. The harmful cells also developed large deferred-lookup and scheduler queues: mean deferred lookup occupancy reached 6.39, 10.63, and 12.97 request-equivalents at C64 for 2K, 4K, and 8K, while mean TTFT grew to 0.82, 1.64, and 2.99 seconds.

The main decision is therefore to **not select a new `minExternalReusableTokens` value from this matrix**. The data support adding or prioritizing a pressure veto. A simple minimum-token threshold cannot express the observed C64 preference—load the 1K case but reject larger, costlier restores—and the nominal benchmark prefix was not always externally restored. The most promising measured pressure indicators were native waiting requests and derived deferred-lookup occupancy; across these twelve points their descriptive Pearson correlations with loading throughput delta were -0.91 and -0.90 respectively. This is strong calibration evidence, not causal proof.

## Validity verdict: conditionally valid

**Valid for the forced-load versus forced-recompute comparison and for establishing a pressure-dependent boundary; conditionally valid for request-level threshold calibration.** All 24 runs finished, every pair used the same pinned node and workload shape, per-pod successful-request-rate CV stayed between 0.4% and 4.4%, and failures were negligible. The result fixes the earlier `gjfjh` imbalance.

The calibration limitation is semantic: each GuideLLM `prefix_tokens` value is a nominal shared prefix, not a measurement that all those tokens came from CPU/NVMe. Under forced load, only 8.9–32.1% of prompt tokens were actually sourced externally; HBM hits and recomputation supplied the rest. The artifacts do not contain a request-level distribution of the EPP's `externalReusableTokens` evidence. Therefore the matrix identifies when the aggregate loading path helps or hurts, but it cannot map a nominal 1K/2K/4K/8K bucket directly to an exact router threshold.

## Main takeaways

- **Measured:** Loading improved successful request throughput in 8 of 12 cells. Its four losses appeared only at higher pressure: 8K/C32 and 2K–8K/C64.
- **Measured:** At C64, loading 1K remained throughput-positive (+9.7%), but mean TTFT was already 13.5% worse. At 2K and above both throughput and latency favored recomputation; TTFT p99 reached 16.1–56.5 seconds under loading versus 0.75–2.99 seconds under recomputation.
- **Measured:** A throughput win was not always a latency win. Of the eight throughput-positive cells, only four also improved both mean and p99 TTFT. A production gate therefore needs an explicit objective or SLO; this report treats successful request throughput as the primary calibration outcome and uses TTFT as a veto and risk signal.
- **Measured:** The loss boundary aligned with queued restore work. Deferred-lookup occupancy was at most 0.51 request-equivalents in every winning cell and at least 3.45 in every losing cell. This perfect separation is specific to these twelve observations and must be validated before becoming a production threshold.
- **Measured:** Workload-node NVMe busy time reached 72–75% in the losing cells, but similarly reached 73.7% in the winning 4K/C32 cell. NVMe busy percentage alone is therefore not a sufficient gate.
- **Measured:** CPU→GPU traffic reached 0.65–3.08 GiB/s, but its aggregate rate did not separate wins from losses. Queue cost, not byte rate alone, better tracked the outcome.
- **Inference:** The policy needs two signals: external reusable tokens as the potential compute saving, plus current queued lookup/promotion pressure as a veto. A prefix-only minimum threshold is structurally unable to reproduce the observed high-pressure decision surface.
- **Conclusion:** Preserve the existing static token threshold as an experimental value signal, but do not claim it is calibrated by this batch. The next implementation experiment should add one conservative pressure threshold using an EPP-consumable pending-work signal, then compare dynamic gating against both forced controls.

## Experiment design and decision rule

The forced-load deployment left `kv_load_tiers` unchanged. The forced-recompute deployment used the selective-KV plugin with `loadPolicy: disable`, causing EPP to send `kv_load_tiers: []`; offloading remained enabled in both controls. Each benchmark used a 2,097,152-token prefix corpus, one turn, 256 non-prefix prompt tokens, 128 requested output tokens, a 32 request/s prewarm, and one five-minute measurement. Deployment cells ran sequentially with NVMe hostPath cleanup enabled.

Forced recomputation is the baseline. The throughput delta is:

$$
\Delta_{load}(L,C)=100\left(\frac{RPS_{load}(L,C)}{RPS_{recompute}(L,C)}-1\right).
$$

Positive values favor loading. The intended dynamic decision is closer to:

$$
\text{load iff } T_{restore}(L,P) < L\,t_{prefill/token}(C),
$$

where $L$ is the request's externally reusable token count and $P$ represents current lookup, storage, CPU-staging, and CPU→GPU-promotion pressure. This matrix shows that $P$ materially changes the result.

## Headline metrics

Forced recompute is listed first conceptually and is the delta baseline. Output length was fixed at 128 tokens, so successful output-token rate is exactly 128 times the listed successful request rate. `Failed R/L` is failed requests in recompute/load; stage-end incomplete sessions are excluded from throughput in both arms.

| Nominal prefix | Concurrency | Recompute req/s | Load req/s | Load delta | Recompute mean TTFT (ms) | Load mean TTFT (ms) | Recompute TTFT p99 (ms) | Load TTFT p99 (ms) | Failed R/L |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1,024 | C16 | 7.910 | 8.423 | +6.5% | 105.1 | 98.4 | 198.9 | 146.5 | 0/0 |
| 1,024 | C32 | 14.253 | 15.283 | +7.2% | 112.8 | 111.7 | 316.4 | 160.1 | 0/0 |
| 1,024 | C64 | 23.823 | 26.143 | +9.7% | 125.4 | 142.4 | 368.1 | 420.5 | 0/1 |
| 2,048 | C16 | 7.327 | 7.537 | +2.9% | 145.3 | 148.0 | 215.8 | 228.8 | 0/0 |
| 2,048 | C32 | 12.210 | 13.350 | +9.3% | 179.2 | 155.9 | 522.8 | 410.2 | 1/0 |
| 2,048 | C64 | 19.847 | 18.467 | -7.0% | 174.0 | 817.8 | 746.1 | 16060.7 | 0/6 |
| 4,096 | C16 | 6.533 | 6.870 | +5.2% | 210.6 | 200.9 | 405.5 | 374.6 | 0/0 |
| 4,096 | C32 | 10.490 | 11.610 | +10.7% | 231.1 | 237.2 | 590.1 | 1011.4 | 0/0 |
| 4,096 | C64 | 15.150 | 12.693 | -16.2% | 281.3 | 1641.1 | 1338.7 | 35386.9 | 0/0 |
| 8,192 | C16 | 5.293 | 5.743 | +8.5% | 384.8 | 352.7 | 931.6 | 1602.1 | 0/0 |
| 8,192 | C32 | 7.737 | 6.517 | -15.8% | 433.7 | 1310.2 | 1333.4 | 26644.3 | 1/0 |
| 8,192 | C64 | 10.077 | 7.907 | -21.5% | 539.2 | 2992.2 | 2992.6 | 56477.3 | 1/2 |

Figure 1 is the main result. It shows a stable pressure boundary, not the alternating node-confounded pattern from the earlier sweep.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Forced-load request-throughput delta versus recompute","width":720,"height":240,"data":{"values":[{"prefix":"1,024","concurrency":"16","delta":6.49},{"prefix":"1,024","concurrency":"32","delta":7.23},{"prefix":"1,024","concurrency":"64","delta":9.74},{"prefix":"2,048","concurrency":"16","delta":2.87},{"prefix":"2,048","concurrency":"32","delta":9.34},{"prefix":"2,048","concurrency":"64","delta":-6.95},{"prefix":"4,096","concurrency":"16","delta":5.15},{"prefix":"4,096","concurrency":"32","delta":10.68},{"prefix":"4,096","concurrency":"64","delta":-16.22},{"prefix":"8,192","concurrency":"16","delta":8.5},{"prefix":"8,192","concurrency":"32","delta":-15.77},{"prefix":"8,192","concurrency":"64","delta":-21.53}]},"mark":"rect","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["1,024","2,048","4,096","8,192"],"title":"Nominal reusable prefix (tokens)"},"y":{"field":"concurrency","type":"ordinal","sort":["16","32","64"],"title":"Concurrent streams"},"color":{"field":"delta","type":"quantitative","title":"Load delta (%)","scale":{"scheme":"redblue","domain":[-25,25],"domainMid":0}},"tooltip":[{"field":"prefix","type":"nominal","title":"Prefix tokens"},{"field":"concurrency","type":"nominal","title":"Concurrency"},{"field":"delta","type":"quantitative","title":"Load delta (%)","format":"+.2f"}]}}
```

At C16, forced load wins throughout. At C32, the boundary moves to the 8K cell. At C64, only 1K remains positive. That direction is incompatible with treating longer externally reusable prefixes as automatically more valuable without accounting for the cost of restoring them under contention.

Figure 2 provides the absolute rates behind Figure 1. Every pair has identical request shape, making successful requests/s the primary outcome.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Successful request throughput across the focused matrix","width":720,"height":300,"data":{"values":[{"prefix":1024,"concurrency":"C16","policy":"Forced load","rps":8.4233},{"prefix":1024,"concurrency":"C32","policy":"Forced load","rps":15.2833},{"prefix":1024,"concurrency":"C64","policy":"Forced load","rps":26.1433},{"prefix":2048,"concurrency":"C16","policy":"Forced load","rps":7.5367},{"prefix":2048,"concurrency":"C32","policy":"Forced load","rps":13.35},{"prefix":2048,"concurrency":"C64","policy":"Forced load","rps":18.4667},{"prefix":4096,"concurrency":"C16","policy":"Forced load","rps":6.87},{"prefix":4096,"concurrency":"C32","policy":"Forced load","rps":11.61},{"prefix":4096,"concurrency":"C64","policy":"Forced load","rps":12.6933},{"prefix":8192,"concurrency":"C16","policy":"Forced load","rps":5.7433},{"prefix":8192,"concurrency":"C32","policy":"Forced load","rps":6.5167},{"prefix":8192,"concurrency":"C64","policy":"Forced load","rps":7.9067},{"prefix":1024,"concurrency":"C16","policy":"Forced recompute","rps":7.91},{"prefix":1024,"concurrency":"C32","policy":"Forced recompute","rps":14.2533},{"prefix":1024,"concurrency":"C64","policy":"Forced recompute","rps":23.8233},{"prefix":2048,"concurrency":"C16","policy":"Forced recompute","rps":7.3267},{"prefix":2048,"concurrency":"C32","policy":"Forced recompute","rps":12.21},{"prefix":2048,"concurrency":"C64","policy":"Forced recompute","rps":19.8467},{"prefix":4096,"concurrency":"C16","policy":"Forced recompute","rps":6.5333},{"prefix":4096,"concurrency":"C32","policy":"Forced recompute","rps":10.49},{"prefix":4096,"concurrency":"C64","policy":"Forced recompute","rps":15.15},{"prefix":8192,"concurrency":"C16","policy":"Forced recompute","rps":5.2933},{"prefix":8192,"concurrency":"C32","policy":"Forced recompute","rps":7.7367},{"prefix":8192,"concurrency":"C64","policy":"Forced recompute","rps":10.0767}]},"mark":{"type":"line","point":true,"strokeWidth":2},"encoding":{"x":{"field":"prefix","type":"quantitative","title":"Nominal reusable prefix (tokens)","scale":{"type":"log","base":2}},"y":{"field":"rps","type":"quantitative","title":"Successful requests/s","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"strokeDash":{"field":"concurrency","type":"nominal","title":"Concurrency"},"tooltip":[{"field":"prefix","type":"quantitative","title":"Prefix tokens"},{"field":"concurrency","type":"nominal"},{"field":"policy","type":"nominal"},{"field":"rps","type":"quantitative","title":"Requests/s","format":".3f"}]}}
```

Figure 3 shows why throughput alone understates the high-pressure failure. At C64, load TTFT p99 jumps from 0.42 seconds at 1K to 16.1, 35.4, and 56.5 seconds at 2K, 4K, and 8K.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3. TTFT p99 exposes restore-path tail collapse","width":720,"height":300,"data":{"values":[{"prefix":"1,024","concurrency":"C16","policy":"Forced recompute","ttft_p99_ms":198.94},{"prefix":"1,024","concurrency":"C16","policy":"Forced load","ttft_p99_ms":146.53},{"prefix":"1,024","concurrency":"C32","policy":"Forced recompute","ttft_p99_ms":316.36},{"prefix":"1,024","concurrency":"C32","policy":"Forced load","ttft_p99_ms":160.14},{"prefix":"1,024","concurrency":"C64","policy":"Forced recompute","ttft_p99_ms":368.08},{"prefix":"1,024","concurrency":"C64","policy":"Forced load","ttft_p99_ms":420.53},{"prefix":"2,048","concurrency":"C16","policy":"Forced recompute","ttft_p99_ms":215.76},{"prefix":"2,048","concurrency":"C16","policy":"Forced load","ttft_p99_ms":228.79},{"prefix":"2,048","concurrency":"C32","policy":"Forced recompute","ttft_p99_ms":522.82},{"prefix":"2,048","concurrency":"C32","policy":"Forced load","ttft_p99_ms":410.16},{"prefix":"2,048","concurrency":"C64","policy":"Forced recompute","ttft_p99_ms":746.15},{"prefix":"2,048","concurrency":"C64","policy":"Forced load","ttft_p99_ms":16060.66},{"prefix":"4,096","concurrency":"C16","policy":"Forced recompute","ttft_p99_ms":405.45},{"prefix":"4,096","concurrency":"C16","policy":"Forced load","ttft_p99_ms":374.63},{"prefix":"4,096","concurrency":"C32","policy":"Forced recompute","ttft_p99_ms":590.08},{"prefix":"4,096","concurrency":"C32","policy":"Forced load","ttft_p99_ms":1011.39},{"prefix":"4,096","concurrency":"C64","policy":"Forced recompute","ttft_p99_ms":1338.74},{"prefix":"4,096","concurrency":"C64","policy":"Forced load","ttft_p99_ms":35386.9},{"prefix":"8,192","concurrency":"C16","policy":"Forced recompute","ttft_p99_ms":931.63},{"prefix":"8,192","concurrency":"C16","policy":"Forced load","ttft_p99_ms":1602.09},{"prefix":"8,192","concurrency":"C32","policy":"Forced recompute","ttft_p99_ms":1333.37},{"prefix":"8,192","concurrency":"C32","policy":"Forced load","ttft_p99_ms":26644.3},{"prefix":"8,192","concurrency":"C64","policy":"Forced recompute","ttft_p99_ms":2992.56},{"prefix":"8,192","concurrency":"C64","policy":"Forced load","ttft_p99_ms":56477.3}]},"mark":{"type":"line","point":true,"strokeWidth":2},"encoding":{"x":{"field":"prefix","type":"ordinal","sort":["1,024","2,048","4,096","8,192"],"title":"Nominal reusable prefix (tokens)"},"y":{"field":"ttft_p99_ms","type":"quantitative","title":"TTFT p99 (ms)","scale":{"type":"log"}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"domain":["Forced recompute","Forced load"],"range":["#1f77b4","#ff7f0e"]}},"strokeDash":{"field":"concurrency","type":"nominal","title":"Concurrency"},"tooltip":[{"field":"prefix","type":"nominal","title":"Prefix tokens"},{"field":"concurrency","type":"nominal"},{"field":"policy","type":"nominal"},{"field":"ttft_p99_ms","type":"quantitative","title":"TTFT p99 (ms)","format":",.1f"}]}}
```

## Prompt-token source evidence

Figure 4 uses the full prompt-token source breakdown from the 15-second Prometheus artifacts. Values are normalized per cell; no categorical samples were omitted.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4. Prompt-token source composition under forced loading","width":720,"height":300,"data":{"values":[{"cell":"1K/C16","prefix":1024,"concurrency":16,"source":"External KV","percent":32.147},{"cell":"1K/C16","prefix":1024,"concurrency":16,"source":"HBM hit","percent":30.181},{"cell":"1K/C16","prefix":1024,"concurrency":16,"source":"Recomputed","percent":37.672},{"cell":"1K/C32","prefix":1024,"concurrency":32,"source":"External KV","percent":30.523},{"cell":"1K/C32","prefix":1024,"concurrency":32,"source":"HBM hit","percent":31.428},{"cell":"1K/C32","prefix":1024,"concurrency":32,"source":"Recomputed","percent":38.049},{"cell":"1K/C64","prefix":1024,"concurrency":64,"source":"External KV","percent":26.687},{"cell":"1K/C64","prefix":1024,"concurrency":64,"source":"HBM hit","percent":32.532},{"cell":"1K/C64","prefix":1024,"concurrency":64,"source":"Recomputed","percent":40.781},{"cell":"2K/C16","prefix":2048,"concurrency":16,"source":"External KV","percent":8.932},{"cell":"2K/C16","prefix":2048,"concurrency":16,"source":"HBM hit","percent":39.036},{"cell":"2K/C16","prefix":2048,"concurrency":16,"source":"Recomputed","percent":52.031},{"cell":"2K/C32","prefix":2048,"concurrency":32,"source":"External KV","percent":11.334},{"cell":"2K/C32","prefix":2048,"concurrency":32,"source":"HBM hit","percent":40.733},{"cell":"2K/C32","prefix":2048,"concurrency":32,"source":"Recomputed","percent":47.932},{"cell":"2K/C64","prefix":2048,"concurrency":64,"source":"External KV","percent":12.865},{"cell":"2K/C64","prefix":2048,"concurrency":64,"source":"HBM hit","percent":41.813},{"cell":"2K/C64","prefix":2048,"concurrency":64,"source":"Recomputed","percent":45.322},{"cell":"4K/C16","prefix":4096,"concurrency":16,"source":"External KV","percent":8.943},{"cell":"4K/C16","prefix":4096,"concurrency":16,"source":"HBM hit","percent":48.061},{"cell":"4K/C16","prefix":4096,"concurrency":16,"source":"Recomputed","percent":42.995},{"cell":"4K/C32","prefix":4096,"concurrency":32,"source":"External KV","percent":10.886},{"cell":"4K/C32","prefix":4096,"concurrency":32,"source":"HBM hit","percent":49.418},{"cell":"4K/C32","prefix":4096,"concurrency":32,"source":"Recomputed","percent":39.696},{"cell":"4K/C64","prefix":4096,"concurrency":64,"source":"External KV","percent":12.712},{"cell":"4K/C64","prefix":4096,"concurrency":64,"source":"HBM hit","percent":50.165},{"cell":"4K/C64","prefix":4096,"concurrency":64,"source":"Recomputed","percent":37.123},{"cell":"8K/C16","prefix":8192,"concurrency":16,"source":"External KV","percent":9.5},{"cell":"8K/C16","prefix":8192,"concurrency":16,"source":"HBM hit","percent":51.691},{"cell":"8K/C16","prefix":8192,"concurrency":16,"source":"Recomputed","percent":38.809},{"cell":"8K/C32","prefix":8192,"concurrency":32,"source":"External KV","percent":10.062},{"cell":"8K/C32","prefix":8192,"concurrency":32,"source":"HBM hit","percent":54.041},{"cell":"8K/C32","prefix":8192,"concurrency":32,"source":"Recomputed","percent":35.897},{"cell":"8K/C64","prefix":8192,"concurrency":64,"source":"External KV","percent":13.229},{"cell":"8K/C64","prefix":8192,"concurrency":64,"source":"HBM hit","percent":54.272},{"cell":"8K/C64","prefix":8192,"concurrency":64,"source":"Recomputed","percent":32.499}]},"mark":"bar","encoding":{"x":{"field":"cell","type":"ordinal","sort":["1K/C16","1K/C32","1K/C64","2K/C16","2K/C32","2K/C64","4K/C16","4K/C32","4K/C64","8K/C16","8K/C32","8K/C64"],"title":"Nominal prefix / concurrency","axis":{"labelAngle":-45}},"y":{"field":"percent","type":"quantitative","title":"Prompt tokens by source (%)","stack":"normalize","axis":{"format":"%"}},"color":{"field":"source","type":"nominal","title":"Prompt source","scale":{"scheme":"category10"}},"tooltip":[{"field":"prefix","type":"quantitative","title":"Prefix tokens"},{"field":"concurrency","type":"quantitative"},{"field":"source","type":"nominal"},{"field":"percent","type":"quantitative","title":"Share (%)","format":".2f"}]}}
```

The 1K cells have the highest external share at 26.7–32.1%. Larger nominal prefixes have only 8.9–13.2% external share and progressively more HBM reuse. This confirms that nominal GuideLLM prefix length is not the same as the EPP's request-level externally reusable token count. Any precise `minExternalReusableTokens` calibration needs decision/source telemetry at request level or at least a histogram of external tokens on admitted requests.

## Restore-path mechanism evidence

“Deferred KV-lookup occupancy” is derived from `sum(rate(vllm:kv_offload_lookup_async_delay_seconds_sum[5m]))`. Its unit is request-seconds per second, interpreted as average concurrent request-equivalents whose asynchronous lookup had deferred; it is not a native count of unique blocked requests. `vllm:num_requests_waiting` is a separate native scheduler gauge. Both are reported because they capture different parts of the same pressure state.

| Nominal prefix | Concurrency | External prompt share | Mean async lookup (s) | Async lookup p90 (s) | Deferred occupancy (request-equiv.) | Waiting requests | CPU→GPU (GiB/s) | NVMe busy | NVMe read/write (GiB/s) |
|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1,024 | C16 | 32.1% | 0.015 | 0.047 | 0.15 | 0.6 | 1.80 | 27.2% | 0.000/1.044 |
| 1,024 | C32 | 30.5% | 0.016 | 0.047 | 0.22 | 0.5 | 2.42 | 35.0% | 0.000/1.473 |
| 1,024 | C64 | 26.7% | 0.020 | 0.048 | 0.51 | 2.7 | 3.08 | 53.3% | 0.008/2.273 |
| 2,048 | C16 | 8.9% | 0.026 | 0.048 | 0.25 | 0.2 | 0.65 | 49.3% | 0.000/1.966 |
| 2,048 | C32 | 11.3% | 0.026 | 0.047 | 0.37 | 0.3 | 1.21 | 61.3% | 0.005/2.576 |
| 2,048 | C64 | 12.9% | 0.222 | 0.337 | 6.39 | 27.3 | 1.87 | 74.6% | 0.046/3.149 |
| 4,096 | C16 | 8.9% | 0.041 | 0.080 | 0.28 | 0.3 | 0.92 | 57.3% | 0.000/2.311 |
| 4,096 | C32 | 10.9% | 0.039 | 0.066 | 0.46 | 1.1 | 1.69 | 73.7% | 0.048/3.153 |
| 4,096 | C64 | 12.7% | 0.531 | 1.761 | 10.63 | 41.0 | 2.39 | 75.2% | 0.061/3.129 |
| 8,192 | C16 | 9.5% | 0.087 | 0.248 | 0.45 | 0.7 | 1.40 | 69.3% | 0.032/2.922 |
| 8,192 | C32 | 10.1% | 0.327 | 0.454 | 3.45 | 16.9 | 1.87 | 72.4% | 0.062/3.011 |
| 8,192 | C64 | 13.2% | 1.000 | 2.987 | 12.97 | 56.7 | 2.86 | 73.7% | 0.114/2.932 |

Figure 5 makes the separation visible: all winning cells are at or below 0.51 request-equivalents, while all losing cells are at or above 3.45.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 5. Derived deferred KV-lookup occupancy under forced load","width":720,"height":240,"data":{"values":[{"prefix":"1,024","concurrency":"16","occupancy":0.149},{"prefix":"1,024","concurrency":"32","occupancy":0.222},{"prefix":"1,024","concurrency":"64","occupancy":0.505},{"prefix":"2,048","concurrency":"16","occupancy":0.25},{"prefix":"2,048","concurrency":"32","occupancy":0.373},{"prefix":"2,048","concurrency":"64","occupancy":6.393},{"prefix":"4,096","concurrency":"16","occupancy":0.276},{"prefix":"4,096","concurrency":"32","occupancy":0.458},{"prefix":"4,096","concurrency":"64","occupancy":10.629},{"prefix":"8,192","concurrency":"16","occupancy":0.446},{"prefix":"8,192","concurrency":"32","occupancy":3.447},{"prefix":"8,192","concurrency":"64","occupancy":12.97}]},"mark":"rect","encoding":{"x":{"field":"prefix","type":"ordinal","sort":["1,024","2,048","4,096","8,192"],"title":"Nominal reusable prefix (tokens)"},"y":{"field":"concurrency","type":"ordinal","sort":["16","32","64"],"title":"Concurrent streams"},"color":{"field":"occupancy","type":"quantitative","title":"Request-equivalents","scale":{"scheme":"oranges"}},"tooltip":[{"field":"prefix","type":"nominal","title":"Prefix tokens"},{"field":"concurrency","type":"nominal"},{"field":"occupancy","type":"quantitative","title":"Deferred occupancy","format":".3f"}]}}
```

Figure 6 directly relates the pressure estimate to the end-to-end result. Across twelve observations, the descriptive Pearson correlation is -0.90. Concurrency and prefix affect both axes, so this chart supports a gate hypothesis but does not establish causality or a universal cutoff.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 6. Loading loses as deferred lookup work accumulates","width":720,"height":300,"data":{"values":[{"prefix":"1,024","concurrency":"C16","occupancy":0.1489,"delta":6.49,"waiting":0.6},{"prefix":"1,024","concurrency":"C32","occupancy":0.2223,"delta":7.226,"waiting":0.45},{"prefix":"1,024","concurrency":"C64","occupancy":0.5054,"delta":9.738,"waiting":2.7},{"prefix":"2,048","concurrency":"C16","occupancy":0.2503,"delta":2.866,"waiting":0.2},{"prefix":"2,048","concurrency":"C32","occupancy":0.3734,"delta":9.337,"waiting":0.35},{"prefix":"2,048","concurrency":"C64","occupancy":6.393,"delta":-6.953,"waiting":27.3},{"prefix":"4,096","concurrency":"C16","occupancy":0.2756,"delta":5.153,"waiting":0.35},{"prefix":"4,096","concurrency":"C32","occupancy":0.4583,"delta":10.677,"waiting":1.05},{"prefix":"4,096","concurrency":"C64","occupancy":10.6294,"delta":-16.216,"waiting":40.95},{"prefix":"8,192","concurrency":"C16","occupancy":0.4457,"delta":8.501,"waiting":0.7},{"prefix":"8,192","concurrency":"C32","occupancy":3.4473,"delta":-15.769,"waiting":16.9},{"prefix":"8,192","concurrency":"C64","occupancy":12.9702,"delta":-21.535,"waiting":56.7}]},"mark":{"type":"point","filled":true,"size":110},"encoding":{"x":{"field":"occupancy","type":"quantitative","title":"Deferred KV-lookup occupancy (request-equivalents)"},"y":{"field":"delta","type":"quantitative","title":"Load throughput delta vs recompute (%)"},"color":{"field":"concurrency","type":"nominal","title":"Concurrency","scale":{"scheme":"category10"}},"shape":{"field":"prefix","type":"nominal","title":"Prefix tokens"},"tooltip":[{"field":"prefix","type":"nominal","title":"Prefix tokens"},{"field":"concurrency","type":"nominal"},{"field":"occupancy","type":"quantitative","title":"Deferred occupancy","format":".3f"},{"field":"waiting","type":"quantitative","title":"Waiting requests","format":".2f"},{"field":"delta","type":"quantitative","title":"Load delta (%)","format":"+.2f"}]}}
```

Figures 7 and 8 retain every available 15-second sample from the four C64 forced-load runs. Each run is aligned to its own measurement-window start; no smoothing or downsampling was applied. The Prometheus histogram/rate queries themselves use five-minute windows, so early samples include some prewarm history.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 7. C64 asynchronous lookup p90 at native 15-second cadence","width":720,"height":300,"data":{"values":[{"elapsed_s":0,"prefix":"1K","seconds":0.048169},{"elapsed_s":15,"prefix":"1K","seconds":0.048148},{"elapsed_s":30,"prefix":"1K","seconds":0.048143},{"elapsed_s":45,"prefix":"1K","seconds":0.048137},{"elapsed_s":60,"prefix":"1K","seconds":0.04805},{"elapsed_s":75,"prefix":"1K","seconds":0.04815},{"elapsed_s":90,"prefix":"1K","seconds":0.048015},{"elapsed_s":105,"prefix":"1K","seconds":0.047737},{"elapsed_s":120,"prefix":"1K","seconds":0.047601},{"elapsed_s":135,"prefix":"1K","seconds":0.047486},{"elapsed_s":150,"prefix":"1K","seconds":0.047603},{"elapsed_s":165,"prefix":"1K","seconds":0.047633},{"elapsed_s":180,"prefix":"1K","seconds":0.047554},{"elapsed_s":195,"prefix":"1K","seconds":0.047554},{"elapsed_s":210,"prefix":"1K","seconds":0.047495},{"elapsed_s":225,"prefix":"1K","seconds":0.047433},{"elapsed_s":240,"prefix":"1K","seconds":0.0474},{"elapsed_s":255,"prefix":"1K","seconds":0.047317},{"elapsed_s":270,"prefix":"1K","seconds":0.047441},{"elapsed_s":285,"prefix":"1K","seconds":0.04758},{"elapsed_s":0,"prefix":"2K","seconds":0.048477},{"elapsed_s":15,"prefix":"2K","seconds":0.048267},{"elapsed_s":30,"prefix":"2K","seconds":0.048102},{"elapsed_s":45,"prefix":"2K","seconds":0.047915},{"elapsed_s":60,"prefix":"2K","seconds":0.047783},{"elapsed_s":75,"prefix":"2K","seconds":0.048426},{"elapsed_s":90,"prefix":"2K","seconds":0.049119},{"elapsed_s":105,"prefix":"2K","seconds":0.082738},{"elapsed_s":120,"prefix":"2K","seconds":0.240598},{"elapsed_s":135,"prefix":"2K","seconds":0.322733},{"elapsed_s":150,"prefix":"2K","seconds":0.403886},{"elapsed_s":165,"prefix":"2K","seconds":0.479739},{"elapsed_s":180,"prefix":"2K","seconds":0.45223},{"elapsed_s":195,"prefix":"2K","seconds":0.457488},{"elapsed_s":210,"prefix":"2K","seconds":0.470297},{"elapsed_s":225,"prefix":"2K","seconds":0.596907},{"elapsed_s":240,"prefix":"2K","seconds":0.653032},{"elapsed_s":255,"prefix":"2K","seconds":0.720295},{"elapsed_s":270,"prefix":"2K","seconds":0.754974},{"elapsed_s":285,"prefix":"2K","seconds":0.771493},{"elapsed_s":0,"prefix":"4K","seconds":0.419412},{"elapsed_s":15,"prefix":"4K","seconds":0.373387},{"elapsed_s":30,"prefix":"4K","seconds":0.27939},{"elapsed_s":45,"prefix":"4K","seconds":0.190033},{"elapsed_s":60,"prefix":"4K","seconds":0.114629},{"elapsed_s":75,"prefix":"4K","seconds":0.256054},{"elapsed_s":90,"prefix":"4K","seconds":0.541949},{"elapsed_s":105,"prefix":"4K","seconds":0.592419},{"elapsed_s":120,"prefix":"4K","seconds":0.854137},{"elapsed_s":135,"prefix":"4K","seconds":2.127902},{"elapsed_s":150,"prefix":"4K","seconds":2.772953},{"elapsed_s":165,"prefix":"4K","seconds":2.719187},{"elapsed_s":180,"prefix":"4K","seconds":2.550492},{"elapsed_s":195,"prefix":"4K","seconds":2.946855},{"elapsed_s":210,"prefix":"4K","seconds":2.7839},{"elapsed_s":225,"prefix":"4K","seconds":2.906152},{"elapsed_s":240,"prefix":"4K","seconds":3.024288},{"elapsed_s":255,"prefix":"4K","seconds":3.208785},{"elapsed_s":270,"prefix":"4K","seconds":3.261851},{"elapsed_s":285,"prefix":"4K","seconds":3.305323},{"elapsed_s":0,"prefix":"8K","seconds":0.67882},{"elapsed_s":15,"prefix":"8K","seconds":0.471961},{"elapsed_s":30,"prefix":"8K","seconds":0.438419},{"elapsed_s":45,"prefix":"8K","seconds":0.244299},{"elapsed_s":60,"prefix":"8K","seconds":0.83488},{"elapsed_s":75,"prefix":"8K","seconds":0.821379},{"elapsed_s":90,"prefix":"8K","seconds":1.256132},{"elapsed_s":105,"prefix":"8K","seconds":1.137512},{"elapsed_s":120,"prefix":"8K","seconds":2.811897},{"elapsed_s":135,"prefix":"8K","seconds":3.557713},{"elapsed_s":150,"prefix":"8K","seconds":3.323853},{"elapsed_s":165,"prefix":"8K","seconds":3.271825},{"elapsed_s":180,"prefix":"8K","seconds":4.075298},{"elapsed_s":195,"prefix":"8K","seconds":4.494219},{"elapsed_s":210,"prefix":"8K","seconds":4.481305},{"elapsed_s":225,"prefix":"8K","seconds":4.593217},{"elapsed_s":240,"prefix":"8K","seconds":5.280254},{"elapsed_s":255,"prefix":"8K","seconds":5.958727},{"elapsed_s":270,"prefix":"8K","seconds":5.993145},{"elapsed_s":285,"prefix":"8K","seconds":6.009277}]},"mark":{"type":"line","strokeWidth":2},"encoding":{"x":{"field":"elapsed_s","type":"quantitative","title":"Elapsed measurement time (s)"},"color":{"field":"prefix","type":"nominal","title":"Nominal prefix","scale":{"scheme":"category10"}},"y":{"field":"seconds","type":"quantitative","title":"Async lookup p90 (s)","scale":{"zero":true}},"tooltip":[{"field":"elapsed_s","type":"quantitative","title":"Elapsed (s)"},{"field":"prefix","type":"nominal"},{"field":"seconds","type":"quantitative","title":"Lookup p90 (s)","format":".3f"}]}}
```

The 2K–8K lookup tails remain elevated and volatile relative to 1K. Figure 8 shows the corresponding scheduler queue buildup.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 8. C64 native vLLM waiting queue at 15-second cadence","width":720,"height":300,"data":{"values":[{"elapsed_s":0,"prefix":"1K","requests":4.0},{"elapsed_s":15,"prefix":"1K","requests":1.0},{"elapsed_s":30,"prefix":"1K","requests":2.0},{"elapsed_s":45,"prefix":"1K","requests":1.0},{"elapsed_s":60,"prefix":"1K","requests":2.0},{"elapsed_s":75,"prefix":"1K","requests":1.0},{"elapsed_s":90,"prefix":"1K","requests":0.0},{"elapsed_s":105,"prefix":"1K","requests":2.0},{"elapsed_s":120,"prefix":"1K","requests":1.0},{"elapsed_s":135,"prefix":"1K","requests":3.0},{"elapsed_s":150,"prefix":"1K","requests":0.0},{"elapsed_s":165,"prefix":"1K","requests":1.0},{"elapsed_s":180,"prefix":"1K","requests":2.0},{"elapsed_s":195,"prefix":"1K","requests":0.0},{"elapsed_s":210,"prefix":"1K","requests":1.0},{"elapsed_s":225,"prefix":"1K","requests":3.0},{"elapsed_s":240,"prefix":"1K","requests":2.0},{"elapsed_s":255,"prefix":"1K","requests":2.0},{"elapsed_s":270,"prefix":"1K","requests":10.0},{"elapsed_s":285,"prefix":"1K","requests":16.0},{"elapsed_s":0,"prefix":"2K","requests":2.0},{"elapsed_s":15,"prefix":"2K","requests":1.0},{"elapsed_s":30,"prefix":"2K","requests":4.0},{"elapsed_s":45,"prefix":"2K","requests":1.0},{"elapsed_s":60,"prefix":"2K","requests":2.0},{"elapsed_s":75,"prefix":"2K","requests":45.0},{"elapsed_s":90,"prefix":"2K","requests":3.0},{"elapsed_s":105,"prefix":"2K","requests":0.0},{"elapsed_s":120,"prefix":"2K","requests":14.0},{"elapsed_s":135,"prefix":"2K","requests":5.0},{"elapsed_s":150,"prefix":"2K","requests":72.0},{"elapsed_s":165,"prefix":"2K","requests":22.0},{"elapsed_s":180,"prefix":"2K","requests":40.0},{"elapsed_s":195,"prefix":"2K","requests":52.0},{"elapsed_s":210,"prefix":"2K","requests":74.0},{"elapsed_s":225,"prefix":"2K","requests":49.0},{"elapsed_s":240,"prefix":"2K","requests":54.0},{"elapsed_s":255,"prefix":"2K","requests":45.0},{"elapsed_s":270,"prefix":"2K","requests":29.0},{"elapsed_s":285,"prefix":"2K","requests":32.0},{"elapsed_s":0,"prefix":"4K","requests":0.0},{"elapsed_s":15,"prefix":"4K","requests":1.0},{"elapsed_s":30,"prefix":"4K","requests":0.0},{"elapsed_s":45,"prefix":"4K","requests":1.0},{"elapsed_s":60,"prefix":"4K","requests":1.0},{"elapsed_s":75,"prefix":"4K","requests":4.0},{"elapsed_s":90,"prefix":"4K","requests":16.0},{"elapsed_s":105,"prefix":"4K","requests":0.0},{"elapsed_s":120,"prefix":"4K","requests":84.0},{"elapsed_s":135,"prefix":"4K","requests":70.0},{"elapsed_s":150,"prefix":"4K","requests":61.0},{"elapsed_s":165,"prefix":"4K","requests":44.0},{"elapsed_s":180,"prefix":"4K","requests":57.0},{"elapsed_s":195,"prefix":"4K","requests":33.0},{"elapsed_s":210,"prefix":"4K","requests":53.0},{"elapsed_s":225,"prefix":"4K","requests":83.0},{"elapsed_s":240,"prefix":"4K","requests":73.0},{"elapsed_s":255,"prefix":"4K","requests":83.0},{"elapsed_s":270,"prefix":"4K","requests":82.0},{"elapsed_s":285,"prefix":"4K","requests":73.0},{"elapsed_s":0,"prefix":"8K","requests":7.0},{"elapsed_s":15,"prefix":"8K","requests":1.0},{"elapsed_s":30,"prefix":"8K","requests":3.0},{"elapsed_s":45,"prefix":"8K","requests":1.0},{"elapsed_s":60,"prefix":"8K","requests":7.0},{"elapsed_s":75,"prefix":"8K","requests":2.0},{"elapsed_s":90,"prefix":"8K","requests":29.0},{"elapsed_s":105,"prefix":"8K","requests":91.0},{"elapsed_s":120,"prefix":"8K","requests":25.0},{"elapsed_s":135,"prefix":"8K","requests":34.0},{"elapsed_s":150,"prefix":"8K","requests":53.0},{"elapsed_s":165,"prefix":"8K","requests":109.0},{"elapsed_s":180,"prefix":"8K","requests":88.0},{"elapsed_s":195,"prefix":"8K","requests":87.0},{"elapsed_s":210,"prefix":"8K","requests":101.0},{"elapsed_s":225,"prefix":"8K","requests":104.0},{"elapsed_s":240,"prefix":"8K","requests":108.0},{"elapsed_s":255,"prefix":"8K","requests":103.0},{"elapsed_s":270,"prefix":"8K","requests":84.0},{"elapsed_s":285,"prefix":"8K","requests":97.0}]},"mark":{"type":"line","strokeWidth":2},"encoding":{"x":{"field":"elapsed_s","type":"quantitative","title":"Elapsed measurement time (s)"},"color":{"field":"prefix","type":"nominal","title":"Nominal prefix","scale":{"scheme":"category10"}},"y":{"field":"requests","type":"quantitative","title":"Waiting requests","scale":{"zero":true}},"tooltip":[{"field":"elapsed_s","type":"quantitative","title":"Elapsed (s)"},{"field":"prefix","type":"nominal"},{"field":"requests","type":"quantitative","title":"Waiting requests","format":".1f"}]}}
```

The queue remains near zero for the winning 1K cell and grows sharply for the losing larger-prefix cells. Together with Figure 7, this provides within-run evidence that the aggregate losses are associated with persistent restore-path pressure rather than a transient outlier.

## Validity and failure evidence

- All 24 MLflow runs finished.
- Client failures totaled 9 under forced load and 3 under forced recomputation across 42,159 and 42,201 successful requests respectively; the largest single-cell load failure count was six at 2K/C64. This is too small to explain the throughput deltas.
- Stage-end incomplete sessions ranged from 38 to 113 for load and 36 to 113 for recompute, broadly following configured concurrency and the fixed stop time. They are not interpreted as request failures.
- Per-pod successful-request-rate CV was 0.5–4.4% for load and 0.4–4.0% for recompute, below the predeclared 10% balance limit. The earlier slow-replica confound is absent.
- GPU utilization and PCIe telemetry files were empty. CPU→GPU transfer comes from the vLLM KV-offload counters, not DCGM. CPU-cache utilization remained below 1.2%, but this metric measures cache occupancy rather than CPU memory-controller saturation.
- NVMe read rate was modest, at most 0.114 GiB/s, while write traffic reached approximately 3.15 GiB/s and busy time reached 75%. This experiment exercises concurrent offload writes as well as reads; device busy percentage does not isolate read-service latency or queue depth.

## Policy implication

The current experimental router policy uses `minExternalReusableTokens` as a lower bound: allow loading when the externally reusable prefix is at least the threshold. That is useful as a value screen in an uncongested system, but this experiment shows it is insufficient by itself. For a throughput objective at C64, the desirable aggregate action is “load the low-cost 1K cell, reject 2K and above once deferred work accumulates,” which no minimum-only threshold can express. A strict TTFT-tail objective would be more conservative because loading already regressed p99 at 1K/C64, 4K/C32, and 8K/C16 despite improving throughput.

The simplest next policy should remain experimental and use two conditions:

1. Preserve the request-level minimum reusable-token test as the estimated benefit screen.
2. Disable loading when an EPP-visible EWMA of pending restore work exceeds one calibrated ceiling, regardless of prefix length.

This matrix suggests a provisional research boundary between 0.51 and 3.45 deferred request-equivalents, but that exact metric is not currently the EPP input and its five-minute Prometheus derivation is unsuitable for a request-path controller. It should be used to validate the EPP-consumable pending-work proxy, not copied as a production knob.

## Conclusion and next experiment

The focused matrix resolves the earlier experimental validity problem and demonstrates the selective-loading opportunity: external restores help at low pressure, but queueing can make them sharply worse at higher pressure. It does not produce a single static `minExternalReusableTokens` threshold because the preferred action changes with concurrency and the nominal prefix is not equal to externally sourced tokens.

The smallest useful next experiment is a three-policy showcase on the same pinned node and workload grid:

1. Forced load.
2. Forced recompute.
3. Experimental selective policy with the existing reusable-token floor plus one pressure veto based on the EPP-visible pending-work EWMA.

Before that run, record the pending-work EWMA or its underlying per-endpoint gauge alongside the existing vLLM metrics. Calibrate only the one pressure ceiling: choose a value below the onset of the 2K/C64 queue but above the 1K/C64 steady state, then verify that the selective arm tracks forced load in the eight winning cells and forced recompute in the four losing cells. Do not add NVMe busy, bandwidth, and CPU-cache knobs yet; the current data do not show that they add decision value beyond queued work.

## Run registry

| Nominal prefix | Concurrency | Policy | MLflow run | Status |
|---:|---:|---|---|---|
| 1,024 | C16 | Forced recompute | [f8f62747](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/414/runs/f8f6274731374faeb82b620b4cad727b?workspace=benchflow) | Finished |
| 1,024 | C16 | Forced load | [fcb12635](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/414/runs/fcb1263593714a2ca111ded363dcfb2a?workspace=benchflow) | Finished |
| 1,024 | C32 | Forced recompute | [d7c236ff](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/415/runs/d7c236ff1b6e4f00bf919b3b91f496f9?workspace=benchflow) | Finished |
| 1,024 | C32 | Forced load | [67914a8f](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/415/runs/67914a8fd0b34e5d8b28c530cacf8458?workspace=benchflow) | Finished |
| 1,024 | C64 | Forced recompute | [ee5a16a9](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/ee5a16a99d3a4f3b9bafd6ea380d33db?workspace=benchflow) | Finished |
| 1,024 | C64 | Forced load | [94e56357](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/416/runs/94e56357e27747d7a81d5d992de995ca?workspace=benchflow) | Finished |
| 2,048 | C16 | Forced recompute | [05049570](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/417/runs/050495708e5f4645ad397409af7d38d7?workspace=benchflow) | Finished |
| 2,048 | C16 | Forced load | [cd502c0c](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/417/runs/cd502c0c1c944d869cce85c24d91fc76?workspace=benchflow) | Finished |
| 2,048 | C32 | Forced recompute | [c5cfd919](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/418/runs/c5cfd919c7034aee8864f75654f674d3?workspace=benchflow) | Finished |
| 2,048 | C32 | Forced load | [6c195d56](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/418/runs/6c195d5666fa44b080178fa4c9d0a0c8?workspace=benchflow) | Finished |
| 2,048 | C64 | Forced recompute | [8f8852ec](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/8f8852ecfa284a0b9b5a09c584cf6d38?workspace=benchflow) | Finished |
| 2,048 | C64 | Forced load | [3289257f](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/419/runs/3289257fca08486f9a2d17716f2aeae3?workspace=benchflow) | Finished |
| 4,096 | C16 | Forced recompute | [643ab7e4](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/420/runs/643ab7e4221a49cda60915b8834b447c?workspace=benchflow) | Finished |
| 4,096 | C16 | Forced load | [e0af40f4](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/420/runs/e0af40f41eae4978b389c35939d439e5?workspace=benchflow) | Finished |
| 4,096 | C32 | Forced recompute | [35361189](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/35361189ac0a40d3a6882e08cebe5966?workspace=benchflow) | Finished |
| 4,096 | C32 | Forced load | [406e529a](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/421/runs/406e529aa5fe48c0bbba669563f513c5?workspace=benchflow) | Finished |
| 4,096 | C64 | Forced recompute | [23bf2941](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/422/runs/23bf2941162a4cea8d43c3d6591b107c?workspace=benchflow) | Finished |
| 4,096 | C64 | Forced load | [719d30df](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/422/runs/719d30df4aa340dfb05e3fd2a39ba45f?workspace=benchflow) | Finished |
| 8,192 | C16 | Forced recompute | [d788943d](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/423/runs/d788943d0a104b47875dc469a345d911?workspace=benchflow) | Finished |
| 8,192 | C16 | Forced load | [a0f5a0af](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/423/runs/a0f5a0afd6914615b9f702c8441930eb?workspace=benchflow) | Finished |
| 8,192 | C32 | Forced recompute | [77111919](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/7711191922b54973a7138700fa6865e5?workspace=benchflow) | Finished |
| 8,192 | C32 | Forced load | [f82a8dd9](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/424/runs/f82a8dd9f48d4d93bc76ec1611c099ed?workspace=benchflow) | Finished |
| 8,192 | C64 | Forced recompute | [20c398fa](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/20c398fa76a24c498ec154384b0f3382?workspace=benchflow) | Finished |
| 8,192 | C64 | Forced load | [bb97035f](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/425/runs/bb97035f2cde4234b162bed54b87ff7b?workspace=benchflow) | Finished |

## Provenance and limitations

Client outcomes are exact GuideLLM summaries over each 300-second measurement. Prometheus artifacts were sampled every 15 seconds; aggregate mechanism values are arithmetic means over the final 300-second benchmark window, and the two time-series figures retain all native samples. Counter-derived rates and histogram means use five-minute Prometheus windows and can include prewarm history near the start. No repetitions were run, following the established GuideLLM single-run practice for this environment. The experiment establishes behavior for this model, topology, node, corpus, and software build only.

Related records: [[2026-09-09 - Qwen3-32B selective-loading crossover sweep]], [[2026-09-09 - Qwen3-32B C96 selective-loading saturation snapshot]], [[../01 - Calibration Protocol|KV Cache Offloading calibration protocol]], and [[Engineering/Projects/Selective KV Loading and Offloading/07 - Selective loading calibration test plan|Selective loading calibration test plan]].