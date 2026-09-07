---
title: Selective loading calibration test plan
date: 2026-09-07
type: test-plan
topic: Selective KV loading
project: Selective KV Loading and Offloading
status: planned
configuration:
  policy: static reusable-token threshold
  comparison:
    - load-allowed
    - forced-recompute
  threshold_unit: external reusable tokens
---

# Selective loading calibration test plan

## Purpose

Calibrate the first static llm-d-router threshold for deciding whether a request should restore reusable KV from the configured external offload hierarchy or recompute the same prefix.

This plan answers one narrow question:

> For a request with (N) externally reusable tokens, is restoring those tokens faster than recomputing them under the deployment's intended operating pressure?

This is distinct from general KV offload pressure calibration. Pressure calibration finds a useful workload and cache configuration. This plan finds the load-versus-recompute break-even point within a fixed model and deployment configuration.

## Scope

The initial vLLM and router contract is binary:

- Omit `kv_load_tiers`: preserve vLLM's configured loading behavior.
- Send `kv_load_tiers: []`: do not query or load from any external offload tier, including primary CPU. Local HBM prefix-cache reuse remains available.

The initial router cannot select CPU versus storage independently. The calibrated threshold therefore applies to the deployment's complete configured load hierarchy.

Loading and offloading remain independent decisions. Keep offload behavior identical between calibration treatments so the experiment isolates the load decision.

## Decision model

For (N) externally reusable tokens:

$$
T_{recompute}(N) \approx \frac{N}{R_{prefill}}
$$

$$
T_{load}(N) \approx T_{fixed} + \frac{N \times B_{token}}{BW_{effective}}
$$

where:

- (R_{prefill}) is effective prefill throughput under the tested pressure.
- (B_{token}) is empirically observed KV bytes per transferred token.
- (BW_{effective}) is effective load bandwidth.
- (T_{fixed}) includes lookup, dispatch, synchronization, and scheduler-visible completion overhead.

The approximate crossing is:

$$
N^* \approx \frac{T_{fixed}}
{1 / R_{prefill} - B_{token} / BW_{effective}}
$$

This analytical model is explanatory only. Continuous batching, transfer contention, and scheduler completion observation are nonlinear. Select the production threshold from paired request-level measurements.

For each prefix bucket, calculate:

$$
D(N) = TTFT_{load}(N) - TTFT_{recompute}(N)
$$

- (D(N) < 0): loading is faster.
- (D(N) > 0): recomputation is faster.

## Threshold input

The policy input should be the number of reusable tokens that require an external tier:

```text
offloaded_match = max(cpu_matched_tokens, storage_matched_tokens)

external_reusable_tokens =
    max(0, offloaded_match - gpu_local_matched_tokens)
```

This formula is provisional until the llm-d cache-evidence contract is confirmed:

- If per-tier values are inclusive prefix lengths, subtract the local GPU prefix.
- If they are exclusive token counts, do not subtract again.
- Never add CPU and storage matches because both tiers may contain the same prefix.
- Align the result and configured threshold with vLLM's KV block or transfer-chunk boundaries.
- Use evidence for the selected backend endpoint, not aggregate evidence from unrelated replicas.

## Deployment fingerprint

A static threshold is valid only for the deployment envelope against which it was measured. Record and hold fixed:

- model name, revision, weights dtype, and KV-cache dtype;
- vLLM image, commit, and selective-load patch version;
- llm-d-router image and commit;
- GPU type and count;
- tensor parallelism, data parallelism, and replica count;
- block size and transfer-chunk size;
- GPU memory utilization and maximum model length;
- maximum sequences and scheduler configuration;
- CPU offload capacity and allocator configuration;
- secondary-tier type, capacity, and worker/thread configuration;
- CPU/NUMA/PCIe topology;
- shared-memory allocation;
- prompt suffix and requested output length;
- offload policy and `max_offload_tokens`;
- concurrency, workload seed, duration, and cache-cleaning method.

During the initial calibration, vary only external reusable prefix length and concurrency or pressure.

## Treatments

### A. Load allowed

Omit `kv_load_tiers`. Keep normal offloading enabled.

### B. Forced recompute

Send:

```json
{
  "kv_transfer_params": {
    "kv_load_tiers": []
  }
}
```

Keep the same offload behavior as treatment A.

### Optional control: connector disabled

Run a no-offload-connector configuration only as a mechanism sanity check. Do not substitute it for treatment B because removing the connector changes more than the per-request load decision.

## Cache-state preparation

Each measured request must begin from an equivalent, verified cache state:

1. Submit a seed request containing the target reusable prefix.
2. Verify that the expected KV was offloaded.
3. Force or wait for the prefix to leave HBM.
4. Verify that the prefix remains discoverable in the configured external hierarchy.
5. Submit the measured continuation request using treatment A or B.
6. Record the router decision, vLLM enforcement outcome, prompt-token source, and latency.
7. Restore equivalent cache state before the next sample.

