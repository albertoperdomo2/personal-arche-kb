---
title: AgentX Weka admission prefetch first exploratory run
date: 2026-08-18
type: research_report
experiment: ABC
status: mechanism_valid_policy_ineffective_performance_inconclusive
model: nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
vllm_version: 0.27.0
image: quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-p1-v2
image_digest: sha256:097cffbde98e2566daf5fda1f1c6c50c9eaad26eccf5050bd2a9001c1b8c155a
tensor_parallelism: 8
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: not explicitly set
client_concurrency: 32
cpu_bytes: 274877906944
offload_spec: TieringOffloadingSpec
secondary_tier: filesystem on node-local NVMe
secondary_tier_threads: 64 read / 64 write
shared_memory: 300Gi
workload: AIPerf inferencex-agentx-mvp, semianalysisai/cc-traces-weka-062126
random_seed: 20260707
benchmark_duration_seconds: 1800
cache_bust: first_turn_prefix
cache_cleaning: node-local hostPath cleaned before each deployment
prometheus_source_cadence: 15 seconds
prefetch_log_cadence: 10 seconds when metrics emitted
---

# AgentX Weka admission prefetch first exploratory run

## Executive summary

This exploratory batch compared the repaired vLLM image with admission prefetch configured at `N=0` and `N=100` while replaying the same 30-minute AgentX Weka trace at client concurrency 32. Both cells completed 863 profiling requests without errors or cancellation, processed the same output-token count and almost identical input-token volume, and used the same immutable image digest. The runs used different H100-named nodes.

The mechanism executed, but the naive first-100 policy was a poor fit for this workload. It attempted 85,010 blocks: 77,356 (90.99%) were already present or pending in CPU and therefore redundant. Only 7,654 were submitted to NVMe; 6,664 (87.08% of submissions) failed because the assumed-resident filesystem block was absent, and 990 (12.93%) eventually became useful. Late was recorded for 7,539 submissions (98.50%), although late overlaps final useful/failed outcomes.

No performance improvement appeared. Request and token throughput were effectively identical because the AgentX replay issued the same trace, while N=100 had +1.05% mean TTFT, +2.62% p95 TTFT, +12.36% p99 TTFT, and +0.56% mean request latency. One pair on different nodes cannot establish causal harm, but it provides no reason to accept N=100.

## Validity verdict — Conditionally valid

**Valid for the descriptive mechanism result:** the request gate and N=100 server setting were active, exact ten-second counter deltas obey `attempted = redundant + promoted`, and all submitted promotions resolved as useful or load-failed.

**Inconclusive for causal performance:** there is one repetition per configuration, runtime nodes differ, `max_num_seqs` was not explicitly controlled, and AgentX throughput is trace-paced. The observed latency regression is a hypothesis, not a ranking.

## Main takeaways

- **The implementation ran:** 85,010 attempted = 77,356 redundant + 7,654 promoted + 0 skipped.
- **The first 100 blocks were normally already hot:** 90.99% of attempts were redundant, consistent with 94.07% reported prompt-cache reads.
- **Blind secondary residency was usually wrong:** 6,664 of 7,654 submissions (87.08%) load-failed; all promotions in the first five profiling minutes failed.
- **Useful opportunity was small:** 990 blocks became useful, only 1.16% of attempts and 12.93% of submissions.
- **Prefetch had little lead time:** server telemetry averaged 0.21 waiting requests for N=0 and 0.24 for N=100; p95 waiting was 2 and 1 respectively. Late was recorded for 98.50% of promotions.
- **No positive latency signal appeared:** the N=100 aggregate was slightly worse at mean/p95 TTFT and E2E, with a larger p99 TTFT difference.
- **Storage pressure was essentially unchanged:** runtime-node NVMe read/write means and GPU↔CPU transfer rates were close between cells, consistent with most attempts being redundant or failing quickly.

## Headline metrics

N=0 is the declared baseline. Relative deltas are descriptive only because the batch has one unpaired-node repetition.

| Metric | N=0 control | N=100 treatment | Treatment vs control |
|---|---:|---:|---:|
| Profiling requests | 863 | 863 | 0 |
| Errors / cancelled | 0 / no | 0 / no | — |
| Request throughput (req/s) | 0.469021 | 0.469021 | −0.00006% |
| Output throughput (tok/s) | 350.383 | 350.383 | −0.00006% |
| Mean TTFT (ms) | 1,129.19 | 1,141.11 | +1.05% |
| Median TTFT (ms) | 592.66 | 585.39 | −1.23% |
| p95 TTFT (ms) | 4,182.04 | 4,291.49 | +2.62% |
| p99 TTFT (ms) | 7,888.62 | 8,863.72 | +12.36% |
| Mean request latency (ms) | 20,875.36 | 20,991.72 | +0.56% |
| Median request latency (ms) | 8,999.51 | 9,091.45 | +1.02% |
| p95 request latency (ms) | 52,505.28 | 53,700.69 | +2.28% |
| Mean ITL (ms) | 26.780 | 27.055 | +1.03% |
| p95 ITL (ms) | 44.489 | 47.319 | +6.36% |
| Mean effective concurrency | 9.79 | 9.85 | +0.56% |
| Mean input length (tokens) | 63,881.60 | 63,881.35 | −0.0004% |
| Mean output length (tokens) | 747.05 | 747.05 | 0% |
| Prompt cache read share | 94.073% | 94.073% | approximately flat |

The identical request and output-token counts validate the aggregate workload shape. Individual request completion order is not pairable across the concurrent runs, so this report does not claim paired per-request deltas.

## Mechanism evidence

Exact profiling-window totals come from the vLLM model log. The counter output is emitted at ten-second cadence when metrics are present; no samples were removed.

| Outcome | Blocks | Fraction |
|---|---:|---:|
| Attempted | 85,010 | 100% of selections |
| Redundant | 77,356 | 90.99% of attempts |
| Promoted/submitted | 7,654 | 9.00% of attempts |
| Load failed | 6,664 | 87.08% of promotions; 7.84% of attempts |
| Useful | 990 | 12.93% of promotions; 1.16% of attempts |
| Late at first demand | 7,539 | 98.50% of promotions; overlaps useful/failed |
| Skipped / wasted / untracked | 0 / 0 / 0 | 0% |

The accounting identities both hold:

$$85{,}010_{attempted}=77{,}356_{redundant}+7{,}654_{promoted}+0_{skipped}$$

$$7{,}654_{promoted}=990_{useful}+6{,}664_{load\ failed}$$

