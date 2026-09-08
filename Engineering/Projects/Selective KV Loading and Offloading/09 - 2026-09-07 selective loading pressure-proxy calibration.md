---
title: Selective loading pressure-proxy calibration and policy proposal
date: 2026-09-07
type: experiment-report
topic: Selective KV loading
experiment: CPU-to-GPU load-pressure proxy calibration
project: Selective KV Loading and Offloading
status: conditionally-valid
model: nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
model_revision: unknown
vllm_version: v0.27.0-based selective-load build
vllm_image: quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v2
vllm_image_digest: sha256:a52851b8bf871f4726a57907414a13363b6dcc2a111651aa534a642e358bb6a2
vllm_source_base: e50f7d36960980c0c89651ffd0ce281a9fb8a466
router_commit: ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4 plus uncommitted threshold-policy prototype
tensor_parallelism: 8
replicas: 1
gpu: 8x H100
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: default
cpu_bytes: 274877906944
offload_spec: CPU primary tier via OffloadingConnector
secondary_tier: none
dev_shm: 300Gi
workload: targeted CPU-load pressure probes
random_seed:
  corrected_matrix: 20260910
  focused_separate_arms: 20260912
  simultaneous_pairs: 20260913
cache_cleaning_state: deterministic priming followed by unique 16k-token churn
configuration:
  target_external_tokens: approximately 1008 to 1024
  background_prompt_tokens: 8192
  background_concurrency: [2, 4, 8, 16, 32]
  pressure_target_delay_seconds: 0.02
  telemetry_sample_rate: 1.0
  epp_metrics_refresh_interval_ms: 50
---

# Selective loading pressure-proxy calibration and policy proposal

## Executive summary

This experiment asked which vLLM signal the llm-d Endpoint Picker should use to decide whether loading an externally reusable prefix from CPU into HBM is likely to cost more than recomputing it. The reusable target was held near 1,024 tokens so the experiment varied transfer and scheduler pressure rather than recalibrating the token floor. The deployment was one Nemotron Ultra 253B FP8 replica on eight H100 GPUs with tensor parallelism 8, 256 GiB of CPU KV capacity, and no secondary tier.

The strongest tested instantaneous proxy for request-specific CPU-to-GPU load wait was `vllm:kv_offload_worker_pending_bytes`: across nine verified CPU-load targets, its Spearman correlation with load wait was 0.82. A nonzero value always represented one approximately 128 MiB per-rank job and increased median load wait from 6.69 ms to 10.08 ms in the focused separate-arm run. Recent submitted-byte rate was weaker, with correlations around 0.52 to 0.56 depending on the window. Completed-byte rate and active-copy duty were weaker still.

Pending bytes is not sufficient for the routing decision. In the same-pressure experiment, targets sometimes experienced large delays before their load allocation while the worker gauge was zero. One load waited 134 ms even though the router-time sample had no pending worker bytes. Another load-enabled request reached its KV decision approximately 1.12 seconds after its paired recompute request. These costs are represented by vLLM's scheduler load-wait or resolution timing, not by active DMA bytes alone.

The proposal is therefore to keep the current pending-bytes EWMA in observe-only mode and add EPP-consumable cumulative scheduler load-wait counters in vLLM. The router should derive a time-aware EWMA of recent request-visible load wait and compare it with the calibrated recomputation value of the external tokens. Pending bytes and effective transfer bandwidth should remain secondary evidence and an optional fast veto. The current `maxPendingLoadBytesEWMA` enforcement should not be enabled from this batch.

## Validity verdict: Conditionally valid

The proxy ranking for request-visible load wait is conditionally valid for this deployment. All nine observations used in that ranking were verified `loaded` resolutions with approximately 1,024 external tokens, and the request-specific telemetry join was preserved in the artifact. The sample is small and covers only zero or one pending worker job.

The batch is invalid for selecting a production pending-byte threshold. Only 3 of 12 separate-arm pairs passed both residency and background-volume matching. The eight same-pressure pairs all had valid load and recompute outcomes, but simultaneous target submission introduced backend admission-order skew from -284 ms to +1,120 ms. That skew dominated raw wall-time differences. The experiment establishes which telemetry is informative and which gaps remain; it does not establish a production pressure cutoff.

## Main takeaways

- Measured: the EPP requests each backend `/metrics` endpoint at approximately 50 ms intervals under its default refresh configuration. Earlier laptop-side samples around 500 ms were port-forward overhead and were not representative of in-cluster collection.
- Measured: pending worker bytes had the strongest relationship with per-request load wait among the tested exported Gauge/Counter candidates, with Spearman correlation 0.82 across nine verified loads.
- Measured: zero pending bytes produced a 6.69 ms median load wait; one approximately 128 MiB job produced a 10.08 ms median in the focused run. A separate pressured episode with one pending job produced a 134 ms load wait, showing that equal worker backlog can hide very different scheduler delay.
- Measured: recent submitted-byte rate was a useful but weaker persistent activity signal. It remained visible after the instantaneous pending gauge returned to zero, but it did not reliably predict load-versus-recompute benefit by itself.
- Measured: the current EPP metric extractor accepts Prometheus Gauge and Counter families and does not consume Histogram families. Targeted extractor and selective-KV plugin tests passed.
- Inference: the desired decision variable is estimated request-visible load cost, including scheduling and completion observation, compared with the recomputation value of the reusable prefix. DMA bandwidth and pending bytes are components of that estimate rather than the objective.
- Decision: retain the 1,024-token minimum as a separate deployment-specific control, leave pressure enforcement observe-only, and add standalone cumulative scheduler load-wait counters to vLLM before enabling a pressure gate.

## Headline evidence

| Evidence set | Accepted samples | Result | Decision use |
|---|---:|---|---|
| Quiescent 1,024-token control | 20 pairs | Loading won 17/20; median recompute-minus-load benefit 106.5 ms | Defines the approximate load-cost budget at this token floor, not a pressure metric threshold |
| Focused request-specific loads | 9 loads | Pending-bytes correlation with load wait 0.82; zero-pending median 6.69 ms; nonzero-pending median 10.08 ms | Ranks pressure proxies |
| Separate-arm pressured A/B | 3 of 12 pairs | Too few pairs survived CPU-residency and background-volume matching | No threshold selected |
| Same-pressure simultaneous A/B | 8 of 8 outcome-valid pairs | Raw recompute won 6/8; backend KV-decision admission skew dominated the comparison | Exposes hidden scheduler cost and test-design bias |
| Same-pressure post-decision diagnostic | 8 pairs | After removing decision-admission skew, loading won 7/8 with a median 63.5 ms benefit | Mechanism evidence only; post-hoc adjustment is not a policy benchmark |

## Proxy evidence

Figure 1 contains every verified load target from the focused separate-arm batch. Pending state is sampled immediately before the target's own rank-0 transfer submission. No observations were downsampled.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Request-visible load wait by pending worker state","width":680,"height":360,"data":{"values":[{"repetition":0,"concurrency":8,"pending_state":"Zero pending","pending_bytes":0,"load_wait_ms":5.884},{"repetition":0,"concurrency":4,"pending_state":"One job (~128 MiB)","pending_bytes":134217728,"load_wait_ms":8.821},{"repetition":0,"concurrency":16,"pending_state":"Zero pending","pending_bytes":0,"load_wait_ms":6.904},{"repetition":1,"concurrency":4,"pending_state":"One job (~128 MiB)","pending_bytes":134217728,"load_wait_ms":11.020},{"repetition":1,"concurrency":8,"pending_state":"Zero pending","pending_bytes":0,"load_wait_ms":5.421},{"repetition":1,"concurrency":16,"pending_state":"Zero pending","pending_bytes":0,"load_wait_ms":7.686},{"repetition":2,"concurrency":4,"pending_state":"One job (~128 MiB)","pending_bytes":134217728,"load_wait_ms":10.082},{"repetition":2,"concurrency":8,"pending_state":"Zero pending","pending_bytes":0,"load_wait_ms":6.470},{"repetition":2,"concurrency":16,"pending_state":"Zero pending","pending_bytes":0,"load_wait_ms":7.457}]},"mark":{"type":"point","filled":true,"size":100,"opacity":0.8},"encoding":{"x":{"field":"pending_state","type":"nominal","title":"Rank-0 worker state before target submission"},"y":{"field":"load_wait_ms","type":"quantitative","title":"Scheduler load wait (ms)","scale":{"zero":true}},"color":{"field":"concurrency","type":"nominal","title":"Background concurrency","scale":{"scheme":"category10"}},"tooltip":[{"field":"repetition","type":"ordinal","title":"Repetition"},{"field":"concurrency","type":"quantitative","title":"Concurrency"},{"field":"pending_bytes","type":"quantitative","title":"Pending bytes"},{"field":"load_wait_ms","type":"quantitative","title":"Load wait (ms)"}]}}
```

The two groups separate in this small sample, but a byte threshold cannot be inferred from them: all nonzero samples have the same one-job magnitude, and another same-pressure cell with the same approximate backlog waited 134 ms. Pending bytes detects active worker work but not all time spent before allocation or completion.

Figure 2 ranks the tested request-arrival or pre-submission proxies by Spearman correlation with request-specific load wait. Every correlation uses the same nine verified load targets. Windows refer to trailing telemetry before target submission.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Pressure-proxy correlation with request-visible load wait","width":700,"height":360,"data":{"values":[{"metric":"Pending bytes","spearman":0.822},{"metric":"Submitted bytes/s, 1 s","spearman":0.562},{"metric":"Submitted bytes/s, 100 ms","spearman":0.560},{"metric":"Submitted bytes/s, 50 ms","spearman":0.522},{"metric":"Submitted bytes/s, 500 ms","spearman":0.441},{"metric":"Completed bytes/s, 100 ms","spearman":0.279},{"metric":"Completed bytes/s, 500 ms","spearman":0.222},{"metric":"Copy duty, 500 ms","spearman":0.220},{"metric":"Completed bytes/s, 1 s","spearman":0.115},{"metric":"Copy duty, 1 s","spearman":0.067}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"spearman","type":"quantitative","title":"Spearman correlation with load wait","scale":{"zero":true,"domain":[0,1]}},"y":{"field":"metric","type":"nominal","title":"Candidate proxy","sort":"-x"},"color":{"field":"metric","type":"nominal","legend":null,"scale":{"scheme":"category10"}},"tooltip":[{"field":"metric","type":"nominal"},{"field":"spearman","type":"quantitative","title":"Spearman correlation"}]}}
```

The ranking supports pending bytes as a fast signal and submitted bytes as a short-lived activity signal. It does not show that either signal is a sufficient classifier for the end-to-end decision.

## Same-pressure comparison and admission bias

Figure 3 shows all eight same-pressure pairs. Positive benefit means loading was faster. `Post-decision diagnostic` subtracts the observed difference in when the two simultaneously launched targets reached their KV decision. This adjustment is shown only to expose admission-order bias; it is not a valid replacement for an independently randomized A/B benchmark.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3. Raw and post-decision load benefit in same-pressure pairs","width":720,"height":380,"data":{"values":[{"cell":"r0-c4","measure":"Raw wall time","benefit_ms":-52.900},{"cell":"r0-c4","measure":"Post-decision diagnostic","benefit_ms":52.471},{"cell":"r0-c8","measure":"Raw wall time","benefit_ms":-26.571},{"cell":"r0-c8","measure":"Post-decision diagnostic","benefit_ms":-20.255},{"cell":"r0-c16","measure":"Raw wall time","benefit_ms":355.013},{"cell":"r0-c16","measure":"Post-decision diagnostic","benefit_ms":70.967},{"cell":"r0-c32","measure":"Raw wall time","benefit_ms":-1058.046},{"cell":"r0-c32","measure":"Post-decision diagnostic","benefit_ms":61.891},{"cell":"r1-c4","measure":"Raw wall time","benefit_ms":-68.543},{"cell":"r1-c4","measure":"Post-decision diagnostic","benefit_ms":41.657},{"cell":"r1-c8","measure":"Raw wall time","benefit_ms":-91.876},{"cell":"r1-c8","measure":"Post-decision diagnostic","benefit_ms":107.513},{"cell":"r1-c16","measure":"Raw wall time","benefit_ms":-32.455},{"cell":"r1-c16","measure":"Post-decision diagnostic","benefit_ms":69.132},{"cell":"r1-c32","measure":"Raw wall time","benefit_ms":213.884},{"cell":"r1-c32","measure":"Post-decision diagnostic","benefit_ms":65.133}]},"layer":[{"mark":{"type":"rule","color":"#666666","strokeDash":[4,4]},"encoding":{"y":{"datum":0}}},{"mark":{"type":"bar"},"encoding":{"x":{"field":"cell","type":"nominal","title":"Repetition and background concurrency"},"xOffset":{"field":"measure"},"y":{"field":"benefit_ms","type":"quantitative","title":"Recompute minus load wall time (ms)"},"color":{"field":"measure","type":"nominal","title":"Measurement","scale":{"scheme":"category10"}},"tooltip":[{"field":"cell","type":"nominal"},{"field":"measure","type":"nominal"},{"field":"benefit_ms","type":"quantitative","title":"Benefit (ms)"}]}}]}
```

The gap between raw and adjusted bars is the experiment's main validity warning. Simultaneous pairing made pressure identical but caused the two targets to compete for backend admission. A future end-to-end threshold benchmark must use independently randomized target treatment while sustaining a repeatable background stream, and it must record the pressure snapshot at EPP arrival.

## Metric semantics and EPP compatibility

The current vLLM telemetry exposes these useful families:

| Metric | Type | What it measures | EPP usability |
|---|---|---|---|
| `vllm:kv_offload_worker_pending_bytes{direction="load",rank="0"}` | Gauge | Payload bytes in submitted rank-0 worker jobs awaiting observed completion | Directly consumable |
| `vllm:kv_offload_worker_pending_jobs{direction="load",rank="0"}` | Gauge | Submitted rank-0 jobs awaiting completion | Directly consumable, but redundant with bytes in this transfer geometry |
| `vllm:kv_offload_load_bytes_total` | Counter | Completed load bytes, aggregated by the API process | Directly consumable; router can calculate deltas/rate |
| `vllm:kv_offload_load_time_total` | Counter | Active load-copy time | Directly consumable; omits queueing and scheduling |
| `vllm:kv_offload_scheduler_load_wait_seconds` | Histogram | Allocation to all-worker completion observation | Best direct cost measurement, but not consumable by the current extractor |
| `vllm:kv_offload_scheduler_resolution_seconds` | Histogram | Lookup start through load/miss/disable resolution | Captures a broader request-visible cost, but not consumable by the current extractor |

The EPP's custom extractor returns scalar values only for Prometheus Gauge and Counter metric families. The generated `_sum` and `_count` text lines of a Histogram remain members of a Histogram family and cannot be selected as independent Counters by the current extractor.

## Proposed policy architecture

### Immediate router state

Do not enable the current `maxPendingLoadBytesEWMA` threshold in production. Keep it as observe-only instrumentation while collecting representative traces. Continue to enforce only the already calibrated token floor and explicit operator policies.

The research configuration can retain detailed diagnostics, but the intended operator surface should be small:

```yaml
loadPolicy: adaptive-threshold
minExternalReusableTokens: 1024
maxEstimatedLoadWaitMilliseconds: 80
```

Metric names, EWMA half-life, hysteresis, staleness, and probing should have tested defaults or remain advanced fields. They should not all be mandatory deployment knobs.

### vLLM metric additions

Expose the scheduler load-wait Histogram observations as standalone monotonic Counters in addition to the existing Histogram:

```text
vllm:kv_offload_scheduler_load_wait_seconds_total{outcome="loaded"}
vllm:kv_offload_scheduler_load_wait_events_total{outcome="loaded"}
```

The counters must be recorded once per logical all-worker load, not once per tensor-parallel rank. Keeping the `outcome` label permits the router to select completed loads while preserving diagnostic outcomes. The existing Histogram remains useful for Prometheus analysis.

Optionally add scheduler-side gauges for logical load work that has been allocated but not fully observed complete:

```text
vllm:kv_offload_scheduler_pending_load_jobs
vllm:kv_offload_scheduler_pending_load_tokens
```

These gauges describe scheduler-visible logical work and avoid tensor-parallel rank aggregation. They are preferable to inventing a universal byte threshold when transfer geometry changes across models, KV dtypes, and parallelism.

### EPP derivation

For two consecutive metric snapshots, derive the recent mean observed load wait:

$$
W_{sample} = \frac{\Delta load\_wait\_seconds\_total}{\Delta load\_wait\_events\_total}
$$

Use a time-aware EWMA so behavior is independent of request rate and duplicate policy evaluations:

$$
\alpha(\Delta t) = 1 - e^{-\Delta t / \tau}
$$

$$
W_{ewma} \leftarrow \alpha(\Delta t) W_{sample} + (1-\alpha(\Delta t))W_{ewma}
$$

The current request-count-based alpha should not be the final interface because an alpha of 0.2 represents different wall-clock memory at different request rates.

Estimate active physical queue cost from existing counters and gauges when their deltas are valid:

$$
BW_{effective} = \frac{\Delta load\_bytes\_total}{\Delta load\_time\_total}
$$

$$
W_{pending} = \frac{pending\_load\_bytes}{BW_{effective}}
$$

Then use a conservative cost estimate:

$$
W_{estimated} = \max(W_{ewma}, W_{pending})
$$

The first candidate activation threshold for this deployment is 80 ms. It is derived from the quiescent 1,024-token median benefit of 106.5 ms with approximately 25 ms of safety margin. It is a hypothesis for the next controlled experiment, not a production-calibrated value.

### Decision flow

```text
if capability or prefix evidence is missing/stale:
    preserve vLLM loading
else if external_reusable_tokens < 1024:
    disable external loading
else if pressure evidence is missing/stale:
    preserve vLLM loading and record observe-only decision
else if estimated_load_wait_ms >= 80:
    disable external loading
else:
    preserve vLLM loading
```

Use hysteresis around the pressure boundary, for example disable at 80 ms and re-enable below 60 ms. A gate that suppresses every load stops producing new load-wait observations, so it must either decay stale pressure toward fail-open behavior or allow a small configured fraction of probe loads. Metric age and decision reason must be logged.

## Why the current pending-byte policy should remain observe-only

The current uncommitted router prototype computes an EWMA of `pending_load_bytes` and compares it with `maxPendingLoadBytesEWMA`. The implementation and unit tests are sound for that contract, and the targeted Go tests pass. The experiment does not validate the contract as the final policy:

1. The observed pending magnitude was almost binary: zero or one approximately 128 MiB rank-local job. It did not supply enough levels to locate a byte cutoff.
2. Equal pending magnitude produced load waits from roughly 9 ms to 134 ms because scheduling and completion observation dominate some episodes.
3. The gauge can return to zero between transfers even while recent or upstream work makes the load-enabled path slow.
4. A per-rank byte threshold changes meaning with tensor parallelism, KV dtype, chunk size, and model geometry.
5. A request-count EWMA alpha changes its effective time horizon with traffic rate.

Pending bytes remains useful as an observable and as an input to an estimated wait calculation. It should not be the sole production gate.

## Rejected and incomplete runs

- The first pressure runner appended a cell inside the background-prefix construction loop. Its output is preserved locally as `.calibration-artifacts/2026-09-07-selective-load-proxy-c32x8k.json` and is rejected for A/B latency comparison.
- The corrected 40-cell matrix is preserved as `.calibration-artifacts/2026-09-07-selective-load-proxy-matrix.json`. Container log rotation retained request-specific telemetry for only the final two cells, so its proxy join is incomplete. It was used to improve the collection method, not to set a threshold.
- The focused separate-arm run with 94 churn requests is preserved as `.calibration-artifacts/2026-09-07-selective-load-pressure-focused.json`. Its targets remained in HBM or were not externally available and the run was stopped after four cells.
- The 116-churn focused run is preserved as `.calibration-artifacts/2026-09-07-selective-load-pressure-focused-v2.json`. Request telemetry is complete, but only 3 of 12 A/B pairs passed the background-volume match.
- The same-pressure run is preserved as `.calibration-artifacts/2026-09-07-selective-load-pressure-simultaneous.json`. All eight target pairs are outcome-valid; end-to-end ranking is confounded by simultaneous admission ordering.

## Verification

| Check | Result |
|---|---|
| Live vLLM target outcome joins | Passed for the focused and simultaneous artifacts |
| Model and EPP pods after experiment | Both Running, Ready, zero restarts |
| vLLM metric unit tests | 29 passed |
| vLLM worker and worker-metadata tests | 13 passed, 2 skipped |
| vLLM broad scheduler suite | Inconclusive locally: 64 failures and 70 passes before interruption; failures attempted Hugging Face DNS access in the restricted environment |
| llm-d-router metric extractor tests | Passed |
| llm-d-router selective-KV plugin tests | Passed |

The broad vLLM scheduler result is an environment limitation rather than evidence of a code assertion failure: the captured failures were `httpx.ConnectError` while constructing model configuration through Hugging Face Hub. The live image completed all targeted requests and the pod did not restart.

## Next experiment

1. Add the two standalone scheduler load-wait Counters to vLLM and preserve the Histogram.
2. Update the router prototype to calculate a time-aware load-wait EWMA from Counter deltas. Retain pending bytes and byte/time counters as diagnostics. Keep decisions observe-only.
3. Run a sustained background stream rather than finite synchronized bursts. Randomize a single target's load/recompute treatment per trial so targets never compete with each other.
4. Require every load trial to resolve with approximately 1,024 external tokens and every recompute trial to resolve as disabled. Reject trials with mismatched background external-token rate at EPP arrival.
5. Sweep observed load-wait regimes around 40, 60, 80, 100, 125, and 150 ms with at least 20 valid independent targets per treatment and regime.
6. Select the boundary from paired or propensity-matched end-to-end TTFT benefit, then validate it on the representative AgentX trace against always load. Include an exploration fraction so the adaptive policy continues to measure suppressed regions.

## Conclusion

For this deployment, pending worker bytes is the best currently consumable instantaneous proxy for CPU-to-GPU load wait, but it is not an adequate standalone selective-loading gate. The policy should optimize estimated request-visible scheduler load wait relative to recomputation value. The smallest implementation step is to export logical load-wait sum and event count as vLLM Counters, calculate their time-aware EWMA in the EPP, and leave enforcement disabled until the 80 ms candidate is tested under a sustained, independently randomized workload.

## Related

- [[00 - Index]]
- [[07 - Selective loading calibration test plan]]
- [[08 - 2026-09-07 selective loading calibration results]]
- [[04 - vLLM selective load - problem statement]]

## Provenance

Direct execution against the `benchflow` deployment on 2026-09-07, vLLM structured offload telemetry at sample rate 1.0, Prometheus exposition, EPP access logs, local request artifacts listed above, direct source inspection of vLLM and llm-d-router, and targeted unit tests. Figures include every accepted observation at the stated grain; no observations were downsampled.