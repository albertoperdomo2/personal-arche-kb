---
title: "2026-08-23 - Working-set oracle AgentX first comparison"
date: "2026-08-23"
type: "experiment-report"
experiment: "ABC"
status: "complete"
verdict: "Mechanism active; negative request-readiness result; performance benefit not established"
workload: "AgentX Weka traces"
model: "FP8 Nemotron 253B"
concurrency: 64
---

# Working-set oracle AgentX first comparison

## Executive conclusion

The two runs make sense operationally and use the intended control/treatment configuration. The working-set treatment is active, moves many correct chunks, and has no load failures. It nevertheless fails the experiment's central oracle condition: only **20 of 2,638 request intents (0.76%)** were completely ready at the connector's first lookup. The remaining **2,618 (99.24%)** still entered deferred reactive loading.

Performance is correspondingly near-neutral and mixed. Mean TTFT improved 1.87%, but p95 and p99 TTFT regressed 1.11% and 6.51%; request throughput improved only 0.16%. This does not meet the predefined 5% TTFT or 3% throughput gate.

This pair is therefore a valid negative result for this admission-time, one-owner working-set implementation under concurrency-64 AgentX. It is **not** a valid test of a true perfect-residency oracle, because the treatment almost never achieved full residency before demand.

## Run identity and comparability

| Role | MLflow run | Deployment |
|---|---|---|
| Working-set treatment | [39a70a1b52e241bcb48abe5338d56110](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/39a70a1b52e241bcb48abe5338d56110?workspace=benchflow) | `clean-prefetch-cpu-kv-offload-nvme` |
| Reactive control | [a34cca262119453a9837a2531c79c3de](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/a34cca262119453a9837a2531c79c3de?workspace=benchflow) | `multi-tier-offloading-nvme` |

Both cells used image digest `sha256:1715fa537b2f754ebec797cb7a47ab9fe2d0ff061f407c2df531c4bf5b338b46`, FP8 Nemotron 253B, TP=8, one replica, RHOAI 3.5.0 on `stable-3.5`, 131,072-token maximum context, 256 GiB CPU KV capacity, local NVMe with 64 read and 64 write workers, AgentX Weka traces, concurrency 64, and the vLLM default `max_num_seqs`.

The treatment used:

- mode `working_set`;
- 8,192 maximum candidate chunks;
- 256 chunks applied per scheduling step;
- load batches of 64;
- at most 64 pending loads;
- one scheduler-ordered owner;
- 8,192 prefetch evictions allowed per request.

The cells ran on different H100 nodes. Neither restarted or reported an OOM, vLLM traceback, proactive load failure, or submission after demand. Both AIPerf clients ended with the same known final cancelled-credit timeout. The profile records report no cancelled requests, but the treatment contains 1,244 requests versus 1,242 in control. This is acceptable for inspecting the mechanism, but it weakens a small performance delta.

## End-to-end result

| Metric | Treatment | Control | Treatment delta | Interpretation |
|---|---:|---:|---:|---|
| Request throughput | 0.6761 req/s | 0.6750 req/s | +0.16% | Neutral |
| Total-token throughput | 43,045.9 tok/s | 42,999.4 tok/s | +0.11% | Neutral |
| Output-token throughput | 457.1 tok/s | 452.4 tok/s | +1.04% | Small positive |
| Mean TTFT | 8.158 s | 8.313 s | -1.87% | Small positive |
| p95 TTFT | 23.716 s | 23.457 s | +1.11% | Small negative |
| p99 TTFT | 31.093 s | 29.192 s | +6.51% | Negative tail |
| Mean end-to-end latency | 43.638 s | 44.005 s | -0.83% | Small positive |
| p95 end-to-end latency | 134.632 s | 137.602 s | -2.16% | Small positive |
| Mean ITL | 54.31 ms | 56.51 ms | -3.89% | Positive |
| p95 ITL | 91.74 ms | 97.79 ms | -6.19% | Positive |

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Treatment relative to control","width":660,"height":300,"data":{"values":[{"metric":"Request throughput","delta":0.16,"class":"Throughput"},{"metric":"Output throughput","delta":1.04,"class":"Throughput"},{"metric":"Mean TTFT","delta":-1.87,"class":"Latency"},{"metric":"p95 TTFT","delta":1.11,"class":"Latency"},{"metric":"p99 TTFT","delta":6.51,"class":"Latency"},{"metric":"Mean E2E","delta":-0.83,"class":"Latency"},{"metric":"p95 E2E","delta":-2.16,"class":"Latency"},{"metric":"Mean ITL","delta":-3.89,"class":"Latency"},{"metric":"p95 ITL","delta":-6.19,"class":"Latency"}]},"mark":{"type":"bar"},"encoding":{"y":{"field":"metric","type":"nominal","sort":null,"title":null},"x":{"field":"delta","type":"quantitative","title":"Relative delta (%) — positive is good only for throughput"},"color":{"field":"class","type":"nominal","scale":{"domain":["Throughput","Latency"],"range":["#2ca02c","#d62728"]}},"tooltip":[{"field":"metric","type":"nominal"},{"field":"delta","type":"quantitative","format":".2f","title":"Delta (%)"}]}}
~~~

