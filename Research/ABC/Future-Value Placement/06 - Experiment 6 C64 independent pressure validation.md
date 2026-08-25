---
title: "COSTAR-KV Experiment 6 — C64 independent pressure validation"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "COSTAR-KV future-value placement / Experiment 6"
status: "valid-trace-conditionally-valid-policy-replay"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
vllm_version: "v0.27.0-derived offline tooling"
code_base_revision: "e50f7d36960980c0c89651ffd0ce281a9fb8a466 plus uncommitted tools/costar experiment code"
tensor_parallelism: 8
concurrency: 64
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
secondary_tier: "filesystem-backed node-local NVMe"
workload: "AgentX/Weka C64"
random_seed: 20260707
duration_seconds: 1800
cache_cleaning_state: "not explicitly recorded in MLflow"
source_run: "f306ab08fb1045c3af877439b778d62e"
trace_bytes: 6959277072
trace_sha256: "1167b512741bb97d2b76744cb238ede58fa0c6c2ef35ad7b3e9892c05b4ece3d"
---

# COSTAR-KV Experiment 6 — C64 independent pressure validation

## Executive summary

This experiment asks whether the C32 future-value findings survive a much higher-pressure AgentX execution and whether C32-derived contextual rules generalize.

The complete 6.959 GB C64 oracle trace was recovered from MLflow, hashed, normalized, and validated. It contains 13,629,779 events, 1,280 closed request lifecycles, 2,480 closed transfers, and exact 131,072-slot capacity conservation.

C64 strongly confirms the **research opportunity and problem framing**:

- recorded placement completes 577/789 external targets;
- matched clairvoyant next-use placement completes 716/789 and avoids 144/212 native reads plus 540.369/803.642 seconds of measured device service;
- forced-admit and bypass-capable next-use produce the same request and movement outcome;
- 91.83% of valuable ordinary arrivals still have no exact-key history;
- 86.03% of target keys come from the dominant originating request.

It also rejects the tested practical policies. LRU is the least harmful generic policy at 537/789 targets, 40 net misses worse than recorded. The frozen C32 prompt-band rule collapses to 209/789 targets and +368 net misses. Whole-bundle LRU is slightly better than blockwise bundle LRU but remains 38 net misses worse than recorded.

## Validity verdict

### Trace and target reconstruction — Valid

- Exact artifact size: 6,959,277,072 bytes.
- SHA-256: `1167b512741bb97d2b76744cb238ede58fa0c6c2ef35ad7b3e9892c05b4ece3d`.
- Normalized events: 13,629,779.
- Requests: 1,280 arrived, first-looked-up, ready, and closed.
- Transfers: 2,480 complete; zero incomplete I/O or result observations.
- Maximum and final CPU occupancy: 131,072/131,072.
- Native movement reconstruction: 0/1,263 non-aborted request mismatches.

### Policy counterfactual — Conditionally valid

Policies receive the same recorded CPU-ready arrivals and equal capacity. Counterfactual demand-fill feedback and protected-block constraints are not modeled. Gross service avoided cannot be netted against newly created reads because their counterfactual service is unknown.

C64 is a held-out execution/pressure condition relative to C32, but both use the same AgentX dataset family and seed. It is not an independent workload distribution.

## Correctness issue discovered and fixed

The offline target loaders selected the **last** resolved connector lookup. Under C64, 34 requests had later resolved cycles using a newer decode-era working-set version. One representative request:

- first resolved cycle: version 1, 3,090 matched chunks after a 3,089-chunk native read;
- later resolved cycle: version 2, zero external chunks.

Using the later result falsely made the native read appear unrelated to the target. The corrected initial-placement target is the **first resolved connector lookup**. The retention deadline is:

- earliest native-demand submission when a native read occurs;
- first resolved lookup when no native read occurs.

After this correction, C64 movement mismatches fall from 88 at the original first-lookup boundary, then 34 with the wrong final target, to **zero**. C32 remains at zero and its headline results are unchanged.

The fix is applied consistently in:

- `tools/costar/finite_retention.py`
- `tools/costar/global_oracle.py`
- `tools/costar/l0_oracle.py`
- `tools/costar/trace_db.py`

Regression tests now contain a later resolved working-set version and require the first resolved target.

## C32 versus C64 opportunity

| Metric | C32 | C64 |
|---|---:|---:|
| Total requests | 901 | 1,280 |
| Requests with external target | 42 | 789 |
| External key references | 157,283 | 3,084,805 |
| Native reads | 12 | 212 |
| Native device service | 36.440 s | 803.642 s |
| Recorded complete external targets | 30/42 (71.43%) | 577/789 (73.13%) |
| Matched next-use complete targets | 42/42 (100%) | 716/789 (90.75%) |
| Native reads avoided | 12/12 (100%) | 144/212 (67.92%) |
| Device service avoided | 36.440 s (100%) | 540.369 s (67.24%) |