A sample is invalid if the intended prefix is still fully resident in HBM, missing from the external tier, or cannot be associated with the measured request.

Alternate or randomize treatment order to reduce cache-temperature, thermal, and time-drift bias.

## Experiment phases

### Phase 0: instrumentation validation

Before collecting threshold data, verify:

- load-allowed requests produce `external_kv_transfer` prompt tokens and positive load bytes;
- forced-recompute requests produce no load operations for the request and attribute the equivalent prefix to `local_compute`;
- local HBM hits remain usable in both treatments;
- router decisions can be correlated with the selected endpoint and vLLM outcome;
- metric counters have the expected per-process or per-rank aggregation semantics;
- telemetry sampling is high enough for calibration requests;
- telemetry does not introduce material request-path overhead.

Do not proceed if treatment behavior cannot be distinguished reliably.

### Phase 1: low-concurrency mechanism sweep

Start at concurrency 1 to estimate the physical break-even region.

Use an initial block-aligned prefix sweep near:

```text
256
512
1,024
2,048
4,096
8,192
16,384
32,768
64,000
```

Extend the range if no crossing is observed. After locating the crossing region, perform a finer sweep around it.

Collect at least 20 to 30 valid samples for every prefix and treatment combination. Increase repetitions if variance or confidence intervals remain large.

### Phase 2: representative pressure sweep

Repeat the narrowed prefix sweep at:

1. Low pressure.
2. Expected production concurrency.
3. High but operationally valid pressure.

Choose concurrency levels around the deployment's measured operating point rather than adopting universal values. A run is not valid for threshold selection if it is dominated by cancellations, failures, persistent capacity saturation, pod restarts, or unbounded queue growth.

If practical, evaluate both an idle or warm CPU tier and a deliberately pressured but valid CPU tier. This reveals whether a single static threshold is stable enough for the intended deployment envelope.

### Phase 3: policy validation

Compare three policies on a representative trace:

- always load: omit `kv_load_tiers` for every eligible request;
- calibrated threshold: disable external loading only below the selected threshold;
- always recompute: send `kv_load_tiers: []` for every eligible request.

Always recompute is a diagnostic boundary, not the expected performance baseline. The primary comparison is calibrated threshold versus always load.

Run multiple seeds or repetitions. Preserve identical offload policy and workload distribution.

## Metrics

### Decision and evidence

Record per request where possible:

- selected backend endpoint;
- GPU-local matched tokens;
- CPU matched tokens;
- secondary-tier matched tokens;
- calculated external reusable tokens;
- evidence age and compatibility status;
- configured threshold;
- decision: preserve load or disable load;
- reason: below threshold, missing evidence, stale evidence, incompatible backend, or policy disabled.

These fields are required to prove that each request entered the expected prefix bucket and policy branch.

### Mechanism confirmation

Use:

- `vllm:prompt_tokens_by_source`, separated into `local_compute`, `local_cache_hit`, and `external_kv_transfer`;
- external prefix-cache queries and hits;
- `vllm:kv_offload_load_bytes`;
- `vllm:kv_offload_load_time`;
- load-operation count or load-size count;
- scheduler resolution duration and outcome;
- scheduler load-wait duration;
- synchronous and asynchronous lookup delay;
- pending transfer jobs and bytes.

Useful derived values are:

$$
BW_{DMA} =
\frac{\operatorname{increase}(load\_bytes)}
{\operatorname{increase}(load\_time)}
$$

$$
T_{DMA,op} =
\frac{\operatorname{rate}(load\_time)}
{\operatorname{rate}(load\_operation\_count)}
$$

$$
B_{token} =
\frac{load\_bytes}
{external\_kv\_transfer\_tokens}
$$

Confirm the counter aggregation semantics before deriving cluster-level values, especially with tensor-parallel workers.

### Primary outcome

The main outcome is request-level time to first token. Report:

- median and p90 or p95 TTFT by prefix bucket and treatment;
- paired or matched TTFT difference (D(N));
- sample count;
- confidence interval, preferably bootstrap;
- proportion of samples for which loading wins.

Also report:

- prefill latency;
- request queue time;
- end-to-end latency;
- inter-token latency;
- request throughput;
- output-token throughput.

### Guardrails and validity

Monitor:

- running requests;
- waiting requests by capacity and deferred reason;
- GPU KV-cache occupancy;
- preemptions;
- CPU-cache total, read, and write utilization;
- CPU allocation failures;
- pending transfer jobs and bytes;
- CPU utilization, memory pressure, and memory bandwidth when available;
- GPU utilization when available;
- request completions, cancellations, and errors;
- pod restarts and OOM events.

GPU KV-cache occupancy is not a substitute for GPU compute utilization.

## Transfer timing interpretation

Active DMA duration does not represent the complete request-visible load cost.

The previously observed transfer-turnaround metric included completion observation or scheduler polling delay and therefore must not be interpreted as pure transfer queueing time. It may still approximate request-visible delay.

For diagnosis, separate:

```text
submission → worker starts copy       transfer queue delay
worker starts → copy completes        active copy time
copy completes → scheduler observes   completion-observation delay
submission → all workers complete     request-visible load wait
```

A dedicated worker-start timestamp would improve attribution. It is not a prerequisite for threshold selection if request-visible load wait and paired TTFT are measured correctly.

## Threshold-selection rule

For every prefix bucket, produce a table containing:

| External reusable tokens | Median ΔTTFT | p95 ΔTTFT | Load-win rate | Throughput delta | Valid samples |
|---:|---:|---:|---:|---:|---:|
| To measure | To measure | To measure | To measure | To measure | To measure |

Select the smallest block-aligned prefix for which:

- median TTFT reliably favors loading;
- tail TTFT is within the deployment's accepted tolerance;
- loading continues to win for larger measured buckets;
- request throughput and inter-token latency do not regress materially;
- the sample count and confidence interval are sufficient;
- the server remains within a valid operating envelope.

Do not use the first noisy negative point. Apply a conservative safety margin by selecting the next validated bucket or by requiring a minimum absolute or relative loading advantage. Define the margin from the deployment's SLO before evaluating results.

If thresholds differ substantially across valid pressure levels, either:

- choose a conservative threshold for the intended production pressure;
- maintain separately calibrated thresholds for distinct deployment profiles; or
- treat the result as evidence for a later dynamic pressure-aware policy.

When evidence is absent, stale, incompatible, or outside the calibrated envelope, fail open by preserving vLLM's default loading behavior.

## Initial router policy

```python
if not backend_supports_binary_load_opt_out:
    preserve_default_loading()
elif cache_evidence_is_missing_or_stale:
    preserve_default_loading()
else:
    external_tokens = calculate_external_reusable_tokens(cache_evidence)

    if external_tokens < configured_threshold:
        disable_external_loading()
    else:
        preserve_default_loading()
```

When disabling external loading:

```python
kv_transfer_params["kv_load_tiers"] = []
```

Otherwise omit `kv_load_tiers`. Recompute the decision after endpoint reselection or retry.

## Invalid conclusions and anti-patterns

Do not select a threshold from:

- raw DMA duration alone;
- transfer turnaround interpreted as pure queue time;
- aggregate TTFT from mixed, unmatched prompt lengths;
- external cache-hit rate alone;
- a fully saturated or failure-dominated run;
- comparisons with different offload policies;
- comparisons in which the treatments begin with different cache residency;
- the previous always-recompute run with zero loads and pathological queueing.

That previous run establishes that disabling every external load can be harmful under its workload. It does not identify the break-even prefix.

## Acceptance criteria

The calibration is complete when:

1. Treatment enforcement is verified for every measured request class.
2. The prefix accounting contract is confirmed.
3. A stable crossing region is observed at the intended production pressure.
4. The chosen threshold includes an explicit statistical and operational safety margin.
5. The threshold policy matches or improves TTFT and request throughput relative to always load.
6. Tail latency, ITL, cancellations, errors, preemptions, and queue growth remain within declared tolerances.
7. The complete deployment fingerprint, workload, cache preparation, run IDs, and result tables are recorded.
8. The router fails open outside the supported capability and evidence envelope.

## Required deliverables

- deployment fingerprint and controlled-variable table;
- workload generator and deterministic prefix set;
- treatment assignment and cache-state procedure;
- request-level decision and latency dataset;
- Prometheus query set and metric aggregation notes;
- MLflow run registry with direct links;
- per-prefix comparison table with uncertainty;
- mechanism and validity plots at the finest available cadence;
- selected threshold and safety-margin rationale;
- list of unresolved telemetry or attribution gaps;
- recommendation for static rollout or dynamic-policy follow-up.

## Recommended execution order

1. Validate instrumentation with a small load-allowed and forced-recompute pair.
2. Run the low-concurrency coarse prefix sweep.
3. Narrow and repeat around the observed crossing.
4. Repeat the narrowed sweep at production pressure.
5. Validate the chosen threshold against always load on a representative trace.
6. Enable the static policy only for the exact calibrated deployment fingerprint.
7. Recalibrate after material changes to model, parallelism, GPU type, KV dtype, transfer geometry, tier topology, or workload pressure.

## Related

- [[00 - Index]]
- [[01 - Initial implementation plan]]
- [[03 - llm-d-router selective load and offload policy and wiring]]
- [[04 - vLLM selective load - problem statement]]
- [[01 - Calibration Protocol]]
- [[03 - KV Transfer Metrics and PromQL]]