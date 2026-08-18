---
title: Phase 1 admission prefetch repaired-image validation
date: 2026-08-18
type: research_report
experiment: ABC
status: mechanism_valid_performance_inconclusive
model: nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
vllm_version: 0.27.0
image: quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-p1-v2
image_digest: sha256:097cffbde98e2566daf5fda1f1c6c50c9eaad26eccf5050bd2a9001c1b8c155a
tensor_parallelism: 8
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 8192
max_num_seqs: 1
concurrency: 8
cpu_bytes: 274877906944
offload_spec: TieringOffloadingSpec
secondary_tier: filesystem on local NVMe
secondary_tier_threads: 64 read / 64 write
shared_memory: 300Gi
workload: 4096 synthetic prompt tokens, 64 output tokens, 256 measured requests
warmup: 1024 requests with abc_admission_prefetch=false
random_seed: 20260814
cache_cleaning: hostPath cleanup enabled between deployments; no reset during measured phase
prometheus_source_cadence: 15 seconds
vllm_metric_delta_cadence: 10 seconds
---

# Phase 1 admission prefetch repaired-image validation

## Executive summary

This batch is the first live validation of the repaired queued-request oracle prefetch implementation. It compared the same repaired custom image with `admission_prefetch_chunks=0` and `100` under the Nemotron FP8 4,096/64 workload at eight client streams. Warm-up requests carried `abc_admission_prefetch=false`; measured requests carried `true`.

The mechanism worked. The N=100 cell attempted exactly 25,600 first-N block promotions from the filesystem tier, promoted 25,344 blocks, classified 256 as redundant, and later observed every promoted block as useful. No promotion failed or was skipped, wasted, or dropped from tracking. A subset of 1,782 promoted blocks (7.03%) was still pending at first demand and subsequently became useful.

The performance comparison is not yet acceptable as causal evidence. The N=0 GuideLLM session again finalized with only 255 of 256 requests processed, the two model pods used different nodes, each configuration has one repetition, and both cells JIT-compiled `_swap_blocks_kernel` during the measured phase. The treatment's better TTFT tail is promising but provisional.

## Validity verdict — Conditionally valid

**Valid for the mechanism claim:** repaired-image proactive NVMe→CPU promotion executed, obeyed its accounting identity, completed without failures, and produced useful copies before demand for most promoted blocks.

**Invalid / inconclusive for the performance claim:** the control is incomplete at 255/256, placement is unpaired, measured-phase JIT remains, and there is only one repetition. Do not claim that prefetch improved TTFT or throughput from this pair alone.

## Main takeaways

- **Mechanism success:** 25,600 attempts = 25,344 promoted + 256 redundant + 0 skipped.
- **Perfect eventual usefulness:** 25,344/25,344 promoted blocks were consumed by demand.
- **Mostly timely:** 23,562 promoted blocks were ready at first demand; 1,782 (7.03%) were late once and then useful.
- **No failure signal:** skipped, wasted, untracked, and load_failed remained absent/zero throughout the treatment.
- **Observed performance was essentially flat at the mean:** request throughput was 1.0583 versus 1.0574 req/s and mean TTFT was 6,556.0 versus 6,563.9 ms.
- **Observed tail was better in treatment:** p95 TTFT was 3.04% lower and p99 12.33% lower; requests above 8 seconds fell from 8 to 1. This is not yet causal evidence.
- **Validity problems remain:** the control left one request processing, nodes differed, actual accelerator metadata is inconsistent, and the swap kernel compiled during measurement.

## Headline metrics

N=0 is the declared direct baseline, but relative deltas are marked N/A because its session is incomplete.

| Configuration | N | Completed | Errors | Throughput (req/s) | Output rate (tok/s) | Mean TTFT (ms) | Median TTFT (ms) | p95 TTFT (ms) | p99 TTFT (ms) | Mean E2E (s) | Valid delta |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| Repaired-image control | 0 | 255/256 | 0 | 1.0574 | 67.86 | 6563.9 | 6564.5 | 7217.1 | 9053.8 | 7.4630 | baseline invalid |
| Repaired-image treatment | 100 | 256/256 | 0 | 1.0583 | 67.96 | 6556.0 | 6592.0 | 6997.7 | 7937.3 | 7.4571 | N/A |

