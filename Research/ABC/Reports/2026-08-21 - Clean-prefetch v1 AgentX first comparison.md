---
title: "Clean-prefetch v1 — first AgentX/Weka comparison"
date: "2026-08-21"
type: "experiment-report"
experiment: "ABC"
status: "conditionally-valid"
workload: "AgentX Weka, concurrency 32, continuous batching"
control_run: "d7f74e2291d348f884b3a0e9016627a4"
treatment_run: "cf327f18bd9a4935b3c217ed46016211"
---

# Clean-prefetch v1 — first AgentX/Weka comparison

## Executive conclusion

**The clean implementation is mechanically correct enough to continue studying, but this pair shows no end-to-end performance benefit.**

The treatment successfully used the new full-cache eviction path: all 256 submitted chunks evicted an ordinary CPU block, all 256 were promoted, all 256 were eventually demanded, and no load failure, waste, eviction regret, or capacity skip was reported. This is strong evidence that the clean implementation fixed the earlier “only use genuinely free CPU slots” limitation.

The central problem is timing. **252 of 256 promoted chunks (98.44%) were late at first demand.** Only four promotions were ready before their first observed use. The mean promotion-ready delay was about 93.8 ms, while the workload maintained almost no admission queue. The mechanism therefore predicts the right data but starts too late to hide its movement.

Headline performance was essentially tied:

- request throughput differed by less than 0.0001%;
- mean request latency improved by 0.14%;
- mean TTFT regressed by 1.64%;
- p95 TTFT regressed by 8.20%;
- mean ITL regressed by 1.35%;
- p95 ITL improved by 4.08%.

Those mixed changes are not evidence of either benefit or a meaningful regression. Request-level pairing also showed a mixed distribution: the treatment had lower TTFT for 61.65% of matched requests, but a small number of slower tail requests raised the mean and p95.

This pair is **conditionally valid for mechanism behavior and inconclusive for performance**. The cells ran concurrently despite the intended sequential placement, only one pair was executed, three requests were cancelled in each cell at shutdown, GPU telemetry was unavailable, and the artifact export omitted deployment manifests and native-cadence Prometheus data.

## Runs and configuration

- [Control — d7f74e2291d348f884b3a0e9016627a4](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/d7f74e2291d348f884b3a0e9016627a4?workspace=benchflow)
  - deployment profile: `multi-tier-offloading-nvme`
  - image tag: `v0.27.0-clean-prefetch-v1`
  - prefetch policy disabled
- [Treatment — cf327f18bd9a4935b3c217ed46016211](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/cf327f18bd9a4935b3c217ed46016211?workspace=benchflow)
  - deployment profile: `clean-prefetch-cpu-kv-offload-nvme`
  - image tag: `v0.27.0-clean-prefetch-v1`
  - `admission_prefetch_chunks=64`
  - `prefetch_global_max_tracked_chunks=64`
  - `prefetch_candidate_chunks=1024`
  - `prefetch_apply_chunks_per_step=64`
  - `prefetch_batch_size=8`
  - `prefetch_max_pending_plans=64`
  - `prefetch_eviction_history_size=4096`

Both used the AgentX Weka trace at concurrency 32, TP=8, a 128 Ki-token model context, and continuous batching without `--max-num-seqs=1`. Each profiling phase recorded 866 sent requests and 863 successful completed requests. Three in-flight requests per cell were cancelled after the 30-second shutdown grace period.

The treatment exported the clean-prefetch metrics, proving that the policy was enabled. The control exported none of those series.

## End-to-end performance

| Metric | Control | Treatment | Relative change |
|---|---:|---:|---:|
| Successful requests | 863 | 863 | 0 |
| Request throughput | 0.469021 req/s | 0.469021 req/s | -0.00003% |
| Total token throughput | 30,311.93 tok/s | 30,312.01 tok/s | +0.00027% |
| Mean TTFT | 1,105.81 ms | 1,123.99 ms | +1.64% |
| p95 TTFT | 4,151.43 ms | 4,491.92 ms | +8.20% |
| Mean request latency | 20,902.74 ms | 20,874.49 ms | -0.14% |
| p95 request latency | 52,746.24 ms | 53,581.01 ms | +1.58% |
| Mean ITL | 26.905 ms | 27.269 ms | +1.35% |
| p95 ITL | 46.345 ms | 44.453 ms | -4.08% |

