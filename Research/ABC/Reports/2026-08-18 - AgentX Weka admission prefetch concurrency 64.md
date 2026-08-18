---
title: AgentX Weka admission prefetch at concurrency 64
date: 2026-08-18
type: research_report
experiment: ABC
status: phase1_mechanism_supported_performance_inconclusive
model: nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
vllm_version: 0.27.0
image: quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-p1-v2
image_digest: sha256:097cffbde98e2566daf5fda1f1c6c50c9eaad26eccf5050bd2a9001c1b8c155a
tensor_parallelism: 8
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: not explicitly set
client_concurrency: 64
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

# AgentX Weka admission prefetch at concurrency 64

## Executive summary

This second AgentX/Weka batch compared `N=0` and `N=100` at client concurrency 64, using the same repaired vLLM image, model, trace, seed, cache-busting mode, CPU/NVMe tiers, and request gate as the concurrency-32 exploration. The purpose was not yet to prove a production performance win. It was to test the Phase 1 intuition that queued requests provide enough lead time for admission-time NVMe→CPU promotion to finish before demand.

That intuition is supported. Mean waiting depth rose from approximately 0.24 requests in the prior N=100 run to 6.22 here. At the same time, `late/promoted` fell from 98.50% to 42.39%, `useful/attempted` rose from 1.16% to 15.81%, and `load_failed/promoted` fell from 87.08% to 37.78%. The treatment selected 123,419 blocks, submitted 33,103, and recorded 19,508 useful proactive copies. This is the clearest AgentX evidence so far that the wiring works and that lead time changes the result in the predicted direction.

The performance result is still not positive. Relative to the one control cell, N=100 had 3.19% higher mean TTFT, 10.44% higher p95 TTFT, and 1.23% higher mean end-to-end latency; median TTFT improved 4.31% and mean/p95 inter-token latency improved 3.15%/8.97%. The pair used different runtime nodes, the treatment produced four fewer requests, both cells operated near full GPU KV occupancy with queueing and preemptions, and there is only one repetition. The mixed aggregate must not be interpreted as a causal ranking.

## Validity verdict — Conditionally valid

**Valid for Phase 1 mechanism and queue-sensitivity:** exact manager counters satisfy selection accounting, 19,508 promotions were observed useful, and added queue pressure substantially reduced lateness while raising useful yield.

**Inconclusive for performance:** one unpaired-node repetition, slight request-count drift (1,248 versus 1,244), heavy saturation, different preemption rates, and trace-paced throughput prevent a causal latency or throughput conclusion.

## Main takeaways

- **The intended pressure appeared:** N=0 averaged 27.56 running / 6.03 waiting; N=100 averaged 27.47 running / 6.22 waiting. The prior concurrency-32 cells averaged only about 11 running and less than 0.25 waiting.
- **Lead time helped exactly where predicted:** lateness fell by 56.11 percentage points, from 98.50% to 42.39% of promotions.
- **The useful population became material:** 19,508 blocks were useful—58.93% of promotions and 15.81% of all attempted blocks, versus 990 and 1.16% of attempts at concurrency 32.
- **Naive selection remains inefficient:** 73.18% of attempts were redundant and 37.78% of submitted loads still failed. N=100 is a useful POC knob, not an accepted policy.
- **The performance signal is mixed, not positive:** median TTFT and ITL improved, while mean and p90–p99 TTFT did not. No production benefit should be claimed from this pair.
- **The cells were highly stressed:** mean GPU KV occupancy was about 91–92%, p95 occupancy was about 99%, and treatment preemption rate was higher. Queue pressure created lead time but also moved the test into a different bottleneck regime.
- **Phase 1 succeeds on its actual objective:** together with the repaired GuideLLM result, this batch proves the request gate, key selection, blind NVMe submission, batching, outcome tracking, and timing intuition. It does not yet prove a generally beneficial heuristic.

## Headline metrics

N=0 is the declared baseline. Deltas are descriptive only.

| Metric | N=0 control | N=100 treatment | Treatment vs control |
|---|---:|---:|---:|
| Profiling requests | 1,248 | 1,244 | −4 (−0.32%) |
| Errors / cancelled | 0 / no | 0 / no | — |
| Request throughput (req/s) | 0.678260 | 0.676086 | −0.32% |
| Output throughput (tok/s) | 453.182 | 453.434 | +0.06% |
| Mean TTFT (ms) | 8,077.82 | 8,335.40 | +3.19% |
| Median TTFT (ms) | 5,981.83 | 5,724.23 | −4.31% |
| p90 TTFT (ms) | 19,578.62 | 21,351.61 | +9.06% |
| p95 TTFT (ms) | 23,589.86 | 26,051.97 | +10.44% |
| p99 TTFT (ms) | 30,180.68 | 31,481.17 | +4.31% |
| Mean request latency (ms) | 43,191.61 | 43,721.94 | +1.23% |
| Median request latency (ms) | 24,080.01 | 24,020.45 | −0.25% |
| p95 request latency (ms) | 136,112.63 | 135,104.12 | −0.74% |
| p99 request latency (ms) | 326,358.49 | 325,271.09 | −0.33% |
| Mean ITL (ms) | 55.062 | 53.329 | −3.15% |
| p95 ITL (ms) | 94.036 | 85.601 | −8.97% |
| Mean effective concurrency | 29.30 | 29.56 | +0.90% |
| Prompt cache read share | 92.897% | 92.915% | +0.018 pp |

The 4-request difference is small (0.32%) but means this is not an exactly matched request set. Individual completions are not pairable across concurrent AgentX runs.

## Mechanism evidence

Exact profiling-window totals come from 181 vLLM metric-log records. No log samples were removed.

| Outcome | Blocks | Fraction |
|---|---:|---:|
| Attempted | 123,419 | 100% |
| Redundant | 90,316 | 73.18% of attempts |
| Promoted/submitted | 33,103 | 26.82% of attempts |
| Useful | 19,508 | 58.93% of promotions; 15.81% of attempts |
| Load failed | 12,506 | 37.78% of promotions |
| Wasted | 594 | 1.79% of promotions |
| Late at first demand | 14,031 | 42.39% of promotions; overlaps final outcomes |
| Unresolved at final profiling log | 495 | 1.50% of promotions |
| Skipped / untracked | 0 / 0 | 0% |

Selection accounting is exact:

$$123{,}419_{attempted}=90{,}316_{redundant}+33{,}103_{promoted}+0_{skipped}$$

Final outcomes observed inside the profiling window cover 32,608 promotions; 495 remained in flight or otherwise unresolved at the last emitted profiling record. This window-edge residue is reported explicitly rather than forced into an outcome.

Figure 1 compares mechanism ratios with the prior concurrency-32 N=100 run. `Late` overlaps the final outcome categories.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 \u2014 Admission-prefetch efficiency changes with queue pressure","width":760,"height":320,"data":{"values":[{"metric":"Redundant / attempted","concurrency":"32","percent":90.99},{"metric":"Redundant / attempted","concurrency":"64","percent":73.17835989596415},{"metric":"Useful / attempted","concurrency":"32","percent":1.1646},{"metric":"Useful / attempted","concurrency":"64","percent":15.806318314035927},{"metric":"Load failed / promoted","concurrency":"32","percent":87.08},{"metric":"Load failed / promoted","concurrency":"64","percent":37.779053258012866},{"metric":"Late / promoted","concurrency":"32","percent":98.5},{"metric":"Late / promoted","concurrency":"64","percent":42.38588647554602}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"metric","type":"nominal","title":"Mechanism ratio"},"y":{"field":"percent","type":"quantitative","title":"Blocks (%)","scale":{"zero":true}},"color":{"field":"concurrency","type":"nominal","title":"Concurrency","scale":{"scheme":"category10"}},"xOffset":{"field":"concurrency"},"tooltip":[{"field":"concurrency","type":"nominal"},{"field":"metric","type":"quantitative"},{"field":"percent","type":"quantitative"}]}}
```

The direction is internally coherent: more waiting supplied lead time, useful yield rose, and both lateness and false-residency failure share fell. The result supports the prefetch timing intuition, but concurrency also changed cache population and reuse, so it is not a pure one-variable causal decomposition.

Detailed native-cadence evidence:

- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Mechanism time series|Mechanism time series]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Scheduler and latency time series|Scheduler and latency time series]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Storage transfer and saturation time series|Storage, transfer, and saturation time series]]

## Configuration and run registry

| Role | Run | N | Runtime node | Image digest | Disposition |
|---|---|---:|---|---|---|
| Control | [beaf48bc…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/beaf48bcd79d46a1b155ba9af508ec2c?workspace=benchflow) | 0 | `…-6kl5z` | `097cffbd…` | valid control observation |
| Treatment | [6febe03b…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/6febe03b9d1f4b4e95f628a34e59c038?workspace=benchflow) | 100 | `…-mt46x` | `097cffbd…` | mechanism accepted; performance inconclusive |

Both request bodies carried `kv_transfer_params.abc_admission_prefetch=true`; N=0 disabled action at the manager. Both deployments used 256 GiB CPU capacity, node-local filesystem secondary storage, 64 read and 64 write threads, TP=8, one replica, 300 GiB `/dev/shm`, `gpu_memory_utilization=0.8`, and `max_model_len=131072`. `max_num_seqs` was not explicitly set. Each deployment used a cleaned node-local cache directory and 66 warm-up requests before profiling.

The runtime pod names identify H100 nodes, while the MLflow accelerator tag says H200. This metadata conflict remains unresolved. The same node/configuration pairing also existed at concurrency 32—control on `…-6kl5z`, treatment on `…-mt46x`—so node and treatment effects are confounded in both AgentX pairs.

## Validity and failure evidence

- Both AIPerf sessions ended normally, with empty error summaries and `was_cancelled=false`; pods and schedulers had zero restarts.
- The model log's filesystem `ERROR` records in the treatment are expected missing-file outcomes from blind assumed-resident proactive loads. They are represented by `load_failed`, not failed HTTP requests or engine failures.
- Control and treatment input volume differed by 102,889 tokens (0.13%) and output volume by 465 tokens (0.06%). This is close but not identical.
- Mean running/waiting depth was nearly matched across cells, but the treatment's five-minute-rate preemption telemetry averaged about 0.084/s versus 0.034/s for control. Because nodes differ, attribution is not possible.
- GPU KV occupancy stayed high: approximately 91–92% mean and about 99% at the p95 sample in both cells. This is a saturation regime, not a clean low-pressure latency benchmark.
- GPU core-utilization telemetry was empty. GPU KV occupancy must not be presented as GPU core utilization.

## Conclusions

### Measured observation

At concurrency 64, admission prefetch selected 123,419 blocks, submitted 33,103 blind NVMe loads, and recorded 19,508 useful copies. Compared with concurrency 32, the waiting population rose by roughly 26×, lateness fell from 98.50% to 42.39%, and useful yield rose from 1.16% to 15.81% of attempts. The end-to-end latency comparison remained mixed and showed no overall TTFT win.

### Inference

Queue lead time is a real control variable for this mechanism. When requests wait, a much larger fraction of proactive copies can complete before first demand. Higher concurrency also changes reuse, NVMe population, HBM pressure, and preemption behavior, so the mechanism improvement does not guarantee a latency improvement at the same operating point.

### Working conclusion

Phase 1 has enough evidence to accept the POC wiring and core intuition: proactive admission-time promotion can execute, can become useful, and becomes less late when lead time exists. N=100 first-N remains an intentionally naive experimental policy and is not validated for production. Performance benefit remains an open question, not a failed premise.

## Next-step discussion seed

The next decision should separate three questions that this run currently combines:

1. **How many blocks should be selected?** Run the planned `N ∈ {0,25,50,100,200}` sweep with repetitions.
2. **How much lead time is helpful without overload?** Use at least concurrency 32 and 64, and consider an intermediate value if it produces sustained but moderate waiting.
3. **Does the effect survive node variation?** Interleave or swap node assignment and run at least three repetitions per cell.

Mechanism gates should be evaluated before performance: useful/attempted, late/promoted, load_failed/promoted, redundant/attempted, unresolved/tracking state, then TTFT/E2E/ITL with preemptions and occupancy. A sensible Phase 1 exit criterion is repeatable useful promotion and reduced lateness without errors; a performance win belongs to the subsequent tuning question.

## Data provenance and limitations

AIPerf aggregates come from `benchmark/profile_export_aiperf.json`. Exact prefetch totals and ten-second progression come from the vLLM model log restricted to the AIPerf profiling interval. Prometheus time series use their native 15-second cadence and the same interval. NVMe metrics are node/device-wide. The comparison has one repetition per configuration, different nodes, slight workload-count drift, missing GPU-utilization telemetry, implicit `max_num_seqs`, and overlapping asynchronous outcomes.

Related report: [[Reports/2026-08-18 - AgentX Weka admission prefetch first exploratory run|concurrency-32 AgentX/Weka exploration]].