A native-record pairing by trace and turn found 1,197 common requests. Treatment minus control had a -138 ms mean and -20 ms median TTFT difference, with treatment faster for 53.6% of pairs. These records share sessions and queue dynamics and are not independent samples; the result is useful as a consistency check, not a significance claim.

Warmup also slightly favored treatment: mean TTFT was 27.85 s versus 28.38 s. Because prefetch was active during warmup and the nodes differ, this cannot isolate a hardware baseline.

## What the prefetch mechanism actually did

### Request-level outcome

| Metric | Count | Share |
|---|---:|---:|
| Admission intents | 2,638 | 100% |
| Requests receiving the single owner | 934 | 35.41% |
| Actual full working set ready at first connector lookup | 20 | **0.76%** |
| Deferred at first connector lookup | 2,618 | **99.24%** |
| Eventually ready after first lookup | 2,618 | 99.24% |
| Manager-local “prefetch complete at first lookup” | 754 | 28.58% |

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Actual connector outcome at first lookup","width":620,"height":220,"data":{"values":[{"outcome":"Ready","requests":20},{"outcome":"Deferred","requests":2618}]},"mark":{"type":"bar"},"encoding":{"y":{"field":"outcome","type":"nominal","sort":["Ready","Deferred"],"title":null},"x":{"field":"requests","type":"quantitative","title":"Request intents"},"color":{"field":"outcome","type":"nominal","scale":{"domain":["Ready","Deferred"],"range":["#2ca02c","#d62728"]},"legend":null},"tooltip":[{"field":"outcome","type":"nominal"},{"field":"requests","type":"quantitative"}]}}
~~~

The decisive measurement is the connector's actual ready/deferred outcome, not the manager-local completion counter. With only one owner, 64.6% of intents never obtained a speculative plan before first demand. Even among planned requests, full coverage was rarely ready in time.

### Chunk-level outcome

| Metric | Count | Derived interpretation |
|---|---:|---|
| Candidate chunks | 10,427,118 | Full requested windows |
| Source-resident at admission probe | 9,147,492 | 87.73% of candidates |
| Already primary-redundant | 2,355,406 | No transfer needed |
| Submitted/promoted | 676,388 | 6.49% of candidates |
| Eventually useful | 673,320 | 99.55% of promoted |
| Late at first observation | 160,688 | 23.76% of promoted |
| Wasted | 3,068 | 0.45% of promoted |
| Cancelled | 6,972,610 | 66.87% of candidates |
| Expired before submission | 4,458,998 | Main cancellation class |
| Cancelled at first demand | 2,513,612 | Work lost to deadline |
| Evicted for prefetch | 676,388 | One ordinary CPU victim per promotion |
| Observed eviction regret | 131,276 | At least 19.41% of victims |
| Eviction outcome untracked | 438,312 | 64.80% of victims |
| Restaged after eviction | 84,280 | 12.46% of promotions |

“99.55% useful” is not evidence that the request was accelerated. It says the individual promoted chunks were eventually demanded. The request still waits when any required external chunk is absent. The 0.76% request-readiness rate and unchanged asynchronous lookup time are the critical-path metrics.

The eviction-regret ratio is a lower bound: the bounded history overflowed for almost 65% of victims. Every proactive promotion displaced an ordinary primary-cache block, so incomplete staging may trade one future miss for another without making the target request runnable.

## Why the two completion metrics disagree

The 754 manager-local completions and 20 connector-ready requests measure different things.

The scheduler initially constructs all complete prompt keys. During plan application, the manager probes source residency. At the first source miss, it shrinks `prefetch_target_chunks` to the current cursor and terminates the plan. Later, `on_prefetch_first_lookup` calls this shortened selected target complete when the corresponding promoted subset is ready.

The connector subsequently performs the authoritative lookup. A filesystem block that missed the earlier admission probe may have become resident through ongoing mirroring by first demand. The connector can therefore discover a longer external working set and defer the request despite the manager reporting its shortened target complete.

This behavior is mechanically explainable but makes the current metric name misleading. It also violates the oracle experiment's intended denominator: a temporary admission-time source miss must not silently redefine “complete working set.”

Required instrumentation correction:

1. retain the immutable original candidate target;
2. report selected/probeable target completion separately from connector full-working-set readiness;
3. count admission probe misses that become hits at first lookup;
4. record contiguous source-resident prefix at admission and first lookup.

## Timing and resource evidence

- Rolling working-set target averaged about 3,810 chunks, while ready coverage averaged about 727 chunks and 25.5%.
- Admission-to-first-lookup p50 averaged 1.21 s and p90 8.73 s.
- Ready-delay p90 averaged 4.81 s.
- After first lookup, time-to-ready p50 averaged 1.32 s and p90 10.76 s. The transfer is therefore still exposed on the request's critical path.
- Admission policy overhead was small: p90 admission about 4.1 ms, application 1.3 ms, and probing 47.6 ms.
- Asynchronous lookup duration was 4.47 s treatment versus 4.41 s control: no reduction.
- Mean capacity-waiting requests were 2.08 treatment versus 1.48 control.
- NVMe reads increased from about 611 to 628 MB/s, while CPU-to-GPU transfer fell from 7.06 to 6.82 GB/s.
- No GPU/DCGM telemetry or direct clean-image CPU-cache free/evictable gauges were exported.

The bottleneck is not policy CPU overhead. It is insufficient request coverage before the scheduling deadline, compounded by eviction churn.

## Verdict

### What this run validates

- The intended image and configuration were deployed.
- Working-set selection and proactive NVMe-to-CPU movement executed.
- Submitted blocks were overwhelmingly real future demand.
- The implementation did not load-fail or submit after demand.
- Reactive fallback completed all deferred requests.

### What this run rejects

For this workload and configuration, **admission-time, one-owner, probe-once working-set staging does not create a request-level oracle**. Correct individual chunks are moved, but the complete working set is not ready early enough to remove reactive deferral. The near-neutral performance result is therefore expected.

### What it does not answer

It does not determine whether a genuine perfect-residency oracle could improve AgentX. That test still requires all of a selected request's external working set to exist in the source and reach CPU before first lookup.

## Next actions

1. **Do not increase the 8,192-chunk ceiling.** No working set was clipped; ownership, source readiness, and lead time are the limiting factors.
2. Fix the metric and target semantics: preserve the original target and distinguish selected-subset completion from authoritative request readiness.
3. Make the oracle condition real: prepopulate or gate on source residency, or re-probe blocks that are still being mirrored instead of terminating the plan at the first transient miss.
4. Improve eviction telemetry before judging economics; the current history loses nearly 65% of victim outcomes.
5. Only after the treatment produces materially complete requests should owner width or deadline-feasible multi-request scheduling be tested.
6. Repeat with a same-node crossover and require at least 50% fewer deferred requests plus either 5% lower mean/p95 TTFT or 3% higher throughput.

## Bottom line

The result is not a mysterious regression or evidence of a broken image. It is a coherent failure of timeliness and request-level completeness: the mechanism moved the right chunks, but mostly did not finish the right request before demand.