Positive values below mean that the treatment was slower; negative values mean it was faster.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Treatment relative to control",
  "width": 700,
  "height": 300,
  "data": {
    "values": [
      {"metric": "Request throughput", "change": -0.00003},
      {"metric": "Mean TTFT", "change": 1.6442},
      {"metric": "p95 TTFT", "change": 8.2018},
      {"metric": "Mean request latency", "change": -0.1352},
      {"metric": "p95 request latency", "change": 1.5826},
      {"metric": "Mean ITL", "change": 1.3545},
      {"metric": "p95 ITL", "change": -4.0830}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "metric", "type": "nominal", "title": null, "sort": null, "axis": {"labelAngle": -30}},
    "y": {"field": "change", "type": "quantitative", "title": "Treatment change (%)"},
    "color": {
      "condition": {"test": "datum.change > 0", "value": "#d62728"},
      "value": "#2ca02c"
    },
    "tooltip": [
      {"field": "metric", "type": "nominal"},
      {"field": "change", "type": "quantitative", "title": "Change (%)", "format": ".3f"}
    ]
  }
}
~~~

The request-level records can be paired by trace ID, outer index, turn, and repeated occurrence for all 863 completions:

| Paired statistic | Treatment minus control |
|---|---:|
| Mean TTFT delta | +18.18 ms |
| Median TTFT delta | -12.10 ms |
| Requests with lower treatment TTFT | 61.65% |
| p90 paired TTFT delta | +160.92 ms |
| p95 paired TTFT delta | +734.64 ms |
| Mean request-latency delta | -28.26 ms |
| Median request-latency delta | -4.92 ms |
| Requests with lower treatment request latency | 51.68% |
| Mean ITL delta | +0.364 ms |

This shape is consistent with a neutral central tendency and a few treatment tail outliers. It does not establish a causal prefetch effect.

## Prefetch mechanism evidence

### Lifecycle counters

| Metric | Treatment total | Interpretation |
|---|---:|---|
| Intents | 866 | One intent per sent profiling request |
| Candidate chunks | 842,149 | Ordered prompt candidates examined |
| Source-resident | 796,750 | Candidates known to exist in the secondary tier |
| Redundant | 794,433 | Candidate already satisfied by CPU state |
| Source miss | 67 | Candidate absent from the probed secondary tier |
| Cancelled | 7,697 | Planned work discarded before submission |
| Submitted | 256 | Chunks actually submitted for promotion |
| Evicted for prefetch | 256 | Ordinary CPU victims displaced to admit the submitted chunks |
| Promoted | 256 | Successful secondary-to-CPU copies |
| Useful | 256 | Later demand consumed every promoted CPU copy |
| Late | 252 | First demand arrived while promotion was still in flight |
| Load failed | 0 | No proactive load failed |
| Wasted | 0 | No completed copy went unused before terminal accounting |
| Eviction regret | 0 | No displaced ordinary victim was later demanded during the tracked regret window |

The critical ratios are:

- redundant / candidate: **94.33%**;
- submitted / candidate: **0.0304%**;
- submitted / intent: **0.296 chunks per request**;
- promoted / submitted: **100%**;
- useful / promoted: **100%**;
- late / promoted: **98.44%**;
- timely / promoted: **1.56%**.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 2 — Clean-prefetch v1 mechanism ratios",
  "width": 700,
  "height": 300,
  "data": {
    "values": [
      {"ratio": "Redundant / candidate", "percent": 94.3340},
      {"ratio": "Submitted / candidate", "percent": 0.0304},
      {"ratio": "Promoted / submitted", "percent": 100.0},
      {"ratio": "Useful / promoted", "percent": 100.0},
      {"ratio": "Late / promoted", "percent": 98.4375},
      {"ratio": "Timely / promoted", "percent": 1.5625},
      {"ratio": "Eviction regret / victims", "percent": 0.0}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {"field": "ratio", "type": "nominal", "title": null, "sort": null, "axis": {"labelAngle": -30}},
    "y": {"field": "percent", "type": "quantitative", "title": "Percent", "scale": {"domain": [0, 100]}},
    "color": {"field": "ratio", "type": "nominal", "legend": null},
    "tooltip": [
      {"field": "ratio", "type": "nominal"},
      {"field": "percent", "type": "quantitative", "title": "Percent", "format": ".3f"}
    ]
  }
}
~~~

### What these counters prove

1. **Full-cache admission works.** Every submitted speculative promotion invoked the intended eviction path. The clean policy is no longer limited to unused physical CPU slots.
2. **Victim selection did not show immediate harm.** There were no observed eviction-regret events. This is encouraging but based on only 256 victims and a finite tracking window.
3. **The selected data was correct.** Every completed promotion became useful and no proactive load failed.
4. **Correctness is not timeliness.** “Useful” records a later CPU hit. It does not mean that the promotion was ready before first demand or that exposed stall was removed. With 252 late events, almost all useful copies missed that stronger goal.
5. **The policy rarely found actionable work.** More than 94% of candidates were already redundant, and only 256 of 842,149 candidates were submitted.

### Timing and scheduler-path cost

| Internal measurement | Count | Total | Mean | Tail estimate |
|---|---:|---:|---:|---:|
| Admission-delay observation | 866 | 0.250 s | 0.289 ms | p90 ~0.442 ms |
| Off-thread source probe | 866 | 9.981 s | 11.526 ms | p90 ~21.38 ms |
| Scheduler apply | 12,505 | 1.210 s | 0.0968 ms | p90 ~0.177 ms |
| Promotion ready delay | 256 | 24.016 s | 93.812 ms | p90 ~218.6 ms |

The scheduler-facing implementation overhead is small: about 0.25 seconds of total admission work and 1.21 seconds of total apply work across roughly 1,840 seconds of profiling. The expensive source probe is off-thread. These measurements do not support “the policy code itself is blocking the scheduler” as the primary explanation for this run.

The limiting quantity is lead time. The ready delay averages about 94 ms and has a tail above 200 ms, while scheduler pressure is low:

- mean running requests: 10.43 control, 10.41 treatment;
- mean waiting requests: 0.095 control, 0.107 treatment;
- p90 waiting requests: 0 in both;
- maximum waiting requests: 4 in both.

At concurrency 32, the model can keep many requests active and most new work obtains little or no waiting horizon. Admission therefore occurs too near demand for NVMe-to-CPU promotion to finish.

## Data-plane and cache context

The two cells were otherwise very close:

| Measurement | Control | Treatment |
|---|---:|---:|
| External-prefix chunks | 2,519,008 | 2,514,912 |
| Local-cache chunks | 49,581,616 | 49,585,920 |
| Locally computed chunks | 5,900,555 | 5,900,526 |
| Secondary-tier load bytes | 325.879 GB | 325.342 GB |
| Secondary-tier load time | 23.032 s | 23.250 s |
| Secondary-tier store bytes | 428.916 GB | 428.901 GB |
| Secondary-tier store time | 20.823 s | 22.342 s |
| Mean running requests | 10.432 | 10.408 |
| Mean waiting requests | 0.095 | 0.107 |
| Mean GPU KV usage | 0.3697 | 0.3688 |

The treatment's extra 256 chunks are tiny relative to the workload's hundreds of gigabytes of normal tier traffic. That explains why a correct but rare mechanism has no visible aggregate throughput effect.

Reactive asynchronous tier lookup was slightly worse in the treatment (mean about 164 ms versus 157 ms, with a broader tail), but a single concurrent cross-node pair cannot attribute that difference to prefetch.

## Validity and artifact-export issues

### Runs overlapped despite requested sequential placement

MLflow timestamps and AIPerf logs show that the cells were concurrent:

- control MLflow start: 20:13:10 UTC-equivalent epoch; benchmark profiling approximately 20:36:25–21:07:05;
- treatment MLflow start: 3.25 seconds later; profiling approximately 20:36:34–21:07:14.

The current experiment definition requests sequential placement, but the runtime did not honor it. Without the missing resolved-run-plan and deployment artifacts, the cause cannot be identified from these runs. A stale orchestration image or a different execution path is plausible, but unproven.

This matters because cells may have experienced shared infrastructure contention, and they certainly ran on different deployment instances. It does not invalidate treatment-side mechanism counters, but it weakens the causal performance comparison.

### Missing artifacts

Only the benchmark subtree was uploaded. Available artifacts include AIPerf logs, request-level JSONL/CSV, console output, and aggregate server metrics. Missing artifacts include:

- resolved run plan;
- rendered manifests and deployment logs;
- deployment image digest and authoritative KV-transfer configuration;
- BenchFlow native-cadence Prometheus query exports;
- GPU telemetry.

AIPerf's `server_metrics_export.json` preserved aggregate vLLM counter and histogram deltas, which is why mechanism accounting remains possible. It does **not** preserve the 15-second samples needed for honest mechanism time-series plots. No time series should be reconstructed from aggregates.

GPU exporter endpoints on ports 9400/9401 were unavailable in both cells, so GPU utilization, clocks, memory bandwidth, and PCIe traffic cannot be compared.

### Final-request race

Both cells sent 866 profiling requests and completed 863 successfully. The remaining three were cancelled after the shutdown grace period, with no request errors. Because the loss is symmetric, headline throughput is comparable, but a future gate should require the same completed request count and record graceful drain explicitly.

## Interpretation against the research reset

This result changes the diagnosis in a useful way:

- It weakens the hypothesis that recent failures were caused mainly by a large, complex scheduler-path policy. The clean implementation's measured scheduler overhead is small.
- It supports full-cache speculative admission as a viable mechanism primitive. Eviction occurred without observed regret.
- It strengthens the audit's claim that **admission-time prediction horizon is the main constraint under realistic continuous batching**.
- It demonstrates why eventual-use precision is insufficient. Precision is perfect among submitted chunks, yet 98.44% are late and performance is flat.
- It does not yet test the audit's primary oracle: a future working set announced early enough to finish staging before demand.

The result should therefore be retained as a **clean admission-time baseline**, not expanded into another complicated admission heuristic.

## Recommended next experiment

Do not sweep concurrency or add queue heuristics yet. First run the smallest discriminating timing experiment on the same clean branch:

1. Keep the same AgentX trace, continuous batching, image, prefetch budget, and full-cache eviction implementation.
2. Introduce a controlled **prefetch lead-time oracle** outside the ordinary admission critical path: announce the exact future request prefix 0, 50, 100, 250, 500, and 1,000 ms before submitting its inference request.
3. Preserve reactive fallback and lower priority for speculative I/O.
4. Measure:
   - timely/promoted rather than useful/promoted;
   - exposed reactive lookup/promotion delay;
   - request TTFT and throughput;
   - bytes prefetched, ordinary victims, and eviction regret;
   - GPU and PCIe telemetry;
   - demand versus speculative I/O queue delay.
5. Require repeated, placement-balanced pairs and non-overlapping cells.

The decisive questions are:

- Does timely/promoted rise sharply once lead time exceeds the observed 94–220 ms promotion delay?
- Does that rise reduce exposed reactive lookup delay?
- Does reduced lookup delay translate into TTFT or throughput improvement?

If even 250–500 ms of exact, zero-cost future knowledge produces no material benefit, the current NVMe-to-CPU-only architecture has little practical headroom for this workload. If it does, the production direction should seek naturally early AgentX signals—tool execution, workflow dependencies, or session continuation—not more elaborate same-request admission logic.

## Stop/go criteria

Proceed beyond the oracle only if, across at least three balanced pairs:

- at least 80% of promoted chunks are ready at first demand;
- demand-path secondary lookup/promotion stall falls by at least 20%;
- mean or p95 TTFT improves by at least 5% without reducing throughput;
- speculative I/O does not increase demand I/O p95 by more than 5%;
- eviction regret remains below 5%;
- every run has complete artifacts, GPU telemetry, and identical successful request counts.

Kill admission-only prefetch as a production direction if exact working-set staging with 250–500 ms lead time fails those gates. Retain it only as a mechanism and correctness baseline.