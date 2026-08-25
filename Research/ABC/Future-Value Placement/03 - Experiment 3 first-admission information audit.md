---
title: "COSTAR-KV Experiment 3 — first-admission information audit"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "COSTAR-KV future-value placement / Experiment 3"
status: "valid-offline-diagnostic"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
vllm_version: "v0.27.0-derived offline tooling"
code_base_revision: "e50f7d36960980c0c89651ffd0ce281a9fb8a466 plus uncommitted tools/costar experiment code"
tensor_parallelism: 8
concurrency: 32
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
secondary_tier: "filesystem-backed node-local NVMe"
workload: "AgentX/Weka C32"
random_seed: "deterministic; no randomized policy"
cache_cleaning_state: "offline audit of recorded native state"
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
corpus_sha256: "788adedcbc9850a64443681fbd8acf36ea62cdce950eeac6311f730f34cce687"
---

# COSTAR-KV Experiment 3 — first-admission information audit

## Executive summary

Experiment 2 found that LRU, FIFO, ARC, LFU/LRFU, 2Q, and LRU-2 all fail to recover the clairvoyant placement opportunity. Experiment 3 asks why: **what information exists when an ordinary mirrored KV block first becomes CPU-ready, before the retention decision must be made?**

An arrival is labeled valuable only when its key is demanded after that arrival and before another free arrival refreshes the same key. This avoids crediting an old generation that did not need to be retained.

The result is decisive:

- 116,409/368,265 ordinary arrivals are valuable under this interval definition;
- **all 116,409 valuable arrivals are first-seen exact keys**;
- **all 116,409 have zero prior exact-key demand history**;
- future demand is strongly request/prefix structured: 83.69% of external key references come from the dominant originating request for that target, and 23/42 external requests derive at least 90% of their target from one origin;
- valuable arrivals have long lead time: 398.49 seconds median, 1,002.18 seconds p90, and 1,739.92 seconds p95 before demand.

The practical implication is not “predict the next block.” It is **retain the right newly created request/prefix bundle for minutes until its likely continuation**.

## Validity verdict — Valid offline diagnostic

The result is valid for the accepted C32 trace and the declared labels. It uses exact arrival/demand sequence ordering and excludes native-demand promotions from the predictive population.

Limits:

- this is one workload trace and only 42 nonempty external-demand requests;
- request IDs are trace-local, not stable session IDs;
- no workload category, user, conversation, tool/DAG, or endpoint-routing identity is present;
- prompt quartile and source-reuse associations are descriptive and are not held-out predictive results;
- dominant-origin analysis describes structure but does not itself provide an online ranking signal.

## Main takeaways

- **Observation:** every valuable ordinary arrival has no exact-key arrival or demand history.
- **Conclusion:** exact-key history policies are structurally unable to identify these keys at the decisive first admission. Experiment 2's failure is expected, not merely a bad parameter choice.
- **Observation:** the dominant source request accounts for 131,626/157,283 external key references (83.69%).
- **Inference:** request/prefix working sets are a much better placement unit than independent blocks.
- **Observation:** source requests that themselves reused external KV have 44.24% valuable-arrival precision versus the 31.61% base rate, a 1.40× lift, but only 3.53% recall.
- **Conclusion:** prior continuation is a useful high-confidence feature, but far too sparse to be the policy alone.
- **Observation:** prompt length is non-monotonic; the second quartile is most valuable at 39.68%, while the first is only 20.94%.
- **Conclusion:** a simple “longer prompt is hotter” rule is contradicted.
- **Observation:** block position within the generated bundle varies only from 30.71% to 32.48% value.
- **Conclusion:** local prefix position alone is weak on this trace.

## Headline metrics

| Quantity | Result |
|---|---:|
| Ordinary CPU-ready arrivals | 368,265 |
| Valuable arrival intervals | 116,409 (31.61%) |
| Valuable arrivals with prior exact-key demand | **0 (0%)** |
| Valuable arrivals with any prior exact-key arrival | **0 (0%)** |
| Median / p90 / p95 time to demand | 398.49 / 1,002.18 / 1,739.92 s |
| External target requests | 42 |
| External key references | 157,283 |
| Keys attributable to dominant origin | 131,626 (83.69%) |
| Single-origin target requests | 14/42 (33.33%) |
| Targets at least 90% from one origin | 23/42 (54.76%) |
| Distinct origins per target, p50 / p95 | 3 / 18 |

## Evidence

Figure 1 compares the precision and recall of signals observable by or shortly after the source request executes. Provenance: all 368,265 ordinary arrival intervals at native event ordering.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — First-admission signal precision and recall","width":720,"height":300,"data":{"values":[{"signal_metric":"Exact key seen before · precision","percent":0.0,"signal":"Exact-key seen"},{"signal_metric":"Exact key seen before · recall","percent":0.0,"signal":"Exact-key seen"},{"signal_metric":"Exact key demanded before · precision","percent":0.0,"signal":"Exact-key demand"},{"signal_metric":"Exact key demanded before · recall","percent":0.0,"signal":"Exact-key demand"},{"signal_metric":"Source reused external KV · precision","percent":44.24,"signal":"Source external reuse"},{"signal_metric":"Source reused external KV · recall","percent":3.53,"signal":"Source external reuse"},{"signal_metric":"Prompt ≥ median · precision","percent":31.79,"signal":"Prompt ≥ median"},{"signal_metric":"Prompt ≥ median · recall","percent":51.00,"signal":"Prompt ≥ median"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"signal_metric","type":"nominal","title":"Signal and metric","sort":null,"axis":{"labelAngle":-28}},"y":{"field":"percent","type":"quantitative","title":"Arrival classification (%)","scale":{"zero":true}},"color":{"field":"signal","type":"nominal","title":"Signal","scale":{"scheme":"category10"}},"tooltip":[{"field":"signal_metric","type":"nominal","title":"Signal metric"},{"field":"percent","type":"quantitative","title":"Percent","format":".2f"}]}}
~~~

Exact-key recency and frequency have zero coverage because the valuable decision occurs on first sight. Source external reuse is meaningfully more precise than the base rate but covers only a small minority of valuable arrivals.

Figure 2 shows categorical value rates for source prompt size and block position within its originating request bundle. Prompt quartile boundaries are 44,234, 62,413, and 90,398 tokens. Provenance: exact ordinary-arrival labels; quartiles are descriptive in-sample groups.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — Future-value rate by admission context","width":680,"height":300,"data":{"values":[{"category":"Prompt Q1","value_percent":20.94,"feature":"Source prompt size"},{"category":"Prompt Q2","value_percent":39.68,"feature":"Source prompt size"},{"category":"Prompt Q3","value_percent":34.01,"feature":"Source prompt size"},{"category":"Prompt Q4","value_percent":29.95,"feature":"Source prompt size"},{"category":"Bundle Q1","value_percent":32.48,"feature":"Bundle position"},{"category":"Bundle Q2","value_percent":32.04,"feature":"Bundle position"},{"category":"Bundle Q3","value_percent":30.71,"feature":"Bundle position"},{"category":"Bundle Q4","value_percent":31.20,"feature":"Bundle position"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"category","type":"nominal","title":"Admission-context bucket","sort":null,"axis":{"labelAngle":-20}},"y":{"field":"value_percent","type":"quantitative","title":"Valuable arrivals (%)","scale":{"zero":true}},"color":{"field":"feature","type":"nominal","title":"Feature","scale":{"scheme":"category10"}},"tooltip":[{"field":"category","type":"nominal","title":"Bucket"},{"field":"value_percent","type":"quantitative","title":"Valuable arrivals (%)","format":".2f"}]}}
~~~

Prompt size contains a possible signal but it is not monotonic and was measured in-sample. Bundle position is nearly flat and should not be pursued alone.

Figure 3 shows where external request targets originate. Provenance: the latest prior physical CPU arrival for every one of the 157,283 authoritative external key references.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3 — Request-level origin coherence","width":570,"height":280,"data":{"values":[{"measure":"Keys from dominant origin","percent":83.69},{"measure":"Requests ≥90% one origin","percent":54.76},{"measure":"Requests exactly one origin","percent":33.33}]},"mark":{"type":"bar","color":"#4c78a8"},"encoding":{"x":{"field":"measure","type":"nominal","title":"Origin-coherence measure","sort":null,"axis":{"labelAngle":-18}},"y":{"field":"percent","type":"quantitative","title":"Share (%)","scale":{"zero":true,"domain":[0,100]}},"tooltip":[{"field":"measure","type":"nominal","title":"Measure"},{"field":"percent","type":"quantitative","title":"Share (%)","format":".2f"}]}}
~~~

The coherence is strong enough to justify testing request-bundle-aware placement. It does not prove that the system can predict *which* source request will be reused.

## What this explains about Experiment 2

The history-only policies were asked to rank keys before those keys had any history. ARC, LFU/LRFU, 2Q, and LRU-2 become informative only after reuse, but the decisive opportunity is the first later reuse of a newly produced prefix.

Their moderate block-hit rates reflect protecting already observed popularity and retaining large portions of targets. Their poor request completeness reflects fragmentation: missing even a small part of a large target still causes reactive movement.

This is why request/prefix completeness and value per retained byte should replace independent block hit rate as the primary objective.

## Decision

Kill exact-key recency/frequency as the primary future-value signal for this workload. Retain LRU/FIFO only as baselines.

Advance two evidence-backed ideas to the next offline replay:

1. **request-bundle-coherent placement**, so blocks created by one request receive a common placement identity and are not valued independently;
2. **simple source-context admission**, beginning with whether the source request reused external KV and coarse prompt-size bands.

The second feature is exploratory because it was selected in-sample. It must be tested against a second trace or with a temporal holdout before any live implementation.

## Next experiment

Run a small contextual policy matrix under the same capacity:

- request-bundle FIFO/LRU baseline;
- source-external-reuse priority;
- coarse prompt-band priority;
- combined request-context priority;
- forced-admit and bypass variants where needed to separate victim ranking from admission rejection.

Report request completeness, net new external misses, target fragmentation, churn, and metadata. If a policy only increases block-hit rate without improving complete targets, reject it.

Then validate any survivor on C64 or a new independently collected trace. Do not deploy a C32-selected prompt threshold live.

## Reproduction and verification

Implementation:

- `tools/costar/admission_information_audit.py`
- `tools/costar/run_admission_information_audit.py`
- metadata extensions in `tools/costar/finite_retention.py`
- `tests/tools/test_costar_admission_information_audit.py`

Command:

~~~text
../vllm/.venv/bin/python -m tools.costar.run_admission_information_audit   /private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite
~~~

Focused verification: 18 tests passed; Ruff passed.

Source corpus: [MLflow C32 run f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow).