---
title: Diadochos selective-load deployment calibration - 2026-09-07
date: 2026-09-07
type: research-report
experiment: Selective loading threshold calibration
status: conditionally-valid
model: nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
model_revision: local mounted snapshot, revision unavailable
image: quay.io/rh-ee-aperdomo/vllm@sha256:a52851b8bf871f4726a57907414a13363b6dcc2a111651aa534a642e358bb6a2
vllm_version: v0.27.0 selective-load v2 local build
tensor_parallelism: 8
replicas: 1
gpu_type: NVIDIA H100
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: default
cpu_bytes: 274877906944
offload_spec: CPU primary tier through OffloadingConnector
secondary_tier: none
telemetry_sample_rate: 1.0
concurrency: 1 for accepted threshold samples
random_seed: 20260907
cache_cleaning_state: unique prefixes followed by explicit HBM eviction churn; source outcome verified per request
---

# Diadochos selective-load deployment calibration - 2026-09-07

## Executive summary

This experiment calibrated the initial static load-versus-recompute threshold for the active Diadochos deployment. It directly compared CPU-load-allowed requests with forced recomputation across eight reusable-prefix sizes from 128 through 2,048 tokens. The accepted dataset contains 160 paired comparisons, 20 pairs per size, and exact mechanism separation: every load-allowed probe transferred external KV and every forced-recompute probe transferred none.

For the first balanced router policy, configure `minExternalReusableTokens: 1024`. Loading won 17 of 20 pairs at 1,024 tokens, with a median 86.0 ms advantage. Every larger measured bucket won at least 18 of 20 pairs. For a deliberately tail-conservative policy, use 1,536 tokens: its empirical p10 benefit was +52.4 ms and loading won 18 of 20 pairs; at 2,048 tokens it won 20 of 20.

Do not deploy the fitted mean crossover of approximately 69 tokens as the threshold. It describes the average low-contention cost curves, but the paired linear model explains only 31% of request-level variance and its bootstrap crossing interval is broad. Likewise, do not configure a pending-byte EWMA veto from this batch: the pressure pilots did not preserve valid CPU residency and are rejected for threshold selection.

## Validity verdict: Conditionally valid

The low-concurrency token-threshold result is valid for this exact deployment fingerprint and CPU-only external hierarchy. It is conditional rather than fully production-valid because it does not establish the crossover under representative concurrent transfer pressure, the model revision was not exposed by the mounted snapshot, and the pressure pilots were invalid.

The two pressure pilots produced 12 intended load targets, all of which were absent from HBM but missed in CPU. They therefore did not exercise load-versus-recompute under pressure and cannot set a pending-byte threshold. The most likely issue is cache preparation or CPU-residency history after the preceding churn-heavy sweeps, but this cause was not proven. No pod restart, error, or cancellation occurred.

## Recommended manual values

| Setting | Value | Intended use | Evidence |
|---|---:|---|---|
| `minExternalReusableTokens` | **1,024 tokens** | Recommended initial balanced policy | 17/20 load wins, median benefit +86.0 ms; all larger buckets at least 18/20 |
| `minExternalReusableTokens` | **1,536 tokens** | Optional tail-conservative policy | 18/20 load wins, empirical p10 benefit +52.4 ms |
| `maxPendingLoadBytesEWMA` | **Not calibrated / disabled** | Pressure veto | Pressure samples invalid; no defensible byte threshold |
| EWMA time constant or alpha | **Not calibrated** | Future dynamic policy | Native `/metrics` observations did not capture the short load backlog at sufficient cadence |

The recommended initial configuration should therefore make only the reusable-token decision. Preserve normal loading when external reusable tokens are at least 1,024; send `kv_load_tiers: []` below 1,024. Fail open to normal loading when compatible endpoint prefix evidence is missing or stale.

## Main takeaways

- Measured: the binary control was exact across all 320 accepted probes: 160 load-allowed probes loaded external KV and 160 forced-recompute probes did not.
- Measured: the median load advantage was already positive at 128 tokens, but win probability and lower-tail behavior were unstable below 1,024 tokens.
- Measured: at 1,024 tokens loading won 85% of pairs and saved 86.0 ms at the median; at 1,536 tokens it won 90% and its empirical p10 benefit became positive.
- Measured: each externally loaded token accounted for 131,072 bytes in the exported load-byte counter. Across all accepted loads, request-visible scheduler load wait was 7.23 ms at p50, 8.80 ms at p90, and 9.07 ms at p99.
- Inference: the static threshold is dominated by request and scheduler variance, not raw CPU-to-GPU copy time, in this quiescent regime.
- Conclusion: 1,024 tokens is the right first balanced value for this deployment; 1,536 is available when avoiding load regressions matters more than capturing smaller wins.
- Limitation: there is no calibrated dynamic pressure veto yet. Pending bytes should remain out of the active policy until a clean-state pressure experiment produces valid target loads.

## Headline metrics

Positive benefit means loading was faster: $D(N)=TTFT_{recompute}(N)-TTFT_{load}(N)$.

| External reusable tokens | Load median (ms) | Recompute median (ms) | Median benefit (ms) | Benefit p10 (ms) | Load wins | Pairs |
|---:|---:|---:|---:|---:|---:|---:|
| 128 | 405.0 | 413.2 | +2.5 | -49.4 | 12 | 20 |
| 256 | 408.6 | 444.6 | +35.3 | -71.5 | 15 | 20 |
| 384 | 396.3 | 435.8 | +14.7 | -49.4 | 14 | 20 |
| 512 | 459.5 | 517.9 | +51.4 | -89.1 | 15 | 20 |
| 768 | 492.6 | 531.9 | +56.6 | -31.8 | 15 | 20 |
| 1,024 | 505.7 | 585.7 | +86.0 | -28.6 | 17 | 20 |
| 1,536 | 513.7 | 638.4 | +187.9 | +52.4 | 18 | 20 |
| 2,048 | 509.2 | 708.1 | +215.4 | +91.9 | 20 | 20 |

## Evidence

Figure 1 shows the median and empirical p10 paired benefit at every measured prefix size. The vertical rules mark the balanced 1,024-token and conservative 1,536-token choices. Provenance is the two raw calibration artifacts listed below; the chart retains every native prefix bucket without aggregation beyond the stated per-bucket statistic.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Paired load benefit by reusable-prefix size","width":720,"height":380,"data":{"values":[{"tokens":128,"statistic":"Median","benefit_ms":2.473},{"tokens":128,"statistic":"P10","benefit_ms":-49.413},{"tokens":256,"statistic":"Median","benefit_ms":35.270},{"tokens":256,"statistic":"P10","benefit_ms":-71.475},{"tokens":384,"statistic":"Median","benefit_ms":14.663},{"tokens":384,"statistic":"P10","benefit_ms":-49.438},{"tokens":512,"statistic":"Median","benefit_ms":51.385},{"tokens":512,"statistic":"P10","benefit_ms":-89.113},{"tokens":768,"statistic":"Median","benefit_ms":56.616},{"tokens":768,"statistic":"P10","benefit_ms":-31.783},{"tokens":1024,"statistic":"Median","benefit_ms":85.993},{"tokens":1024,"statistic":"P10","benefit_ms":-28.567},{"tokens":1536,"statistic":"Median","benefit_ms":187.882},{"tokens":1536,"statistic":"P10","benefit_ms":52.351},{"tokens":2048,"statistic":"Median","benefit_ms":215.416},{"tokens":2048,"statistic":"P10","benefit_ms":91.939}]},"layer":[{"mark":{"type":"rule","color":"#666666","strokeDash":[4,4]},"encoding":{"y":{"datum":0}}},{"mark":{"type":"rule","color":"#1f77b4","strokeDash":[6,4]},"encoding":{"x":{"datum":1024}}},{"mark":{"type":"rule","color":"#ff7f0e","strokeDash":[2,3]},"encoding":{"x":{"datum":1536}}},{"mark":{"type":"line","point":{"filled":true,"size":70},"strokeWidth":2},"encoding":{"x":{"field":"tokens","type":"quantitative","title":"External reusable prefix (tokens)"},"y":{"field":"benefit_ms","type":"quantitative","title":"Recompute minus load latency (ms)"},"color":{"field":"statistic","type":"nominal","title":"Paired benefit statistic","scale":{"scheme":"category10"}},"tooltip":[{"field":"tokens","type":"quantitative","title":"External tokens"},{"field":"statistic","type":"nominal"},{"field":"benefit_ms","type":"quantitative","title":"Benefit (ms)"}]}}]}
```

Figure 1 shows why a single analytical crossing is insufficient. Median benefit is slightly positive even at 128 tokens, while the lower tail remains negative through 1,024 tokens. The 1,024-token recommendation chooses a stable 85%-win boundary; 1,536 is the first bucket whose empirical p10 is positive.

Figure 2 shows the observed fraction of paired requests won by loading. Provenance and granularity are the same accepted 20-pair buckets.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Observed load win rate by reusable-prefix size","width":720,"height":340,"data":{"values":[{"tokens":128,"win_rate":0.60},{"tokens":256,"win_rate":0.75},{"tokens":384,"win_rate":0.70},{"tokens":512,"win_rate":0.75},{"tokens":768,"win_rate":0.75},{"tokens":1024,"win_rate":0.85},{"tokens":1536,"win_rate":0.90},{"tokens":2048,"win_rate":1.00}]},"layer":[{"mark":{"type":"rule","color":"#666666","strokeDash":[4,4]},"encoding":{"y":{"datum":0.8}}},{"mark":{"type":"rule","color":"#1f77b4","strokeDash":[6,4]},"encoding":{"x":{"datum":1024}}},{"mark":{"type":"line","point":{"filled":true,"size":80},"strokeWidth":2,"color":"#1f77b4"},"encoding":{"x":{"field":"tokens","type":"quantitative","title":"External reusable prefix (tokens)"},"y":{"field":"win_rate","type":"quantitative","title":"Load win rate (fraction)","scale":{"domain":[0,1]}},"tooltip":[{"field":"tokens","type":"quantitative","title":"External tokens"},{"field":"win_rate","type":"quantitative","title":"Load win rate","format":".0%"}]}}]}
```

