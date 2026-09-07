---
title: Selective loading calibration results - 2026-09-07
date: 2026-09-07
type: experiment-report
topic: Selective KV loading
experiment: Selective loading threshold calibration
project: Selective KV Loading and Offloading
status: conditionally-valid
model: nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
model_revision: unknown
vllm_version: v0.27.0-based selective-load build
vllm_image: quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v2
vllm_image_digest: sha256:a52851b8bf871f4726a57907414a13363b6dcc2a111651aa534a642e358bb6a2
router_commit: ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4
tensor_parallelism: 8
replicas: 1
gpu: 8x H100
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: default
cpu_bytes: 274877906944
offload_spec: CPU primary tier via OffloadingConnector
secondary_tier: none
secondary_tier_threads: not-applicable
dev_shm: 300Gi
workload: targeted paired sweep plus aiperf-agentx-inference-15m
random_seed: 20260707 for AgentX runs
duration:
  targeted_sweep_seconds: 1468.76
  agentx_profile_seconds: 900
cache_cleaning_state: seeded target, HBM eviction by unique-token churn, then two-second quiescence
configuration:
  targeted_treatments:
    load_allowed: omit kv_load_tiers
    forced_recompute: kv_load_tiers=[]
  router_offload_policy: preserve
  telemetry:
    enabled: true
    primary_sample_rate: 1.0
    overhead_control_sample_rate: 0.0
---

# Selective loading calibration results - 2026-09-07

## Executive summary

This experiment tested whether the vLLM binary selective-load contract works end to end and estimated the external-prefix size above which restoring KV from the CPU tier is faster than recomputation for this exact Nemotron 253B, TP=8, H100 deployment. A targeted paired sweep compared load allowed with `kv_load_tiers` omitted against forced recomputation with `kv_load_tiers=[]` at nine prefix sizes and three repetitions per treatment. Three 15-minute AgentX runs then exercised the two policy extremes under concurrency 32.

The binary mechanism worked exactly as intended in all 54 targeted probes: every one of the 27 load-allowed requests consumed external KV, and none of the 27 forced-recompute requests did. In the quiescent targeted sweep, 752 observed external tokens was the first bucket where loading won in all three repetitions; it saved a median 36.6 ms. Loading also won 3/3 at 1,008 tokens and every larger bucket. Because each bucket has only three repetitions and the 496-token bucket contained a 797 ms load outlier, 752 tokens is an observed crossing candidate, not a production-calibrated threshold. A conservative first router configuration should use **1,024 external reusable tokens**, then be validated under representative contention.

The concurrency-32 runs establish that unconditional recomputation is unsafe for this long-prefix workload. They do not provide a clean quantitative policy comparison: the fastest load-allowed run landed on a different H100 host, and the same-host load run changed telemetry sampling from 1.0 to 0.0. The telemetry-overhead hypothesis is therefore unresolved.

## Validity verdict: Conditionally valid

The functional opt-out result is **valid**. The quiescent threshold sweep is **valid for mechanism calibration on this exact deployment**, but it is under-replicated and does not model a pressured CPU transfer queue. The AgentX performance comparison is **directional only** because host placement and telemetry sampling were not both controlled. The telemetry overhead comparison is **invalid / inconclusive** because its two arms ran on different hosts.

## Main takeaways

- Measured: 27/27 load-allowed targeted probes reported external KV tokens; 0/27 forced-recompute probes did. This validates the router-to-vLLM binary opt-out path.
- Measured: loading won 3/3 repetitions beginning at 752 observed external tokens and continued to win 3/3 at 1,008, 2,032, 4,080, and 8,176 tokens.
- Measured: results below 752 tokens were noisy. Win rates were 2/3 at 128 and 368 tokens and 1/3 at 240 and 496 tokens. One 496-token load took 796.8 ms and lost by 458.3 ms.
- Decision: use 1,024 external reusable tokens as the conservative initial static router threshold. The experiment identifies 752 as the candidate crossing, while 1,024 supplies one tested bucket of safety margin.
- Measured: under AgentX concurrency 32, forced recomputation completed 145 requests at 0.154 requests/s, while the two load-allowed runs completed 489 at 0.520 requests/s and 176 at 0.187 requests/s. These raw outcomes must not be ranked quantitatively because deployment host and telemetry-sampling controls drifted.
- Measured: sampled load-resolution records show request-visible wait increasing sharply when pending scheduler jobs exceeded one: 1.12–2.25 seconds at two to four pending jobs versus mostly 38–583 ms at one. This is mechanism evidence that a token threshold alone cannot protect a saturated transfer path.
- Inference: the production policy should ultimately combine a static minimum-prefix threshold with a pressure veto based on transfer backlog or recent load-wait latency, with hysteresis and fail-open behavior when evidence is unavailable.

## Headline metrics

The declared macro baseline is forced recomputation. Deltas are intentionally `N/A`: neither load-allowed run held both node placement and telemetry sampling constant relative to that baseline.

| Configuration | Node | Telemetry sample | Completed | Errors / cancellations | Request throughput (req/s) | Output rate (tok/s) | TTFT p50 (ms) | TTFT p90 (ms) | TTFT p95 (ms) | E2E p50 (ms) | E2E p90 (ms) | ITL mean (ms) | Delta vs baseline |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|---|
| Forced recompute | psap-diadochos-gpu-h100-gjfjh | 1.0 | 145 | 0 / 0 | 0.154 | 62.39 | 2,833.1 | 49,310.1 | 72,773.1 | 61,321.4 | 267,643.2 | 241.55 | Baseline |
| Load allowed | psap-diadochos-gpu-h100-6kl5z | 1.0 | 489 | 0 / 0 | 0.520 | 355.60 | 678.6 | 4,033.3 | 6,281.3 | 11,457.1 | 46,097.5 | 35.78 | N/A: different host |
| Load allowed, sampling disabled | psap-diadochos-gpu-h100-gjfjh | 0.0 | 176 | 0 / 0 | 0.187 | 83.07 | 2,356.9 | 21,862.4 | 27,542.9 | 57,743.1 | 204,318.2 | 208.20 | N/A: telemetry setting differs |

All AgentX runs used the same model, image digest, TP=8, one replica, 900-second profiling duration, concurrency 32, dataset `semianalysisai/cc-traces-weka-062126`, cache bust mode `first_turn_prefix`, maximum context 128,000, and seed 20260707. No request errors or cancellations were observed.

## Targeted threshold evidence

The targeted runner seeded a known prefix, churned approximately 1.83–1.856 million unique tokens to evict it from HBM while retaining it in the approximately 2.06-million-token CPU cache, waited two seconds for quiescence, then alternated load-allowed and forced-recompute probes. The requested sizes differ from observed external tokens because a 16-token template prefix remained locally cached.

| Requested prefix | Observed external tokens | Median load (ms) | Median recompute (ms) | Median benefit, recompute - load (ms) | Load wins | Repetitions |
|---:|---:|---:|---:|---:|---:|---:|
| 128 | 128 | 304.8 | 307.5 | 2.7 | 2 | 3 |
| 256 | 240 | 345.1 | 348.2 | -8.1 | 1 | 3 |
| 384 | 368 | 295.5 | 375.1 | 79.6 | 2 | 3 |
| 512 | 496 | 335.7 | 338.4 | -40.6 | 1 | 3 |
| 768 | 752 | 314.5 | 346.5 | 36.6 | 3 | 3 |
| 1,024 | 1,008 | 299.1 | 362.1 | 68.1 | 3 | 3 |
| 2,048 | 2,032 | 418.6 | 497.5 | 73.6 | 3 | 3 |
| 4,096 | 4,080 | 501.8 | 796.4 | 294.6 | 3 | 3 |
| 8,192 | 8,176 | 592.8 | 1,339.4 | 725.7 | 3 | 3 |

Figure 1 shows every paired repetition at the finest available categorical grain. Positive values mean loading was faster. The blue rule marks the observed 752-token all-win crossing; the orange rule marks the recommended 1,024-token configuration boundary.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Paired latency benefit by external reusable-prefix size","width":720,"height":380,"data":{"values":[{"tokens":128,"rep":1,"benefit_ms":-57.537},{"tokens":128,"rep":2,"benefit_ms":2.693},{"tokens":128,"rep":3,"benefit_ms":63.391},{"tokens":240,"rep":1,"benefit_ms":-34.311},{"tokens":240,"rep":2,"benefit_ms":-8.093},{"tokens":240,"rep":3,"benefit_ms":44.894},{"tokens":368,"rep":1,"benefit_ms":148.212},{"tokens":368,"rep":2,"benefit_ms":79.582},{"tokens":368,"rep":3,"benefit_ms":-98.368},{"tokens":496,"rep":1,"benefit_ms":-458.339},{"tokens":496,"rep":2,"benefit_ms":-40.597},{"tokens":496,"rep":3,"benefit_ms":113.400},{"tokens":752,"rep":1,"benefit_ms":25.718},{"tokens":752,"rep":2,"benefit_ms":50.623},{"tokens":752,"rep":3,"benefit_ms":36.563},{"tokens":1008,"rep":1,"benefit_ms":4.049},{"tokens":1008,"rep":2,"benefit_ms":151.298},{"tokens":1008,"rep":3,"benefit_ms":68.101},{"tokens":2032,"rep":1,"benefit_ms":40.337},{"tokens":2032,"rep":2,"benefit_ms":146.244},{"tokens":2032,"rep":3,"benefit_ms":73.551},{"tokens":4080,"rep":1,"benefit_ms":289.827},{"tokens":4080,"rep":2,"benefit_ms":352.881},{"tokens":4080,"rep":3,"benefit_ms":294.604},{"tokens":8176,"rep":1,"benefit_ms":926.619},{"tokens":8176,"rep":2,"benefit_ms":714.873},{"tokens":8176,"rep":3,"benefit_ms":725.723}]},"layer":[{"mark":{"type":"rule","color":"#666666","strokeDash":[4,4]},"encoding":{"y":{"datum":0}}},{"mark":{"type":"rule","color":"#1f77b4","strokeDash":[6,4]},"encoding":{"x":{"datum":752}}},{"mark":{"type":"rule","color":"#ff7f0e","strokeDash":[2,3]},"encoding":{"x":{"datum":1024}}},{"mark":{"type":"point","filled":true,"size":85,"opacity":0.8},"encoding":{"x":{"field":"tokens","type":"quantitative","title":"External reusable tokens (tokens)","scale":{"type":"log"}},"y":{"field":"benefit_ms","type":"quantitative","title":"Recompute minus load wall time (ms)"},"color":{"field":"rep","type":"nominal","title":"Repetition","scale":{"scheme":"category10"}},"tooltip":[{"field":"tokens","type":"quantitative","title":"External tokens"},{"field":"rep","type":"ordinal","title":"Repetition"},{"field":"benefit_ms","type":"quantitative","title":"Benefit (ms)"}]}}]}
```

The point cloud is not monotonic below 752 tokens, which is expected when fixed scheduling and lookup costs are comparable to the recomputation time. The five consecutive all-win buckets from 752 through 8,176 tokens are the useful signal; three repetitions are still insufficient for confidence intervals or a production acceptance claim.

## Macro workload evidence

Figure 2 shows raw TTFT percentiles for the three AgentX runs. It is diagnostic, not a controlled ranking: labels expose the host and telemetry-setting differences.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. AgentX TTFT by policy arm and run environment","width":700,"height":360,"data":{"values":[{"configuration":"Recompute / gjfjh / sample 1","percentile":"p50","latency_ms":2833.129},{"configuration":"Recompute / gjfjh / sample 1","percentile":"p90","latency_ms":49310.138},{"configuration":"Load / 6kl5z / sample 1","percentile":"p50","latency_ms":678.640},{"configuration":"Load / 6kl5z / sample 1","percentile":"p90","latency_ms":4033.334},{"configuration":"Load / gjfjh / sample 0","percentile":"p50","latency_ms":2356.851},{"configuration":"Load / gjfjh / sample 0","percentile":"p90","latency_ms":21862.351}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Policy / node / telemetry sample rate","axis":{"labelAngle":-20}},"xOffset":{"field":"percentile"},"y":{"field":"latency_ms","type":"quantitative","title":"Time to first token (ms)","scale":{"zero":true}},"color":{"field":"percentile","type":"nominal","title":"Percentile","scale":{"scheme":"category10"}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"percentile","type":"nominal"},{"field":"latency_ms","type":"quantitative","title":"TTFT (ms)"}]}}
```

The same-host raw comparison is consistent with loading helping the long-prefix workload in aggregate: p90 TTFT fell from 49.3 seconds to 21.9 seconds and completions increased from 145 to 176. It is not causal proof because telemetry sampling changed. Request-level matching reinforces that limitation: the median matched TTFT difference was +10.1 ms for load minus recompute even though the mean was -6.20 seconds, indicating that a minority of heavily delayed requests dominated the aggregate gain and only a small number of requests actually performed external loads.

Figure 3 shows raw request throughput. The 0.520 requests/s result cannot be attributed solely to loading because it ran on `6kl5z`, while the other two ran on `gjfjh`.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3. AgentX request throughput by policy arm and run environment","width":700,"height":340,"data":{"values":[{"configuration":"Recompute / gjfjh / sample 1","throughput":0.154255},{"configuration":"Load / 6kl5z / sample 1","throughput":0.520212},{"configuration":"Load / gjfjh / sample 0","throughput":0.187234}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Policy / node / telemetry sample rate","axis":{"labelAngle":-20}},"y":{"field":"throughput","type":"quantitative","title":"Request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"scheme":"category10"}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"throughput","type":"quantitative","title":"Throughput (req/s)"}]}}
```

## Transfer pressure evidence

Figure 4 uses all 20 request-resolution records extracted from the sample-rate-1 load run. These records could not be reliably joined to the AIPerf profiling phase because the logged completion identifiers and exported request identifiers used incompatible forms, so they may include warmup and are used only as mechanism evidence. Point size represents external tokens.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4. Request-visible load wait versus pending transfer jobs","width":680,"height":360,"data":{"values":[{"jobs":1,"wait_ms":168.096,"external_tokens":32752},{"jobs":1,"wait_ms":90.115,"external_tokens":29024},{"jobs":1,"wait_ms":95.356,"external_tokens":34352},{"jobs":1,"wait_ms":159.212,"external_tokens":49424},{"jobs":1,"wait_ms":271.372,"external_tokens":88768},{"jobs":1,"wait_ms":242.630,"external_tokens":110144},{"jobs":1,"wait_ms":192.664,"external_tokens":98928},{"jobs":3,"wait_ms":2250.907,"external_tokens":20048},{"jobs":4,"wait_ms":1606.216,"external_tokens":51840},{"jobs":1,"wait_ms":140.033,"external_tokens":71168},{"jobs":2,"wait_ms":1117.806,"external_tokens":23584},{"jobs":1,"wait_ms":110.847,"external_tokens":41904},{"jobs":1,"wait_ms":163.798,"external_tokens":90368},{"jobs":1,"wait_ms":38.160,"external_tokens":128},{"jobs":1,"wait_ms":71.579,"external_tokens":15216},{"jobs":1,"wait_ms":71.414,"external_tokens":8768},{"jobs":1,"wait_ms":208.451,"external_tokens":114112},{"jobs":1,"wait_ms":583.113,"external_tokens":51232},{"jobs":1,"wait_ms":98.613,"external_tokens":32320},{"jobs":1,"wait_ms":212.479,"external_tokens":116976}]},"mark":{"type":"point","filled":true,"opacity":0.75},"encoding":{"x":{"field":"jobs","type":"quantitative","title":"Pending scheduler transfer jobs (jobs)","scale":{"zero":true}},"y":{"field":"wait_ms","type":"quantitative","title":"Request-visible load wait (ms)","scale":{"zero":true}},"size":{"field":"external_tokens","type":"quantitative","title":"External tokens (tokens)"},"color":{"field":"jobs","type":"nominal","title":"Pending jobs","scale":{"scheme":"category10"}},"tooltip":[{"field":"jobs","type":"quantitative","title":"Pending jobs"},{"field":"wait_ms","type":"quantitative","title":"Load wait (ms)"},{"field":"external_tokens","type":"quantitative","title":"External tokens"}]}}
```

At one pending job, observed wait was usually below 300 ms, with one 583 ms point. At two to four pending jobs, all three observations exceeded one second. The sample is small and observational, but it directly demonstrates why queue state belongs in a later dynamic gate: a large reusable prefix may still be a poor load candidate when transfer work is backlogged.

## Mechanism and telemetry interpretation

The custom vLLM metrics needed for pressure-aware follow-up are present: scheduler load-wait and resolution histograms, load tokens, transfer turnaround/copy/non-copy timing, pending jobs and bytes, lookup delays, CPU cache allocation failures, and prompt tokens by source. `transfer_turnaround` must not be treated as pure DMA or queue time because it includes completion polling and scheduler observation. The request-visible load-wait metric is the appropriate outcome for a dynamic veto; pending jobs or bytes provide a cheaper leading indicator.

The first dynamic design should preserve loading only when both conditions hold:

$$
external\_reusable\_tokens \ge 1024
$$

and

$$
transfer\_pressure < pressure\_veto
$$

Use an EWMA or short-window p90 of scheduler load wait together with pending bytes/jobs, add separate enter and exit thresholds for hysteresis, and preserve vLLM's default loading when evidence is missing or stale. CPU-cache allocation failures should be a hard veto. A threshold on CPU utilization alone is not sufficient because it does not measure memory-copy queueing or NUMA/PCIe contention directly.

## Validity and failure evidence

- All three MLflow runs finished successfully and recorded zero request errors and cancellations.
- The targeted sweep completed all 54 planned probes and produced exact policy-source separation.
- No secondary storage tier was configured, so this result applies only to CPU-to-HBM restoration.
- GPU-core telemetry was unavailable in the AIPerf console export. Prometheus artifacts include GPU and cache series, but no cross-run time-series claim is made here because the macro comparison was already invalidated by placement drift.
- The sample-rate-0 benchmark client's final polling step experienced a DNS interruption, but the remote benchmark completed successfully and its artifacts were recovered from the results PVC. This does not invalidate the run measurements.
- The telemetry-overhead experiment is inconclusive. Sample-rate 1 ran on `6kl5z`; sample-rate 0 ran on `gjfjh`. The large performance difference is evidence of a placement confound, not telemetry cost.
- Only three repetitions were collected per targeted bucket, below the test plan's 20–30 sample target. No confidence interval is reported.
- The two-second quiescence after churn intentionally estimates the low-contention crossing. It does not establish the crossing under production transfer pressure.

## Run registry

| Treatment | MLflow run | Run name | Acceptance |
|---|---|---|---|
| Forced recompute, sample 1.0 | [38c2c844b9dc422b83e518d93543441a](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/403/runs/38c2c844b9dc422b83e518d93543441a?workspace=benchflow) | selective-mare-691 | Accepted as functional boundary; raw performance only |
| Load allowed, sample 1.0 | [f9e85c2f754e4912890a9953ae3a2722](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/403/runs/f9e85c2f754e4912890a9953ae3a2722?workspace=benchflow) | thoughtful-croc-793 | Accepted as mechanism evidence; cross-host performance comparison rejected |
| Load allowed, sample 0.0 | [ebd384ce3c9b438d955bf9788d765f00](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/403/runs/ebd384ce3c9b438d955bf9788d765f00?workspace=benchflow) | whimsical-auk-441 | Accepted as same-host raw observation; telemetry overhead comparison rejected |

The targeted sweep is not an MLflow run. Its raw request-level artifact is preserved locally at `.calibration-artifacts/2026-09-07-targeted-threshold-sweep.json`; macro exports and native Prometheus artifacts are under the corresponding `.calibration-artifacts/2026-09-07-agentx-c32-*` directories.

## Conclusion

This batch establishes the end-to-end binary control and supplies a useful initial configuration, but it does not complete production calibration. Configure the next router prototype with a 1,024 external-token minimum, scoped to this deployment fingerprint and failing open when prefix evidence is absent or incompatible. Do not yet add a hard dynamic veto from these 20 observational load records.

The smallest decisive next experiment is a same-node, sample-rate-1 paired sweep around 512, 768, 1,024, 1,536, and 2,048 external tokens with at least 20 valid repetitions at concurrency 1 and at representative transfer pressure. Then run an AgentX policy comparison among always load, the 1,024-token threshold, and always recompute while pinning the model pod to one node and keeping telemetry sampling fixed. Collect router decision reasons and a request ID that joins router, vLLM resolution, and AIPerf records.

If the crossing moves materially with pending transfer work, add a pressure veto driven first by pending jobs/bytes and recent request-visible load wait. If it remains stable, retain the simpler static threshold.

## Related

- [[00 - Index]]
- [[04 - vLLM selective load - problem statement]]
- [[05 - 2026-09-04 vLLM binary opt-out cluster validation]]
- [[06 - 2026-09-05 naive llm-d-router static gating prototype]]
- [[07 - Selective loading calibration test plan]]

## Provenance

Direct cluster execution on 2026-09-07, local AIPerf and Prometheus artifacts collected with BenchFlow, request-level targeted sweep output, vLLM structured telemetry, and MLflow run metadata verified with the MLflow CLI. No data was invented or downsampled in the charts; Figure 1 contains every paired targeted repetition and Figure 4 contains every extracted load-resolution record.