Figure 1 shows request-level placement headroom at equal CPU capacity. Provenance: exact external targets and physical CPU arrival streams from the two validated corpora.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Recorded versus matched clairvoyant request completeness","width":650,"height":300,"data":{"values":[{"condition_policy":"C32 · recorded","complete_percent":71.43,"condition":"C32"},{"condition_policy":"C32 · next-use","complete_percent":100.0,"condition":"C32"},{"condition_policy":"C64 · recorded","complete_percent":73.13,"condition":"C64"},{"condition_policy":"C64 · next-use","complete_percent":90.75,"condition":"C64"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"condition_policy","type":"nominal","title":"Pressure condition and placement","sort":null,"axis":{"labelAngle":-18}},"y":{"field":"complete_percent","type":"quantitative","title":"External targets CPU-complete (%)","scale":{"zero":true,"domain":[0,100]}},"color":{"field":"condition","type":"nominal","title":"Pressure condition","scale":{"scheme":"category10"}},"tooltip":[{"field":"condition_policy","type":"nominal","title":"Condition"},{"field":"complete_percent","type":"quantitative","title":"Complete targets (%)","format":".2f"}]}}
~~~

C64 does not reproduce C32's perfect matched result, but the absolute opportunity is much larger: 139 additional complete targets and 540.37 seconds of measured native service.

## Information audit

| Admission signal/structure | C32 | C64 |
|---|---:|---:|
| Valuable ordinary arrival intervals | 31.61% | 74.10% |
| Valuable with no exact-key history | 100.00% | 91.83% |
| Valuable with no prior exact-key demand | 100.00% | 94.28% |
| Median / p90 time to demand | 398.49 / 1,002.18 s | 130.06 / 744.86 s |
| Target keys from dominant origin | 83.69% | 86.03% |
| Targets at least 90% one origin | 54.76% | 65.15% |
| Distinct origins per target, median / p95 | 3 / 18 | 3 / 18 |

Figure 2 shows which structural findings reproduce. Provenance: interval labels for all ordinary CPU-ready arrivals and latest prior origins for all external key references.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — Reproduced admission and request-structure evidence","width":700,"height":300,"data":{"values":[{"condition_measure":"C32 · valuable arrivals","percent":31.61,"condition":"C32"},{"condition_measure":"C64 · valuable arrivals","percent":74.10,"condition":"C64"},{"condition_measure":"C32 · valuable without history","percent":100.0,"condition":"C32"},{"condition_measure":"C64 · valuable without history","percent":91.83,"condition":"C64"},{"condition_measure":"C32 · keys from dominant origin","percent":83.69,"condition":"C32"},{"condition_measure":"C64 · keys from dominant origin","percent":86.03,"condition":"C64"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"condition_measure","type":"nominal","title":"Condition and measure","sort":null,"axis":{"labelAngle":-25}},"y":{"field":"percent","type":"quantitative","title":"Share (%)","scale":{"zero":true,"domain":[0,100]}},"color":{"field":"condition","type":"nominal","title":"Pressure condition","scale":{"scheme":"category10"}},"tooltip":[{"field":"condition_measure","type":"nominal","title":"Measure"},{"field":"percent","type":"quantitative","title":"Share (%)","format":".2f"}]}}
~~~

The base value rate changes with pressure, but the decisive structural findings remain: most valuable blocks are cold at first admission, and demand is request/prefix coherent.

## Prompt-size association: replicated descriptively, failed as policy

C32-derived boundaries were frozen at 44,234, 62,413, and 90,398 tokens. C64's independently computed quartiles are close: 45,383, 63,051, and 86,762.

Figure 3 compares descriptive arrival value by prompt quartile. Provenance: per-trace quartile descriptions; this chart does not use the fixed boundaries for C64.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3 — Arrival value by source-prompt quartile","width":700,"height":300,"data":{"values":[{"condition_bucket":"C32 · Q1","value_percent":20.94,"condition":"C32"},{"condition_bucket":"C32 · Q2","value_percent":39.68,"condition":"C32"},{"condition_bucket":"C32 · Q3","value_percent":34.01,"condition":"C32"},{"condition_bucket":"C32 · Q4","value_percent":29.95,"condition":"C32"},{"condition_bucket":"C64 · Q1","value_percent":70.30,"condition":"C64"},{"condition_bucket":"C64 · Q2","value_percent":82.19,"condition":"C64"},{"condition_bucket":"C64 · Q3","value_percent":78.69,"condition":"C64"},{"condition_bucket":"C64 · Q4","value_percent":66.47,"condition":"C64"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"condition_bucket","type":"nominal","title":"Pressure condition and prompt quartile","sort":null,"axis":{"labelAngle":-22}},"y":{"field":"value_percent","type":"quantitative","title":"Valuable ordinary arrivals (%)","scale":{"zero":true}},"color":{"field":"condition","type":"nominal","title":"Pressure condition","scale":{"scheme":"category10"}},"tooltip":[{"field":"condition_bucket","type":"nominal","title":"Bucket"},{"field":"value_percent","type":"quantitative","title":"Valuable arrivals (%)","format":".2f"}]}}
~~~

The non-monotonic pattern reproduces: Q2 is hottest and Q4 is colder. However, using the frozen C32 bands as a lexicographic bypass priority on C64 completes only 209/789 targets and creates 368 net additional misses. A marginal association is not a safe hard placement decision.