For orientation only, treatment versus the incomplete control was +0.08% request throughput, −0.12% mean TTFT, +0.42% median TTFT, −3.04% p95 TTFT, −12.33% p99 TTFT, −0.08% mean E2E, and −2.67% p95 E2E. These observations must not be presented as a feature ranking.

## Mechanism evidence

Exact totals below come from summing the vLLM ten-second metric deltas covering the measured interval. The archived Prometheus 15-second rate series independently shows the same `tier="1:fs"` labels and the attempted partition identity.

| Outcome | Blocks | Fraction |
|---|---:|---:|
| Attempted | 25,600 | 100% of selected blocks |
| Promoted | 25,344 | 99% of attempts |
| Redundant | 256 | 1% of attempts |
| Skipped | 0 | 0% |
| Useful | 25,344 | 100% of promoted |
| Late at first demand | 1,782 | 7.03% of promoted |
| Ready at first demand | 23,562 | 92.97% of promoted |
| Wasted | 0 | 0% |
| Untracked | 0 | 0% |
| Load failed | 0 | 0% |

The accounting identity holds exactly:

$$25{,}600_{attempted}=25{,}344_{promoted}+256_{redundant}+0_{skipped}$$

Late and useful are not mutually exclusive. The 1,782 late blocks were pending at the first demand observation and were later counted useful after their promotion completed.

Figure 1 shows the nonzero mechanism counters. Counts are exact measured-interval totals from the model log.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Admission-prefetch block outcomes","width":700,"height":280,"data":{"values":[{"outcome":"Attempted","blocks":25600},{"outcome":"Promoted","blocks":25344},{"outcome":"Useful","blocks":25344},{"outcome":"Ready at first demand","blocks":23562},{"outcome":"Late at first demand","blocks":1782},{"outcome":"Redundant","blocks":256}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"outcome","type":"nominal","title":null,"sort":null,"axis":{"labelAngle":-20}},"y":{"field":"blocks","type":"quantitative","title":"Blocks (count)","scale":{"zero":true}},"color":{"field":"outcome","type":"nominal","title":"Outcome","scale":{"scheme":"category10"}},"tooltip":[{"field":"outcome","type":"nominal"},{"field":"blocks","type":"quantitative","title":"Blocks"}]}}
```

Figure 1 establishes the proof-of-concept's central claim: blind first-N selection caused real secondary-tier promotion and almost all copies were ready by first demand.

## Performance and tail evidence

Figure 2 uses individual GuideLLM request records at their native request granularity. The control contains 255 completed requests; treatment contains 256.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — Completed requests above TTFT thresholds","width":700,"height":280,"data":{"values":[{"category":">7 s — N=0","configuration":"N=0 control","threshold":">7 s","requests":28},{"category":">7 s — N=100","configuration":"N=100 treatment","threshold":">7 s","requests":12},{"category":">8 s — N=0","configuration":"N=0 control","threshold":">8 s","requests":8},{"category":">8 s — N=100","configuration":"N=100 treatment","threshold":">8 s","requests":1},{"category":">9 s — N=0","configuration":"N=0 control","threshold":">9 s","requests":3},{"category":">9 s — N=100","configuration":"N=100 treatment","threshold":">9 s","requests":0}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"category","type":"nominal","title":"TTFT threshold and configuration","sort":null,"axis":{"labelAngle":-25}},"y":{"field":"requests","type":"quantitative","title":"Completed requests (count)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"scheme":"category10"}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"threshold","type":"nominal","title":"TTFT threshold"},{"field":"requests","type":"quantitative","title":"Requests"}]}}
```

The treatment had fewer slow requests at every shown threshold, consistent with the p95/p99 aggregates. This is a signal worth repeating, not proof: a missing control request, different runtime nodes, JIT, and one pair can all alter the tail.

## Prefetch time-series evidence

The next two figures use every native ten-second vLLM metric-delta emission during the N=100 measured interval; no samples were dropped or downsampled. The deltas were accumulated from measurement start so the lines show the experiment's exact running totals.

Figure 3 plots the cumulative admission-prefetch counters. Attempted and promoted advance almost in lockstep; useful temporarily trails while copies wait for demand and catches promoted exactly at the end. Late is a subset, not a partition of useful.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3 — Cumulative vLLM admission-prefetch metrics","width":760,"height":320,"data":{"values":[{"elapsed_s":9,"metric":"Attempted","blocks":1600},{"elapsed_s":9,"metric":"Promoted","blocks":1584},{"elapsed_s":9,"metric":"Useful","blocks":1584},{"elapsed_s":9,"metric":"Late at first demand","blocks":891},{"elapsed_s":19,"metric":"Attempted","blocks":2600},{"elapsed_s":19,"metric":"Promoted","blocks":2574},{"elapsed_s":19,"metric":"Useful","blocks":2376},{"elapsed_s":19,"metric":"Late at first demand","blocks":990},{"elapsed_s":29,"metric":"Attempted","blocks":3600},{"elapsed_s":29,"metric":"Promoted","blocks":3564},{"elapsed_s":29,"metric":"Useful","blocks":3069},{"elapsed_s":29,"metric":"Late at first demand","blocks":990},{"elapsed_s":39,"metric":"Attempted","blocks":4700},{"elapsed_s":39,"metric":"Promoted","blocks":4653},{"elapsed_s":39,"metric":"Useful","blocks":4455},{"elapsed_s":39,"metric":"Late at first demand","blocks":990},{"elapsed_s":49,"metric":"Attempted","blocks":5800},{"elapsed_s":49,"metric":"Promoted","blocks":5742},{"elapsed_s":49,"metric":"Useful","blocks":5148},{"elapsed_s":49,"metric":"Late at first demand","blocks":990},{"elapsed_s":59,"metric":"Attempted","blocks":6800},{"elapsed_s":59,"metric":"Promoted","blocks":6732},{"elapsed_s":59,"metric":"Useful","blocks":6633},{"elapsed_s":59,"metric":"Late at first demand","blocks":1089},{"elapsed_s":69,"metric":"Attempted","blocks":7900},{"elapsed_s":69,"metric":"Promoted","blocks":7821},{"elapsed_s":69,"metric":"Useful","blocks":7623},{"elapsed_s":69,"metric":"Late at first demand","blocks":1188},{"elapsed_s":79,"metric":"Attempted","blocks":8900},{"elapsed_s":79,"metric":"Promoted","blocks":8811},{"elapsed_s":79,"metric":"Useful","blocks":8712},{"elapsed_s":79,"metric":"Late at first demand","blocks":1188},{"elapsed_s":89,"metric":"Attempted","blocks":10000},{"elapsed_s":89,"metric":"Promoted","blocks":9900},{"elapsed_s":89,"metric":"Useful","blocks":9702},{"elapsed_s":89,"metric":"Late at first demand","blocks":1188},{"elapsed_s":99,"metric":"Attempted","blocks":11100},{"elapsed_s":99,"metric":"Promoted","blocks":10989},{"elapsed_s":99,"metric":"Useful","blocks":10791},{"elapsed_s":99,"metric":"Late at first demand","blocks":1188},{"elapsed_s":109,"metric":"Attempted","blocks":12100},{"elapsed_s":109,"metric":"Promoted","blocks":11979},{"elapsed_s":109,"metric":"Useful","blocks":11781},{"elapsed_s":109,"metric":"Late at first demand","blocks":1188},{"elapsed_s":119,"metric":"Attempted","blocks":13200},{"elapsed_s":119,"metric":"Promoted","blocks":13068},{"elapsed_s":119,"metric":"Useful","blocks":12870},{"elapsed_s":119,"metric":"Late at first demand","blocks":1287},{"elapsed_s":129,"metric":"Attempted","blocks":14300},{"elapsed_s":129,"metric":"Promoted","blocks":14157},{"elapsed_s":129,"metric":"Useful","blocks":13860},{"elapsed_s":129,"metric":"Late at first demand","blocks":1287},{"elapsed_s":139,"metric":"Attempted","blocks":15300},{"elapsed_s":139,"metric":"Promoted","blocks":15147},{"elapsed_s":139,"metric":"Useful","blocks":14949},{"elapsed_s":139,"metric":"Late at first demand","blocks":1287},{"elapsed_s":149,"metric":"Attempted","blocks":16400},{"elapsed_s":149,"metric":"Promoted","blocks":16236},{"elapsed_s":149,"metric":"Useful","blocks":15939},{"elapsed_s":149,"metric":"Late at first demand","blocks":1287},{"elapsed_s":159,"metric":"Attempted","blocks":17500},{"elapsed_s":159,"metric":"Promoted","blocks":17325},{"elapsed_s":159,"metric":"Useful","blocks":17028},{"elapsed_s":159,"metric":"Late at first demand","blocks":1386},{"elapsed_s":169,"metric":"Attempted","blocks":18500},{"elapsed_s":169,"metric":"Promoted","blocks":18315},{"elapsed_s":169,"metric":"Useful","blocks":18018},{"elapsed_s":169,"metric":"Late at first demand","blocks":1485},{"elapsed_s":179,"metric":"Attempted","blocks":19600},{"elapsed_s":179,"metric":"Promoted","blocks":19404},{"elapsed_s":179,"metric":"Useful","blocks":19107},{"elapsed_s":179,"metric":"Late at first demand","blocks":1584},{"elapsed_s":189,"metric":"Attempted","blocks":20700},{"elapsed_s":189,"metric":"Promoted","blocks":20493},{"elapsed_s":189,"metric":"Useful","blocks":20097},{"elapsed_s":189,"metric":"Late at first demand","blocks":1584},{"elapsed_s":199,"metric":"Attempted","blocks":21700},{"elapsed_s":199,"metric":"Promoted","blocks":21483},{"elapsed_s":199,"metric":"Useful","blocks":21186},{"elapsed_s":199,"metric":"Late at first demand","blocks":1683},{"elapsed_s":209,"metric":"Attempted","blocks":22800},{"elapsed_s":209,"metric":"Promoted","blocks":22572},{"elapsed_s":209,"metric":"Useful","blocks":22176},{"elapsed_s":209,"metric":"Late at first demand","blocks":1782},{"elapsed_s":219,"metric":"Attempted","blocks":23900},{"elapsed_s":219,"metric":"Promoted","blocks":23661},{"elapsed_s":219,"metric":"Useful","blocks":23562},{"elapsed_s":219,"metric":"Late at first demand","blocks":1782},{"elapsed_s":229,"metric":"Attempted","blocks":24900},{"elapsed_s":229,"metric":"Promoted","blocks":24651},{"elapsed_s":229,"metric":"Useful","blocks":24255},{"elapsed_s":229,"metric":"Late at first demand","blocks":1782},{"elapsed_s":239,"metric":"Attempted","blocks":25600},{"elapsed_s":239,"metric":"Promoted","blocks":25344},{"elapsed_s":239,"metric":"Useful","blocks":25344},{"elapsed_s":239,"metric":"Late at first demand","blocks":1782}]},"mark":{"type":"line","point":true,"strokeWidth":2},"encoding":{"x":{"field":"elapsed_s","type":"quantitative","title":"Elapsed measured time (s)"},"y":{"field":"blocks","type":"quantitative","title":"Cumulative blocks (count)","scale":{"zero":true}},"color":{"field":"metric","type":"nominal","title":"Prefetch metric","scale":{"scheme":"category10"}},"tooltip":[{"field":"elapsed_s","type":"quantitative","title":"Elapsed time (s)"},{"field":"metric","type":"nominal"},{"field":"blocks","type":"quantitative","title":"Cumulative blocks"}]}}
```

The steady rise proves admission prefetch remained active throughout the run rather than appearing only during startup. The final points reproduce the exact accounting totals: 25,600 attempted, 25,344 promoted and useful, and 1,782 late.

Figure 4 normalizes cumulative useful and late counts by cumulative promoted blocks. This separates eventual utility from timing.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4 — Cumulative usefulness and lateness","width":760,"height":300,"data":{"values":[{"elapsed_s":9,"metric":"Useful / promoted","percent":100},{"elapsed_s":9,"metric":"Late / promoted","percent":56.25},{"elapsed_s":19,"metric":"Useful / promoted","percent":92.31},{"elapsed_s":19,"metric":"Late / promoted","percent":38.46},{"elapsed_s":29,"metric":"Useful / promoted","percent":86.11},{"elapsed_s":29,"metric":"Late / promoted","percent":27.78},{"elapsed_s":39,"metric":"Useful / promoted","percent":95.74},{"elapsed_s":39,"metric":"Late / promoted","percent":21.28},{"elapsed_s":49,"metric":"Useful / promoted","percent":89.66},{"elapsed_s":49,"metric":"Late / promoted","percent":17.24},{"elapsed_s":59,"metric":"Useful / promoted","percent":98.53},{"elapsed_s":59,"metric":"Late / promoted","percent":16.18},{"elapsed_s":69,"metric":"Useful / promoted","percent":97.47},{"elapsed_s":69,"metric":"Late / promoted","percent":15.19},{"elapsed_s":79,"metric":"Useful / promoted","percent":98.88},{"elapsed_s":79,"metric":"Late / promoted","percent":13.48},{"elapsed_s":89,"metric":"Useful / promoted","percent":98},{"elapsed_s":89,"metric":"Late / promoted","percent":12},{"elapsed_s":99,"metric":"Useful / promoted","percent":98.2},{"elapsed_s":99,"metric":"Late / promoted","percent":10.81},{"elapsed_s":109,"metric":"Useful / promoted","percent":98.35},{"elapsed_s":109,"metric":"Late / promoted","percent":9.92},{"elapsed_s":119,"metric":"Useful / promoted","percent":98.48},{"elapsed_s":119,"metric":"Late / promoted","percent":9.85},{"elapsed_s":129,"metric":"Useful / promoted","percent":97.9},{"elapsed_s":129,"metric":"Late / promoted","percent":9.09},{"elapsed_s":139,"metric":"Useful / promoted","percent":98.69},{"elapsed_s":139,"metric":"Late / promoted","percent":8.5},{"elapsed_s":149,"metric":"Useful / promoted","percent":98.17},{"elapsed_s":149,"metric":"Late / promoted","percent":7.93},{"elapsed_s":159,"metric":"Useful / promoted","percent":98.29},{"elapsed_s":159,"metric":"Late / promoted","percent":8},{"elapsed_s":169,"metric":"Useful / promoted","percent":98.38},{"elapsed_s":169,"metric":"Late / promoted","percent":8.11},{"elapsed_s":179,"metric":"Useful / promoted","percent":98.47},{"elapsed_s":179,"metric":"Late / promoted","percent":8.16},{"elapsed_s":189,"metric":"Useful / promoted","percent":98.07},{"elapsed_s":189,"metric":"Late / promoted","percent":7.73},{"elapsed_s":199,"metric":"Useful / promoted","percent":98.62},{"elapsed_s":199,"metric":"Late / promoted","percent":7.83},{"elapsed_s":209,"metric":"Useful / promoted","percent":98.25},{"elapsed_s":209,"metric":"Late / promoted","percent":7.89},{"elapsed_s":219,"metric":"Useful / promoted","percent":99.58},{"elapsed_s":219,"metric":"Late / promoted","percent":7.53},{"elapsed_s":229,"metric":"Useful / promoted","percent":98.39},{"elapsed_s":229,"metric":"Late / promoted","percent":7.23},{"elapsed_s":239,"metric":"Useful / promoted","percent":100},{"elapsed_s":239,"metric":"Late / promoted","percent":7.03}]},"mark":{"type":"line","point":true,"strokeWidth":2},"encoding":{"x":{"field":"elapsed_s","type":"quantitative","title":"Elapsed measured time (s)"},"y":{"field":"percent","type":"quantitative","title":"Fraction of promoted blocks (%)","scale":{"zero":true,"domain":[0,100]}},"color":{"field":"metric","type":"nominal","title":"Outcome ratio","scale":{"scheme":"category10"}},"tooltip":[{"field":"elapsed_s","type":"quantitative","title":"Elapsed time (s)"},{"field":"metric","type":"nominal"},{"field":"percent","type":"quantitative","title":"Percent (%)"}]}}
```

The initial burst was timing-sensitive: 56.25% of the first 1,584 promotions were still pending at first demand. That cumulative late fraction fell rapidly and finished at 7.03%, while cumulative usefulness recovered to 100%. The first burst coincides with measured-phase `_swap_blocks_kernel` JIT, so precompiling that path is the clearest next step for improving timeliness.

## Offload and storage diagnostics

Native 15-second samples restricted to each measured window show broadly similar reactive-offload pressure:

| Metric | N=0 control | N=100 treatment |
|---|---:|---:|
| External prompt-token share mean | 46.66% | 46.09% |
| Tiering async lookup mean | 4.152 s | 4.244 s |
| Waiting requests mean | 7.00 | 6.65 |
| CPU→GPU KV rate mean | 476.8 MiB/s | 518.2 MiB/s |
| Runtime-node NVMe read mean | 216.2 MiB/s | 225.7 MiB/s |
| Runtime-node NVMe write mean | 212.0 MiB/s | 205.7 MiB/s |

The treatment's higher CPU→GPU and NVMe-read means are consistent with active KV movement, but the total storage work is similar because both cells ultimately retrieve the same replayed KV. The async lookup aggregate includes different work under proactive promotion and should not be treated as a like-for-like request-stall metric.

GPU utilization series were unavailable in the archived artifacts. GPU memory and KV-cache occupancy must not be substituted for GPU core utilization.

## Run registry and configuration evidence

| Role | Run | Revision | N | Image digest | Model node | Disposition |
|---|---|---|---:|---|---|---|
| Control | [3581db3f…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/3581db3f82d7427c883ff72113390121?workspace=benchflow) | `custom-no-prefetch` | 0 | `097cffbd...` | `...-mt46x` | invalid performance control; 255/256 |
| Treatment | [b28bd1db…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/b28bd1db0836406a94c31c2e3faa7c35?workspace=benchflow) | `custom-prefetch` | 100 | `097cffbd...` | `...-6kl5z` | mechanism accepted; performance provisional |

Both pods resolved `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-p1-v2` to the same repaired digest `sha256:097cffbde98e2566daf5fda1f1c6c50c9eaad26eccf5050bd2a9001c1b8c155a`. The server logs parsed N=0 and N=100 correctly. Warm-up and measurement HTTP gates were false and true respectively.

The MLflow accelerator tag says H200 while both model node names contain `gpu-h100`; actual accelerator SKU remains unresolved.

## Failure and validity evidence

The control scheduler state recorded 256 created, 255 processed, 255 successful, one processing, zero pending/queued, null `end_processing_time`, and one remaining request in the max-request constraint. GuideLLM nevertheless emitted completion and BenchFlow marked the run FINISHED. This is the same incomplete-final-request symptom seen in the immediately preceding control rerun.

Treatment recorded 256 created, processed, and successful, no processing requests, and a valid `end_processing_time`.

Both model logs warned that `_swap_blocks_kernel` JIT compilation occurred during inference immediately after measurement began. The unrelated weight-loader message “Auto-prefetch is disabled” refers to safetensors model loading, not ABC KV-cache prefetch.

## Conclusions

### Measured observation

The repaired implementation performed 25,344 successful NVMe→CPU promotions. Every promotion was later consumed, and 92.97% were complete before the first corresponding demand observation. No proactive load failed.

### Inference

The queued-request oracle has enough lead time at concurrency eight to make most N=100 promotions timely. The remaining 7.03% late subset indicates overlap is not perfect and gives the next tuning work a concrete target.

### Working conclusion

Phase 1 has passed its mechanism proof. Proactive, blind admission prefetch demonstrably works for this constructed replay workload. Phase 1 has **not yet passed the performance proof**: the current pair does not establish a TTFT or throughput improvement.

## Next steps

1. Fix or work around the GuideLLM final-request race; reject every cell that does not report exactly 256 processed and successful requests.
2. Precompile `_swap_blocks_kernel` before measurement by adding a marked warm-up phase that exercises the admission promotion shape without contaminating measured counters, or otherwise ensuring the kernel shape is compiled before timing.
3. Run at least five paired N=0/N=100 repetitions on the same node, or interleave and balance placements when pinning is impossible.
4. Preserve the same repaired digest and the warm-up/measurement request gates.
5. Evaluate paired per-request TTFT and throughput only after all sessions are complete. Treat the observed p95/p99 improvement as the hypothesis for that batch.
6. Track late/promoted as the timing diagnostic. After the performance proof, sweep N in `{25, 50, 100, 200}` to test whether a smaller N retains tail benefit with fewer late promotions and less storage pressure.

## Data provenance and limitations

GuideLLM aggregates and individual request records come from `results/benchmark_output.json`. Exact prefetch totals come from vLLM's ten-second metric-delta log entries covering the measured interval. Prometheus mechanism and resource series are archived at native 15-second cadence. Pod manifests and model logs establish arguments, image digest, placement, and JIT timing.

Device-level NVMe metrics are node-wide. Single-run temporal association does not establish causality. Missing GPU utilization telemetry, unmatched nodes, one incomplete control request, and one repetition limit the performance interpretation.