Figure 1 compares the exact outcome totals. `Late` is diagnostic and overlaps the final useful/load-failed resolution; it is not an additional partition.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","title":"Figure 1 \u2014 N=100 admission-prefetch outcomes","width":760,"height":320,"data":{"values":[{"outcome":"Attempted","blocks":85010},{"outcome":"Redundant","blocks":77356},{"outcome":"Promoted","blocks":7654},{"outcome":"Late","blocks":7539},{"outcome":"Load failed","blocks":6664},{"outcome":"Useful","blocks":990}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"outcome","type":"nominal","title":"Outcome","sort":null,"axis":{"labelAngle":-20}},"y":{"field":"blocks","type":"quantitative","title":"Blocks (count)","scale":{"zero":true}},"color":{"field":"outcome","type":"nominal","title":"Outcome","scale":{"scheme":"category10"}},"tooltip":[{"field":"outcome","type":"nominal"},{"field":"blocks","type":"quantitative"}]}}
```

The first-100 policy mostly rediscovered CPU-resident prefix blocks. When it did submit work, the assumed-resident NVMe premise was usually false.

Full native-cadence plots:

- [[Reports/2026-08-18 - AgentX Weka admission prefetch first exploratory run/Admission prefetch mechanism|Admission prefetch mechanism]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch first exploratory run/Scheduler and latency comparison|Scheduler and latency comparison]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch first exploratory run/Storage and transfer comparison|Storage and transfer comparison]]

## Configuration and run registry

| Role | Run | Revision | N | Runtime node | Image digest | Disposition |
|---|---|---|---:|---|---|---|
| Control | [d82302a3…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/d82302a3769541cd9f98ad91bd8c3a69?workspace=benchflow) | `custom-no-prefetch-weka` | 0 | `…-6kl5z` | `097cffbd…` | valid control observation |
| Treatment | [915dac9e…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/915dac9e54d54b18b9b5a79ac8f69c2b?workspace=benchflow) | `custom-prefetch-weka` | 100 | `…-mt46x` | `097cffbd…` | mechanism valid; policy ineffective |

Both request bodies carried `kv_transfer_params.abc_admission_prefetch=true`; N=0 disabled the control at the manager. Both deployments used 256 GiB CPU capacity, node-local filesystem secondary storage, 64 read and 64 write threads, TP=8, one replica, 300 GiB `/dev/shm`, `gpu_memory_utilization=0.8`, and `max_model_len=131072`. AIPerf also issued 37 warm-up requests per cell with the same gate before the 863-request profiling phase. The node-local cache directory was cleaned before each deployment.

`max_num_seqs` was not explicitly passed and must not be described as `1`. The model node names contain `gpu-h100`, while the MLflow accelerator tag says H200; accelerator identity remains inconsistent in metadata.

## Validity and diagnostic evidence

- Both AIPerf sessions ended normally with 863 profiling records, empty error summaries, and `was_cancelled=false`.
- Input-token volume differed by only 219 tokens out of approximately 55.13 million; output tokens were identical at 644,705.
- Theoretical prefix-cache hit was 94.411% in both cells and server-reported prompt-cache read share was 94.073%, explaining the large redundant fraction.
- N=100 began producing useful outcomes only after approximately 5.2 profiling minutes. The first five minutes contained 2,329 promotions and 2,329 failures.
- Mean waiting depth remained below 0.25 in both cells. This run did not reproduce the high queueing visible in the earlier Weka reference run.
- GPU-utilization telemetry was empty. GPU KV-cache occupancy must not be substituted for GPU core utilization.

## Conclusions

### Measured observation

Admission prefetch ran throughout the N=100 trace, but only 990 of 85,010 selected blocks became useful. Most selections were already in CPU; most submitted NVMe reads targeted absent files; nearly all submitted work was late at first demand. Aggregate latency was not better.

### Inference

The earliest 100 hash blocks primarily cover a very hot shared/conversational prefix that remains in CPU. The remaining non-CPU keys frequently belong to cache-busted or not-yet-stored prefixes, making blind assumed residency inaccurate. The near-zero waiting depth also denies the mechanism the lead time demonstrated by the `max_num_seqs=1` toy workload.

### Working conclusion

The implementation is functioning, but `N=100` first-N admission prefetch should be rejected for this AgentX configuration. The experiment does not show that proactive prefetch is generally ineffective; it shows that selection location, secondary residency, and queue lead time matter.

## Next steps

1. **Run an N sweep under the same Weka workload:** `N ∈ {0, 25, 50, 100, 200}`, with at least three repetitions and balanced/interleaved node placement. Add `N=400` only if N=200 does not create capacity, storage, or tracking pressure.
2. **Use mechanism gates before comparing latency:** require exact accounting, then rank N by useful/attempted, load_failed/promoted, late/promoted, redundant/attempted, and resource pressure. N=100 currently provides the reference values 1.16%, 87.08%, 98.50%, and 90.99% respectively.
3. **Interpret the sweep in both directions:** smaller N tests whether overhead can be reduced without losing the few useful copies; larger N tests whether later conversational blocks beyond the always-hot prefix are more likely to reside on NVMe.
4. **Repeat the queue-pressure dimension after the N sweep:** the current Weka cells averaged fewer than 0.25 waiting requests. Increase offered concurrency or otherwise create a controlled waiting population, while preserving the AgentX trace, to test whether lead time reduces `late/promoted`.
5. **If every first-N value remains mostly redundant/failed, change selection rather than increasing N indefinitely:** the next policy should target a later window or use a cheap reuse/residency signal. That is the evidence-based transition toward the proposed heuristic phase.
6. **Retain N=0 in every batch and preserve the repaired digest.** Do not claim throughput improvement from this trace-paced scenario; focus on latency distributions and mechanism telemetry.

## Data provenance and limitations

AIPerf aggregates and request records come from `benchmark/profile_export_aiperf.json` and `.jsonl`. Exact prefetch totals and native ten-second progression come from the vLLM model log restricted to the profiling interval. Prometheus series are archived at native 15-second cadence and restricted to each AIPerf profiling window. Runtime-node NVMe metrics are node/device-wide. One repetition, different nodes, missing GPU utilization, implicit `max_num_seqs`, and asynchronous overlap between late and final outcomes limit causal interpretation.