## Practical policy results

Recorded placement completes 577/789 external targets.

| Policy | Complete targets | Key-hit rate | Reads avoided | New misses | Net miss delta |
|---|---:|---:|---:|---:|---:|
| LRU | 537/789 (68.06%) | 69.12% | 35 | 75 | **+40** |
| Bundle LRU | 536/789 (67.93%) | 69.36% | 31 | 72 | **+41** |
| Whole-bundle LRU | 539/789 (68.31%) | 68.22% | 32 | 70 | **+38** |
| Source-reuse bypass | 306/789 (38.78%) | 58.62% | 56 | 327 | **+271** |
| Frozen prompt-band bypass | 209/789 (26.49%) | 39.46% | 47 | 415 | **+368** |
| Combined context bypass | 216/789 (27.38%) | 44.67% | 65 | 426 | **+361** |

Figure 4 shows net request substitution; positive values are regressions. Provenance: baseline native-read requests recovered minus previously complete external requests made incomplete.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4 — C64 net external misses versus recorded placement","width":680,"height":290,"data":{"values":[{"policy":"LRU","net_misses":40},{"policy":"Bundle LRU","net_misses":41},{"policy":"Whole-bundle LRU","net_misses":38},{"policy":"Source reuse","net_misses":271},{"policy":"Frozen prompt band","net_misses":368},{"policy":"Combined context","net_misses":361}]},"mark":{"type":"bar","color":"#e45756"},"encoding":{"x":{"field":"policy","type":"nominal","title":"Placement policy","sort":null,"axis":{"labelAngle":-20}},"y":{"field":"net_misses","type":"quantitative","title":"Additional incomplete external requests (count)","scale":{"zero":true}},"tooltip":[{"field":"policy","type":"nominal","title":"Policy"},{"field":"net_misses","type":"quantitative","title":"Net additional misses"}]}}
~~~

Whole-bundle LRU is three net misses better than blockwise bundle LRU, but all tested policies remain below recorded placement.

## Admission decomposition

| Matched clairvoyant policy | Complete targets | Reads avoided | Service avoided | Admissions | Rejections | Evictions |
|---|---:|---:|---:|---:|---:|---:|
| Forced-admit next-use | 716/789 | 144/212 | 540.369 s | 949,932 | 0 | 818,860 |
| Bypass-capable next-use | 716/789 | 144/212 | 540.369 s | 695,289 | 195,322 | 564,217 |

As on C32, bypass contributes churn reduction but no additional request or movement benefit under the matched scoring rule. Future-aware victim ranking remains the main measured lever.

## Conclusions

### Supported across C32 and C64

1. There is substantial equal-capacity retention-placement headroom.
2. Most valuable arrivals lack exact-key history when the decision is first made.
3. Future demand is strongly request/prefix coherent.
4. A future-aware victim order can capture the movement opportunity without mandatory admission rejection.
5. Block-hit ratio and gross recovered reads are unsafe objectives because new incomplete requests dominate.

### Rejected

1. Generic exact-key LRU/ARC/LFU/LRFU as the primary improvement.
2. Source-request grouping without a strong value score.
3. Whole-bundle FIFO/LRU as a sufficient mechanism.
4. Source-external-reuse as a hard priority.
5. C32-derived prompt bands as a hard bypass priority.

### Promising but unproven

Prompt size contains a reproducible non-monotonic association, but it is too weak and coarse for binary admission. It may contribute to a **soft, calibrated expected-value score** alongside stronger session/workflow signals, predicted time to reuse, complete-prefix contribution, size, and reload cost.

## Next experiment

Build an offline request/prefix value-ranking dataset across C32 and C64:

- unit: originating request/prefix bundle, not individual key;
- label: future complete-target contribution, time to first reuse, number of future uses, and measured/estimated reload cost;
- features: prompt size, source external reuse, bundle size, prefix position/coherence, age, pressure state, and any stable session/workflow identifiers available;
- split: train/calibrate on one execution and evaluate ranking on the other;
- policy: soft expected-value-per-byte ranking blended with recency and a completion constraint, not lexicographic hard bypass;
- metric: held-out net complete requests and avoided exposed cost, with bandwidth/churn penalties.

If the available trace fields cannot produce held-out lift over recorded placement or LRU, stop and add stable session/conversation/tool/DAG identity before further policy work.

## Reproduction

~~~text
../vllm/.venv/bin/python -m tools.costar.replay_native   /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite

../vllm/.venv/bin/python -m tools.costar.run_finite_retention   /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite

../vllm/.venv/bin/python -m tools.costar.run_admission_information_audit   /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite

../vllm/.venv/bin/python -m tools.costar.run_practical_policy_benchmark   /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite --compact

../vllm/.venv/bin/python -m tools.costar.run_contextual_policy_benchmark   /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite

../vllm/.venv/bin/python -m tools.costar.run_whole_bundle_benchmark   /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite

../vllm/.venv/bin/python -m tools.costar.run_oracle_decomposition   /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite
~~~

Verification: all 43 COSTAR tests pass; Ruff and `git diff --check` pass.

Source run: [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow).