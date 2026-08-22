---
title: "Clean-prefetch v1 — AgentX/Weka concurrency-64 comparison"
date: "2026-08-22"
type: "experiment-report"
experiment: "ABC"
status: "conditionally-valid"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-clean-prefetch-v1"
vllm_version: "0.27.0 plus clean-prefetch v1"
tensor_parallelism: 8
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: "vLLM default"
concurrency: 64
cpu_bytes: 274877906944
offload_spec: "TieringOffloadingSpec"
secondary_tier: "local NVMe filesystem"
secondary_tier_threads: "64 read / 64 write"
shared_memory: "300 GiB"
workload: "AIPerf inferencex-agentx-mvp / Weka trace"
random_seed: 20260707
duration_seconds: 1800
cache_cleaning_state: "BenchFlow hostPath cleanup enabled; pre-run device contents not independently fingerprinted"
control_run: "84aca7c754044fdbabfb7779508bff25"
treatment_run: "15322a87e2864f99a75972f59913fe9f"
---

# Clean-prefetch v1 — AgentX/Weka concurrency-64 comparison

## Executive summary

This experiment asked whether the clean admission-time exact-working-set implementation could use the natural queue created by a realistic concurrency-64 AgentX/Weka workload to stage NVMe-resident KV chunks in CPU before demand, without restricting vLLM to `--max-num-seqs=1`.

**The mechanism worked and the higher concurrency materially improved prefetch timeliness, but this single pair does not establish an end-to-end performance benefit.** Treatment request throughput was only 0.24% higher and total-token throughput 0.09% higher. Mean, p50, p95, and p99 TTFT improved, but p90 TTFT, p95 request latency, and ITL regressed. Paired request evidence was centered near zero.

The mechanism telemetry identifies a more important policy defect. The global 64-chunk tracked set was saturated for most of profiling. Pending request plans remained FIFO-queued after their request had already entered demand lookup, so freed capacity could be spent on stale work. Promotion readiness consequently reached 71.96 seconds mean and approximately 319 seconds p90. Of 958 promoted chunks, 638 were eventually useful, but only 353 were ready on first demand; 256 were wasted. Every one of 966 submissions evicted an ordinary CPU block, and 586 of those victims were later demanded again.

The direct scheduler-path policy cost was small. The evidence points instead to stale plan lifetime, transfer queueing, and eviction opportunity cost.

## Validity verdict

# Conditionally valid

The pair is valid for mechanism behavior and for showing that concurrency 64 creates a natural queue. It is inconclusive for performance because only one control/treatment pair exists, the cells ran on different H100 nodes, AgentX branch completion diverged slightly, and DCGM GPU telemetry was unavailable. No request errors, model-server exceptions, prefetch probe failures, or proactive load failures were found.

Do not claim that prefetch improves performance from this pair. Do not tune `N` against this implementation until stale post-demand work is removed.

## Runs and authoritative configuration

- [Control — 84aca7c754044fdbabfb7779508bff25](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/84aca7c754044fdbabfb7779508bff25?workspace=benchflow)
  - deployment profile: `multi-tier-offloading-nvme`
  - no `admission_prefetch_chunks` server setting
  - runtime node: `diadochos-hqxzk-gpu-h100-fx7c8`
- [Treatment — 15322a87e2864f99a75972f59913fe9f](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/15322a87e2864f99a75972f59913fe9f?workspace=benchflow)
  - deployment profile: `clean-prefetch-cpu-kv-offload-nvme`
  - `admission_prefetch_chunks=64`
  - `admission_prefetch_max_candidate_chunks=1024`
  - `admission_prefetch_apply_chunks_per_step=64`
  - `admission_prefetch_load_batch_chunks=8`
  - `admission_prefetch_max_pending_intents=64`
  - `admission_prefetch_max_tracked_chunks=64`
  - `admission_prefetch_max_eviction_history_chunks=4096`
  - source tier index 0
  - runtime node: `diadochos-hqxzk-gpu-h100-6kl5z`

Both cells used the same image, model, TP=8, one replica, 256 GiB CPU KV tier, local filesystem secondary tier with 64 read and 64 write threads, 128 Ki-token context, `--gpu-memory-utilization=0.8`, default `max_num_seqs`, and this request-body gate:

`{"ignore_eos":true,"kv_transfer_params":{"abc_admission_prefetch":true}}`

The request gate is inert in the control because its server configuration leaves admission prefetch disabled.

## Main takeaways

- **Measured:** treatment completed 1,248 requests versus 1,245 control requests with no profiling errors or cancellations.
- **Measured:** average waiting depth was 4.84 treatment and 5.13 control; p50 was four and p90 was 11 versus 12. Concurrency 64 therefore provided real admission lead time.
- **Measured:** late/useful fell to 44.67%, compared with 98.44% in the concurrency-32 clean-prefetch run.
- **Measured:** the tracked gauge stayed at 64 from p25 through p99, and the last in-window 15-second sample was also 64.
- **Measured:** only 36.85% of promoted chunks were ready at first demand, while eviction regret affected 60.66% of CPU victims.
- **Inference:** the current placement exchange is unfavorable: 353 on-time useful chunks versus 586 subsequently demanded victims. These counts are not equal-cost stall measurements, but they are a strong rejection signal.
- **Inference:** the large ready-delay tail is dominated by stale request plans, not by synchronous planner overhead.

## Headline metrics

| Metric | Control | Treatment | Relative treatment change |
|---|---:|---:|---:|
| Successful profiling requests | 1,245 | 1,248 | +0.24% |
| Request throughput | 0.676630 req/s | 0.678260 req/s | +0.24% |
| Total token throughput | 43,171.78 tok/s | 43,209.78 tok/s | +0.09% |
| Output token throughput | 453.51 tok/s | 456.44 tok/s | +0.65% |
| Mean TTFT | 8,267.63 ms | 8,079.96 ms | -2.27% |
| p50 TTFT | 6,150.58 ms | 5,661.03 ms | -7.96% |
| p90 TTFT | 19,782.39 ms | 20,313.15 ms | +2.68% |
| p95 TTFT | 25,047.41 ms | 23,265.65 ms | -7.11% |
| p99 TTFT | 31,943.00 ms | 28,017.02 ms | -12.29% |
| Mean request latency | 43,716.17 ms | 43,551.08 ms | -0.38% |
| p95 request latency | 130,700.37 ms | 135,343.46 ms | +3.55% |
| Mean ITL | 53.795 ms | 55.022 ms | +2.28% |
| p95 ITL | 88.498 ms | 92.165 ms | +4.14% |
| p99 ITL | 129.964 ms | 162.411 ms | +24.97% |
| AIPerf effective concurrency | 29.580 | 29.539 | -0.14% |
| Profiling errors | 0 | 0 | 0 |

Negative latency changes are improvements; positive latency changes are regressions.

Figure 1 summarizes the mixed end-to-end signal. Provenance is the AIPerf profiling summaries stored in the two MLflow runs.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Treatment relative to control",
  "width": 760,
  "height": 320,
  "data": {
    "values": [
      {"metric": "Request throughput", "change_pct": 0.241},
      {"metric": "Total-token throughput", "change_pct": 0.088},
      {"metric": "Mean TTFT", "change_pct": -2.270},
      {"metric": "p50 TTFT", "change_pct": -7.960},
      {"metric": "p90 TTFT", "change_pct": 2.683},
      {"metric": "p95 TTFT", "change_pct": -7.114},
      {"metric": "p99 TTFT", "change_pct": -12.291},
      {"metric": "p95 request latency", "change_pct": 3.552},
      {"metric": "Mean ITL", "change_pct": 2.282},
      {"metric": "p99 ITL", "change_pct": 24.966}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "metric", "type": "nominal", "title": null, "sort": null, "axis": {"labelAngle": -35}},
    "y": {"field": "change_pct", "type": "quantitative", "title": "Treatment change (%)"},
    "color": {"field": "metric", "type": "nominal", "legend": null, "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "metric", "type": "nominal"},
      {"field": "change_pct", "type": "quantitative", "title": "Change (%)", "format": ".3f"}
    ]
  }
}
~~~

The chart is deliberately neutral about the sign: negative is beneficial for latency but positive is beneficial for throughput. The mixture and small rate deltas do not constitute a win.

## Request-level evidence

Logical trace requests were paired by source trace, conversation, turn, source indices, kind, depth, and repeated chronological occurrence. This produced 1,238 exploratory pairs across 72 source traces.

| Paired statistic | Treatment minus control |
|---|---:|
| Mean TTFT delta | approximately -197 ms |
| Median TTFT delta | near 0 ms |
| Requests with lower treatment TTFT | approximately 51% |
| Trace-cluster bootstrap 95% interval for mean TTFT | approximately -441 to +46 ms |

The interval crosses zero. Pairing is approximate because AgentX branches and generated histories can diverge between executions; only 238 common records had exactly equal input and output token counts. For those strict same-work records, mean TTFT was 45 ms higher in treatment and median TTFT was 7 ms higher. The request-level evidence is therefore consistent with no stable causal effect.

## Scheduler pressure and reactive lookup

AIPerf scraped vLLM metrics approximately every 333 ms during profiling.

| Metric | Control | Treatment |
|---|---:|---:|
| Mean running requests | 26.63 | 26.74 |
| Mean waiting requests | 5.13 | 4.84 |
| Waiting p50 | 4 | 4 |
| Waiting p90 | 12 | 11 |
| Waiting p95 | 13 | 13 |
| Waiting p99 | 16 | 16 |
| Waiting maximum | 29 | 41 |
| Mean vLLM request queue time | 7.758 s | 7.560 s |
| Median vLLM request queue time | 4.830 s | 4.374 s |
| p95 vLLM request queue time | 25.594 s | 23.630 s |

Figure 2 shows that both cells operated under sustained continuous-batching pressure. Provenance is the high-frequency AIPerf vLLM gauge export.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 2 — Scheduler pressure summary",
  "width": 760,
  "height": 300,
  "data": {
    "values": [
      {"category": "Mean — Control", "configuration": "Control", "requests": 5.131},
      {"category": "Mean — Treatment", "configuration": "Treatment", "requests": 4.841},
      {"category": "p50 — Control", "configuration": "Control", "requests": 4},
      {"category": "p50 — Treatment", "configuration": "Treatment", "requests": 4},
      {"category": "p90 — Control", "configuration": "Control", "requests": 12},
      {"category": "p90 — Treatment", "configuration": "Treatment", "requests": 11},
      {"category": "Maximum — Control", "configuration": "Control", "requests": 29},
      {"category": "Maximum — Treatment", "configuration": "Treatment", "requests": 41}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "category", "type": "nominal", "title": null, "sort": null, "axis": {"labelAngle": -30}},
    "y": {"field": "requests", "type": "quantitative", "title": "Waiting requests (count)", "scale": {"zero": true}},
    "color": {"field": "configuration", "type": "nominal", "title": "Configuration", "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "configuration", "type": "nominal"},
      {"field": "category", "type": "nominal"},
      {"field": "requests", "type": "quantitative", "title": "Requests"}
    ]
  }
}
~~~

This is the lead-time regime missing from the concurrency-32 run, where mean waiting was about 0.1 and p90 was zero.

Reactive offload delays were slightly better at the center and mixed in the tail:

| Metric | Control | Treatment |
|---|---:|---:|
| Offload lookup async mean | 3.044 s | 2.979 s |
| Offload lookup async p50 estimate | 0.752 s | 0.634 s |
| Offload lookup async p90 estimate | 9.106 s | 9.435 s |
| Tiering blocked interval mean | 4.274 s | 4.215 s |
| Tiering blocked interval p50 estimate | 2.339 s | 2.171 s |
| Tiering blocked interval p90 estimate | 12.277 s | 12.547 s |

## Prefetch mechanism evidence

### Lifecycle counters

| Metric | Profiling value | Interpretation |
|---|---:|---|
| Accepted intents | 1,257 | Requests accepted into clean prefetch planning |
| Candidate keys | 1,206,734 | Bounded exact prompt candidates published |
| Source-resident | 1,128,306 | Probe results present in NVMe |
| Cancelled | 265,907 | Remaining planned keys discarded when requests finished |
| Primary redundant | 215,388 | Candidate already present or pending in CPU at application |
| Source miss | 12 | Probed source absence |
| Submitted | 966 | Chunks admitted into CPU promotion |
| Promoted | 958 | Successful NVMe-to-CPU completions |
| Useful | 638 | Later demand observed a CPU hit |
| Late | 285 | First demand observed the promotion still in flight |
| Wasted | 256 | Tracked proactive chunks never consumed before terminal accounting |
| Evicted for prefetch | 966 | CPU victims displaced for proactive admission |
| Eviction regret | 586 | A displaced ordinary victim was later demanded while absent |
| Load failed | 0 | No proactive transfer failed |
| Restaged after eviction | 0 | No tracked victim was proactively staged again |

The principal ratios are:

- source-resident/candidate: **93.50%**;
- submitted/candidate: **0.080%**;
- submitted: **0.77 chunk per accepted intent**;
- promoted/submitted: **99.17%**;
- useful/promoted: **66.60%**;
- late/useful: **44.67%**;
- on-time useful/promoted: **36.85%**;
- wasted/promoted: **26.72%**;
- eviction regret/evicted victims: **60.66%**.

Figure 3 shows the downstream outcomes where the policy decision matters. Provenance is the profiling-phase AIPerf vLLM counter delta.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 3 — Clean-prefetch downstream lifecycle outcomes",
  "width": 760,
  "height": 310,
  "data": {
    "values": [
      {"outcome": "Submitted", "chunks": 966},
      {"outcome": "Promoted", "chunks": 958},
      {"outcome": "Useful", "chunks": 638},
      {"outcome": "On-time useful", "chunks": 353},
      {"outcome": "Late", "chunks": 285},
      {"outcome": "Wasted", "chunks": 256},
      {"outcome": "Eviction regret", "chunks": 586}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "outcome", "type": "nominal", "title": null, "sort": null, "axis": {"labelAngle": -25}},
    "y": {"field": "chunks", "type": "quantitative", "title": "Chunks (count)", "scale": {"zero": true}},
    "color": {"field": "outcome", "type": "nominal", "legend": null, "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "outcome", "type": "nominal"},
      {"field": "chunks", "type": "quantitative"}
    ]
  }
}
~~~

“Useful” alone overstates the mechanism. Only 353 useful chunks were ready at first demand. Against that, 586 ordinary CPU victims were later missed and attributed as eviction regret. The counts do not include the duration of each stall and therefore cannot be subtracted as a performance estimate, but the direction is unfavorable.

### Planner and readiness timing

| Internal measurement | Count | Mean | p90 estimate | Total scheduler time |
|---|---:|---:|---:|---:|
| Admission intent publication | 1,257 | 0.356 ms | 0.534 ms | 0.447 s |
| Off-thread source probe | 1,257 | 13.923 ms | 23.794 ms | off scheduler thread |
| Scheduler completed-plan application | 33,320 observations | 0.0134 ms | 0.050 ms | 0.446 s |
| Admission-to-CPU-ready | 958 chunks | 71.956 s | approximately 318.9 s | N/A |

The policy does not spend enough synchronous time to explain the latency shape directly. The admission-to-ready metric includes time waiting behind the saturated tracked footprint and transfer work.

The tracked-prefetch gauge averaged 54.23 chunks and was exactly 64 at p25, p50, p75, p90, p95, and p99. The last 15-second sample inside profiling was also 64.

Figure 4 converts the cumulative ready histogram into exclusive latency ranges. Provenance is the profiling-phase `vllm:kv_offload_prefetch_ready_delay_seconds` histogram.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 4 — Admission-to-CPU-ready delay distribution",
  "width": 760,
  "height": 300,
  "data": {
    "values": [
      {"delay_range": "≤0.1 s", "chunks": 254},
      {"delay_range": "0.1–0.5 s", "chunks": 0},
      {"delay_range": "0.5–1 s", "chunks": 128},
      {"delay_range": "1–2 s", "chunks": 0},
      {"delay_range": "2–5 s", "chunks": 128},
      {"delay_range": "5–10 s", "chunks": 192},
      {"delay_range": "10–30 s", "chunks": 0},
      {"delay_range": ">30 s", "chunks": 256}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "delay_range", "type": "ordinal", "title": "Admission-to-ready delay range", "sort": null, "axis": {"labelAngle": -25}},
    "y": {"field": "chunks", "type": "quantitative", "title": "Promoted chunks (count)", "scale": {"zero": true}},
    "color": {"field": "delay_range", "type": "nominal", "legend": null, "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "delay_range", "type": "ordinal"},
      {"field": "chunks", "type": "quantitative"}
    ]
  }
}
~~~

Exactly 256 promotions took more than 30 seconds to become ready, and exactly 256 promotions were wasted. This equality is not per-key proof, but it strongly supports stale plan execution as the dominant source of waste.

## Storage and resource context

Prometheus NVMe data was available at native 15-second cadence during profiling:

| Metric | Control | Treatment | Relative change |
|---|---:|---:|---:|
| Mean runtime-node NVMe read | 0.938 GB/s | 0.922 GB/s | -1.79% |
| Mean runtime-node NVMe write | 0.475 GB/s | 0.476 GB/s | +0.27% |
| Mean aggregate container CPU usage | 466.16% | 465.86% | -0.06% |
| Reactive CPU-to-GPU offload load bytes | 6.988 TB | 6.924 TB | -0.92% |
| GPU telemetry | unavailable | unavailable | N/A |

The treatment did not produce obvious aggregate NVMe bandwidth amplification. This does not prove absence of demand interference: proactive and reactive work share the secondary-tier path, and the current artifacts do not separate their queue/service bytes.

External-KV prompt tokens were 51.235 million treatment versus 51.540 million control. The approximately 0.59% difference is much larger than the token coverage of 638 useful 16-token chunks, so it must not be attributed entirely to prefetch.

## Root-cause diagnosis

The clean implementation maintains a FIFO deque of completed plans. In `_apply_prefetch_plans()`, a full global tracked footprint causes an immediate `break`, leaving the oldest plan at the head. A plan is cancelled only if its request state is absent or finished.

That lifetime is too long for prompt prefetch. The first demand lookup is the deadline: once normal prefix lookup has begun for a request, unsubmitted proactive work for that request can no longer hide that lookup. Current code can nevertheless submit more keys later while the request remains active.

The measured causal chain is:

```text
64 tracked chunks fill
        ↓
oldest plan blocks later plans
        ↓
request begins reactive demand lookup
        ↓
old plan remains eligible until request finish
        ↓
a slot opens and stale prompt work is submitted
        ↓
very long ready delay, unused copy, and unnecessary CPU eviction
```

This is a policy-correctness defect, not a KV correctness defect. The normal reactive lookup remains authoritative, requests complete correctly, and no model output error was observed.

FIFO is also the wrong ordering under continuous batching. Among still-valid requests, the best next plan is the one closest to first demand, provided the estimated remaining horizon can still cover transfer time. The clean implementation already has exact request keys, request identity, admission time, scheduler request state, and observed transfer-ready cost. It does not need another predictor to implement this baseline.

## Decision and implementation action

Implement both changes before repeating:

1. **Demand cutoff:** on a request’s first tiering lookup, mark its proactive deadline passed and cancel all remaining unsubmitted plan work for that request. Never start another proactive load for it. Already in-flight work may complete safely and normal reactive fallback is unchanged.
2. **Queue-position ordering:** replace asynchronous probe-completion FIFO with a bounded request-priority policy over existing exact intents. Under FCFS, use arrival order; under priority scheduling, use the scheduler's native `(priority, arrival)` order. Expire work as soon as demand begins. Do not add an uncalibrated time-to-demand estimator in this patch.
3. **Metrics:** add explicit `cancelled_at_first_demand`, `expired_before_submit`, and `submitted_after_demand` accounting. The final counter is an invariant and must remain zero.
4. **Keep scope narrow:** do not restore V2/V3 temperature prediction, reserve leasing, single-owner state, or synchronous key ranking. Reactive correctness and full-cache eviction remain unchanged.

This is consistent with [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|the independent research reset]], which recommends deadline-aware working-set staging and demand-preemptible budgets rather than fixed-N FIFO work.


## Follow-up implementation — clean-prefetch v1.1

The surgical correction described above was applied after this experiment in the uncommitted `experimental/clean-prefetch-poc` worktree at `/Users/aperdomo/workspace/redhat/vllm-clean-prefetch`. It deliberately changes prefetch timing and plan selection only; reactive lookup, CPU eviction, tier transfer semantics, and model correctness remain unchanged.

### Implemented behavior

1. `RequestState.prefetch_deadline_passed` becomes true on the request's first `TieringOffloadingManager.lookup()` call.
2. Completed but unsubmitted plan suffixes for that request are removed immediately and counted as both `cancelled` and `cancelled_at_first_demand`.
3. A source probe that finishes after the cutoff is discarded before placement and counted as `expired_before_submit`.
4. New intents for a request are rejected after its deadline.
5. A defensive guard refuses any promotion path that somehow reaches placement after demand and records `submitted_after_demand`. This is an invariant-violation metric; it must remain zero in experiments.
6. Completed plans are selected by the scheduler's native order rather than worker probe-completion FIFO:
   - FCFS: `(0, request.arrival_time)`;
   - priority scheduling: `(request.priority, request.arrival_time)`;
   - intent ID breaks exact ties.
7. Explicit detached prefetch hints remain independent of the synthetic hint request's lifetime and are not cancelled by this cutoff.

The first lookup is a real deadline, not an estimate. Once lookup begins, an unsubmitted promotion cannot hide that lookup. The queue-position key is also factual scheduler state. This patch does **not** estimate whether the remaining queue duration can cover a transfer. Such a model should be added only after the corrected run provides calibrated admission-to-demand and admission-to-ready distributions.

### Touched code

| File | Change |
|---|---|
| `vllm/v1/kv_offload/tiering/manager.py` | First-demand cutoff, late-probe expiry, defensive post-demand submission guard, and bounded scheduler-order plan selection |
| `vllm/v1/kv_offload/tiering/prefetch.py` | Carries the immutable scheduler-order key on each intent |
| `vllm/v1/kv_offload/base.py` | Extends the optional prefetch-intent API with `scheduler_order` |
| `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py` | Maps vLLM's configured FCFS/priority policy and request fields into the intent key |
| `tests/v1/kv_offload/tiering/test_tiering_offloading.py` | Tests cutoff before apply, probe completion after cutoff, post-demand guard, and scheduler ordering |
| `tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py` | Tests FCFS and priority key propagation |

### Verification

- 69 relevant tests passed: the entire tiering manager file plus the focused admission-working-set scheduler class.
- Ruff lint and formatting passed on all six touched production/test files.
- `git diff --check` passed.
- The broader scheduler file contains model-fixture tests that attempted Hugging Face network access and failed with DNS/connect errors in the restricted environment; those failures were unrelated to this change. The isolated scheduler class passed.
- No commit was created.


## Next experiment and gates

After the surgical fix, repeat the same realistic concurrency-64 pair with the same image/configuration matrix and no `max_num_seqs` restriction. Use at least three balanced pairs before claiming performance.

Do not sweep `N` until all of these hold:

- `submitted_after_demand=0`;
- no plan remains queued after its request’s first lookup;
- ready-delay p90 is below the measured queue horizon rather than hundreds of seconds;
- on-time useful exceeds eviction regret in both chunk count and estimated stall cost;
- wasted/promoted is below 10%;
- mean and p95 ITL remain within 5% of control;
- treatment improves p95 TTFT by at least 5% or throughput by at least 3% across repeated pairs.

If deadline-valid exact staging still cannot beat eviction regret, compare reuse-aware CPU retention against movement before adding a more complex predictor.

## Run registry

| Role | Run ID | Disposition |
|---|---|---|
| Control | [84aca7c754044fdbabfb7779508bff25](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/84aca7c754044fdbabfb7779508bff25?workspace=benchflow) | Valid reactive comparator; one repetition |
| Treatment | [15322a87e2864f99a75972f59913fe9f](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/15322a87e2864f99a75972f59913fe9f?workspace=benchflow) | Mechanism valid; performance inconclusive; policy requires demand cutoff and deadline ordering |

## Related evidence

- [[2026-08-21 - Clean-prefetch v1 AgentX first comparison|Clean-prefetch v1 concurrency-32 comparison]]
- [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|Independent research audit and redirection]]
- [[2026-08-18 - AgentX Weka admission prefetch concurrency 64|Earlier admission-prefetch concurrency-64 experiment]]