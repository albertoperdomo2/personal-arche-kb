---
title: "COSTAR-KV Experiment 8 — held-out age-conditioned reuse hazard"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "COSTAR-KV future-value placement / Experiment 8"
status: "valid-negative-result"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
vllm_version: "v0.27.0-derived offline tooling"
code_base_revision: "e50f7d36960980c0c89651ffd0ce281a9fb8a466 plus uncommitted tools/costar hazard-ranking code"
tensor_parallelism: 8
concurrency:
  C32: 32
  C64: 64
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
secondary_tier: "filesystem-backed node-local NVMe"
workload: "AgentX/Weka; bidirectional C32/C64 pressure holdout"
random_seed: 20260707
duration_seconds: 1800
cache_cleaning_state: "not explicitly recorded in MLflow"
source_runs:
  C32: "f0ea8db6be2044d9a3affbaffbbb87a0"
  C64: "f306ab08fb1045c3af877439b778d62e"
hazard_horizons_seconds: [30, 120, 600]
hazard_age_boundaries_seconds: [0, 10, 30, 60, 120, 300, 600, 1200]
prefix_bands_chunks: ["0-63", "64-255", "256-1023", "1024+"]
---

# COSTAR-KV Experiment 8 — held-out age-conditioned reuse hazard

## Executive summary

This experiment tests whether the missing ingredient in Experiment 7 was **time**: can an age-conditioned reuse hazard, evaluated at a real full-cache arrival-versus-LRU-victim decision, approximate future-aware placement better than a static bundle score?

It cannot with the current trace signals.

Piecewise empirical hazards were trained on all observable reuse/censor intervals in one pressure condition and frozen on the other. Models use age alone, prompt band, source-request reuse context, online absolute prefix position, and their combination. They are evaluated at 30-, 120-, and 600-second horizons over 499,673 informative C64 decisions and 91,550 informative C32 decisions.

No hazard variant beats the stronger of two trivial actions—always admit the new mirror or keep the current LRU victim—in both directions. The learned hazard shape largely collapses to one global action and reverses across pressure:

- C32-trained aggregate hazard keeps the old victim on C64;
- C64-trained aggregate, request, and prefix hazards admit the candidate on C32.

This is a **valid no-go result for trace-only static and age-conditioned context**. It does not falsify the equal-capacity placement opportunity. It says the currently recorded identities do not explain that opportunity robustly enough.

## Validity verdict

### Valid offline information diagnostic — no-go for cache replay

The experiment preserves the accepted C32/C64 targets and capacity. It adds no future information to features:

- an ordinary mirror starts a value generation;
- a reactive native-demand promotion does not erase the earlier mirror's counterfactual value;
- a later ordinary mirror censors the older generation;
- every demand resets the recurrent time-to-next-use interval;
- hazard tables are fitted only on the training execution;
- the test execution uses frozen age buckets, horizons, smoothing, prompt boundaries, and prefix bands.

The LRU state is always-admit and is not changed by a hazard prediction. This isolates information quality from counterfactual cache feedback.

The utility metric is a diagnostic, not TTFT or measured device service:

$$
U(\Delta t;H)=e^{-\Delta t/H}
$$

For each pair, utility recovery is selected utility divided by clairvoyant next-use utility. It values imminent reuse more than distant reuse and exposes mistakes hidden by raw pair counts.

## Main takeaways

- **Measured:** C64 has 499,673 informative full-cache pairs; the arriving candidate is needed sooner in 65.87%.
- **Measured:** C32 has 91,550 informative pairs; the candidate is sooner in 54.91%, but rare old-victim wins are vastly more urgent at the 30-second horizon.
- **Measured:** C32→C64 request+prefix hazard recovers 54.30%, 54.30%, and 65.96% of oracle utility at 30/120/600 seconds. Always admit recovers 68.52%, 78.73%, and 84.44%.
- **Measured:** C64→C32 request+prefix hazard collapses to always admit: 0.58%, 34.05%, and 60.91% utility. Keeping the LRU victim recovers 99.47% and 70.20% at 30 and 120 seconds.
- **Measured:** online prefix bands do not rescue the model in either direction.
- **Inference:** pressure changes not only reuse prevalence but the conditional urgency distribution. A hazard learned at one concurrency is not transportable to the other.
- **Decision:** do not replay or implement these hazard scores. Add stable session/workflow/routing identity before another future-value model.

## Training evidence and decision populations

| Metric | C32 | C64 |
|---|---:|---:|
| Ordinary generations | 368,265 | 680,286 |
| Reuse intervals | 525,548 | 3,765,091 |
| Observed reuse events | 157,283 | 3,084,805 |
| Censored intervals | 368,265 | 680,286 |
| Informative test pairs | 91,550 | 499,673 |
| Candidate needed sooner | 50,270 (54.91%) | 329,135 (65.87%) |
| LRU-victim age p50 | 439.27 s | 148.42 s |
| LRU-victim age p95 | 575.97 s | 460.92 s |

The very different decision ages and reuse prevalence are measured workload properties. They make a pressure-independent age prior difficult, but a useful conditional model would still need to outperform the appropriate simple action after seeing context.

## Hazard variants

All age-conditioned variants use fixed age buckets at 0, 10, 30, 60, 120, 300, 600, and 1,200 seconds.

- **Aggregate:** age only.
- **Prompt:** age and training-corpus prompt quartile.
- **Request context:** age, prompt quartile, and whether the originating request itself reused external KV.
- **Prefix:** age and online absolute source-bundle position.
- **Request + prefix:** all preceding context.

Prefix position uses fixed, online-legal bands: chunks 0–63, 64–255, 256–1023, and 1024+. It is not a radix-tree frontier and does not claim to represent exact contiguous-prefix marginal value.

## Headline outcomes

### C32 trained, C64 held out

| Model | Pair accuracy | Utility recovery H=30 s | H=120 s | H=600 s |
|---|---:|---:|---:|---:|
| Always admit candidate | **65.87%** | **68.52%** | **78.73%** | **84.44%** |
| Keep LRU victim | 34.13% | 48.88% | 54.30% | 59.35% |
| Static request context | 51.69% / 51.69% / 46.69% | 58.50% | 66.93% | 68.75% |
| Aggregate hazard | 34.13% | 48.88% | 54.30% | 59.35% |
| Prompt hazard | 38.02% / 34.13% / 43.93% | 54.25% | 54.30% | 64.64% |
| Request-context hazard | 37.78% / 34.13% / 44.83% | 50.84% | 54.30% | 65.55% |
| Prefix hazard | 37.71% / 34.13% / 37.97% | 52.94% | 54.30% | 62.79% |
| Request + prefix hazard | 40.40% / 34.13% / 44.04% | 54.30% | 54.30% | 65.96% |
| Clairvoyant next use | 100% | 100% | 100% | 100% |

Where pair accuracy changes with horizon, the three values are H=30/H=120/H=600.

### C64 trained, C32 held out

| Model | Pair accuracy | Utility recovery H=30 s | H=120 s | H=600 s |
|---|---:|---:|---:|---:|
| Always admit candidate | 54.91% | 0.58% | 34.05% | **60.91%** |
| Keep LRU victim | 45.09% | **99.47%** | **70.20%** | 52.93% |
| Static request context | 48.36% / 49.31% / 46.99% | 45.97% | 43.28% | 49.88% |
| Aggregate hazard | 54.91% | 0.58% | 34.05% | **60.91%** |
| Prompt/request/prefix hazards | 54.91% | 0.58% | 34.05% | **60.91%** |
| Clairvoyant next use | 100% | 100% | 100% | 100% |

The C64-trained conditional tables produce the same action as aggregate age on C32. Context does not change a single headline outcome.

## Evidence

Figure 1 shows the most important cost-aware comparison at the 30-second horizon. Provenance: every informative full-capacity pair; C32 and C64 models are frozen cross-condition.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Short-horizon utility exposes opposite pressure regimes","width":740,"height":310,"data":{"values":[{"direction_model":"C32→C64 · always admit","utility_percent":68.52,"direction":"C32→C64"},{"direction_model":"C32→C64 · keep victim","utility_percent":48.88,"direction":"C32→C64"},{"direction_model":"C32→C64 · static context","utility_percent":58.50,"direction":"C32→C64"},{"direction_model":"C32→C64 · request + prefix hazard","utility_percent":54.30,"direction":"C32→C64"},{"direction_model":"C32→C64 · clairvoyant","utility_percent":100.0,"direction":"C32→C64"},{"direction_model":"C64→C32 · always admit","utility_percent":0.58,"direction":"C64→C32"},{"direction_model":"C64→C32 · keep victim","utility_percent":99.47,"direction":"C64→C32"},{"direction_model":"C64→C32 · static context","utility_percent":45.97,"direction":"C64→C32"},{"direction_model":"C64→C32 · request + prefix hazard","utility_percent":0.58,"direction":"C64→C32"},{"direction_model":"C64→C32 · clairvoyant","utility_percent":100.0,"direction":"C64→C32"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"direction_model","type":"nominal","title":"Frozen direction and decision rule","sort":null,"axis":{"labelAngle":-28}},"y":{"field":"utility_percent","type":"quantitative","title":"30-second discounted oracle utility recovered (%)","scale":{"zero":true,"domain":[0,100]}},"color":{"field":"direction","type":"nominal","title":"Training→test direction","scale":{"scheme":"category10"}},"tooltip":[{"field":"direction_model","type":"nominal","title":"Evaluation"},{"field":"utility_percent","type":"quantitative","title":"Utility recovered (%)","format":".2f"}]}}
~~~

Raw pair accuracy would favor admitting the candidate in C32 because candidates win 54.91% of pairs. Figure 1 shows why that conclusion is wrong: the 45.09% of victim wins contain essentially all imminent 30-second value.

Figure 2 compares the combined request+prefix hazard with the best of the two trivial actions at each horizon. Provenance: same native-granularity pair population; “best simple” is reported diagnostically after showing both constituent actions in the tables, not as a deployable oracle chooser.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — No hazard closes the gap to the stronger simple action","width":720,"height":300,"data":{"values":[{"case":"C32→C64 · H30 · best simple","utility_percent":68.52,"direction":"C32→C64"},{"case":"C32→C64 · H30 · hazard","utility_percent":54.30,"direction":"C32→C64"},{"case":"C32→C64 · H120 · best simple","utility_percent":78.73,"direction":"C32→C64"},{"case":"C32→C64 · H120 · hazard","utility_percent":54.30,"direction":"C32→C64"},{"case":"C32→C64 · H600 · best simple","utility_percent":84.44,"direction":"C32→C64"},{"case":"C32→C64 · H600 · hazard","utility_percent":65.96,"direction":"C32→C64"},{"case":"C64→C32 · H30 · best simple","utility_percent":99.47,"direction":"C64→C32"},{"case":"C64→C32 · H30 · hazard","utility_percent":0.58,"direction":"C64→C32"},{"case":"C64→C32 · H120 · best simple","utility_percent":70.20,"direction":"C64→C32"},{"case":"C64→C32 · H120 · hazard","utility_percent":34.05,"direction":"C64→C32"},{"case":"C64→C32 · H600 · best simple","utility_percent":60.91,"direction":"C64→C32"},{"case":"C64→C32 · H600 · hazard","utility_percent":60.91,"direction":"C64→C32"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"case","type":"nominal","title":"Direction, horizon, and decision rule","sort":null,"axis":{"labelAngle":-28}},"y":{"field":"utility_percent","type":"quantitative","title":"Discounted oracle utility recovered (%)","scale":{"zero":true,"domain":[0,100]}},"color":{"field":"direction","type":"nominal","title":"Training→test direction","scale":{"scheme":"category10"}},"tooltip":[{"field":"case","type":"nominal","title":"Evaluation"},{"field":"utility_percent","type":"quantitative","title":"Utility recovered (%)","format":".2f"}]}}
~~~

The hazard never provides robust conditional selection. In the reverse short-horizon test it makes the exact globally wrong choice.

Figure 3 shows the pressure-dependent victim ages at the actual informative decisions. Provenance: always-admit LRU shadow state at full capacity; candidate age is always zero.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3 — LRU-victim ages differ materially across pressure","width":620,"height":280,"data":{"values":[{"condition_stat":"C32 · p50","age_seconds":439.27,"condition":"C32"},{"condition_stat":"C32 · p95","age_seconds":575.97,"condition":"C32"},{"condition_stat":"C64 · p50","age_seconds":148.42,"condition":"C64"},{"condition_stat":"C64 · p95","age_seconds":460.92,"condition":"C64"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"condition_stat","type":"nominal","title":"Test condition and percentile","sort":null},"y":{"field":"age_seconds","type":"quantitative","title":"LRU-victim age since last use (s)","scale":{"zero":true}},"color":{"field":"condition","type":"nominal","title":"Test condition","scale":{"scheme":"category10"}},"tooltip":[{"field":"condition_stat","type":"nominal","title":"Condition / percentile"},{"field":"age_seconds","type":"quantitative","title":"Victim age (s)","format":".2f"}]}}
~~~

The age shift helps explain transport failure, but it is not itself a useful selector: the conditional tables learned from one regime do not identify which old victims carry imminent value in the other.

## What this falsifies

The following variants should not proceed to cache replay or live implementation on the present evidence:

1. static prompt/request/bundle scoring;
2. aggregate age-conditioned hazard;
3. prompt-conditioned hazard;
4. source-reuse-conditioned hazard;
5. coarse absolute prefix-position hazard;
6. combinations of those fields.

This does **not** establish that all hazard or prefix-aware policies are ineffective. The current trace lacks stable session identity, request category, active/suspended state, turn structure, tool/DAG state, tenant/application class, and routing lineage. It also lacks an exact online radix-prefix frontier. Those are precisely the signals that the research program and adjacent workload-aware systems expect to make conditional reuse stable.

## Decision and next experiment

### Stop adding generic models to the current feature set

Experiments 7 and 8 now fail two distinct approximations:

- timeless expected value;
- time-varying conditional reuse.

The next experiment is an **instrumentation and data-sufficiency experiment**, not another policy.

1. Inspect the AgentX/Weka source records for stable conversation/session and turn lineage.
2. Propagate privacy-safe hashes and structural fields into the oracle trace:
   - session/conversation ID;
   - turn index and parent request;
   - seconds since prior turn;
   - active, suspended, tool-running, or finished state;
   - tool/DAG successor information;
   - workload/application class;
   - origin and destination replica/routing identity.
3. Run a small trace and verify lifecycle closure plus identity coverage.
4. Repeat only the information gate: does session/workflow context improve cross-run, cost-weighted arrival-versus-victim utility over the stronger simple action?
5. Only then allow a counterfactual cache replay.

Stop/go criterion: if stable activity identity does not produce positive held-out utility lift across at least two runs per pressure condition, stop increasing predictor complexity and focus on route-to-data, explicit application cache intent, or deterministic policy controls.

## Reproduction and verification

~~~text
../vllm/.venv/bin/python -m tools.costar.run_hazard_ranking \
  /private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite \
  /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite

../vllm/.venv/bin/python -m pytest tests/tools/test_costar_*.py -q
ruff check tools/costar tests/tools/test_costar_*.py
git diff --check
~~~

Verification: 49 COSTAR tests passed; Ruff and diff checks passed. Nothing is committed.

## Source runs

- C32: [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- C64: [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow)

Related: [[07 - Experiment 7 held-out soft bundle-value ranking]], [[06 - Experiment 6 C64 independent pressure validation]], and [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching]].