The observed win rate first exceeds 80% at 1,024 tokens and remains above that level for every larger bucket.

## Cost-model diagnostic

Across all accepted raw samples, ordinary least squares produced:

$$
T_{load}(N) \approx 433.27 + 0.04438N\ \text{ms}
$$

$$
T_{recompute}(N) \approx 425.20 + 0.16322N\ \text{ms}
$$

Fitting the paired difference directly produced:

$$
D(N) \approx -8.19 + 0.11894N\ \text{ms}
$$

The mean crossing is therefore approximately 68.8 tokens. A stratified 10,000-resample bootstrap gave crossing percentiles of -104.8, 69.9, and 205.2 tokens at p05, p50, and p95. The paired linear fit had $R^2=0.313$, so it is explanatory rather than a production threshold selector. It indicates that loading is physically cheap on this deployment, while the empirical win-rate rule supplies the operational safety margin.

The observed median load-byte ratio was 131,072 bytes per externally transferred token. Median effective load bandwidth was 13.91 GB/s across all sizes, with a p10 of 5.46 GB/s. These are mechanism diagnostics for this image and metric implementation, not portable hardware constants.

## Invalid pressure pilots

Two pilot batches attempted to place a 1,024-token target behind one, two, four, or eight concurrent reusable background requests. Treatment order was randomized and target telemetry was joined by response ID. All 12 load-allowed targets resolved as external misses with zero loaded target tokens. Forced-recompute targets correctly resolved as disabled.

Because the load arm did not load, the observed wall times are not load-versus-recompute comparisons. They must not be used to select `maxPendingLoadBytesEWMA`, EWMA alpha, or a concurrency threshold. The experiment was stopped rather than expanding invalid samples.

The next pressure attempt needs a clean CPU-cache state and an explicit preparation oracle that confirms target external residency before launching each cell. A model-server restart would provide the cleanest state, but no restart was performed during this calibration.

## Deployment fingerprint and post-run health

- Cluster namespace: `benchflow` on Diadochos.
- Model pod: `llm-d-selective-loading-28d3b96d7e-kserve-f479d97d6-59gzt`.
- Node: `diadochos-hqxzk-gpu-h100-fx7c8`.
- Image digest: `sha256:a52851b8bf871f4726a57907414a13363b6dcc2a111651aa534a642e358bb6a2`.
- Model: `nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8`, mounted at `/mnt/models/models/nvidia-Llama-3_1-Nemotron-Ultra-253B-v1-FP8`.
- TP8, GPU memory utilization 0.8, maximum model length 131,072, prefix caching enabled.
- CPU primary offload allocation: 274,877,906,944 bytes; no secondary tier configured.
- Offload telemetry enabled with sample rate 1.0.
- Accepted workload: concurrency 1, one generated token, randomized load/recompute probe order within each eviction batch.
- Post-run state: model pod ready, zero restarts, zero running requests, and zero waiting requests.

## Artifacts

- `.calibration-artifacts/2026-09-07-selective-load-cost-calibration-c1-small.json`: 128, 256, and 384 tokens; 20 pairs per bucket.
- `.calibration-artifacts/2026-09-07-selective-load-cost-calibration-c1.json`: 512, 768, 1,024, 1,536, and 2,048 tokens; 20 pairs per bucket.
- `.calibration-artifacts/2026-09-07-selective-load-pressure-pilot.json`: rejected two-repetition pressure pilot.
- `.calibration-artifacts/2026-09-07-selective-load-pressure-pilot-v2.json`: rejected store-quiescence pressure pilot.
- `.calibration-artifacts/run_selective_load_ewma_calibration.py`: runner used for the accepted sweeps and pilots; pressure treatment order was randomized and store quiescence was added during diagnosis.

## Conclusion and next step

Use 1,024 external reusable tokens as the initial manually calibrated threshold for this deployment. Keep the pending-byte gate disabled. If the first policy comparison prioritizes tail-risk avoidance, use 1,536 instead and report that choice explicitly.

The smallest remaining experiment is a clean-state pressure calibration. Restart or redeploy the model server, prime one target and a controlled set of background prefixes, verify the target exists externally before every measured cell, and then compare randomized load/recompute treatments at the same pressure. Only after valid loads span several pending-byte levels should `maxPendingLoadBytesEWMA` and its time constant be configured.

## Related

- [[00 - Index]]
- [[07 - Selective loading calibration test plan]]
- [[08 - 2026-09-07 selective loading calibration results]]

## Provenance

Direct execution against the active Diadochos deployment on 2026-09-07, request-level wall times, vLLM prompt-source and offload counters, structured per-request offload telemetry, Kubernetes deployment metadata, and post-run health checks. No MLflow run was created for these targeted probes. The plotted tables use every accepted prefix bucket; no bucket was dropped or downsampled.