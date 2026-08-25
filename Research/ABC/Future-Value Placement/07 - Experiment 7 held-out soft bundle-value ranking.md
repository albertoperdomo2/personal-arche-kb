---
title: "COSTAR-KV Experiment 7 — held-out soft bundle-value ranking"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "COSTAR-KV future-value placement / Experiment 7"
status: "valid-negative-result"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
vllm_version: "v0.27.0-derived offline tooling"
code_base_revision: "e50f7d36960980c0c89651ffd0ce281a9fb8a466 plus uncommitted tools/costar value-ranking code"
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
---

# COSTAR-KV Experiment 7 — held-out soft bundle-value ranking

## Executive summary

This experiment tests the next gate from the future-value research program: **can online request and completed-bundle context rank which CPU-mirrored request bundles deserve future retention, before cache-policy mechanics are added?**

The answer is **not robustly with the currently recorded features**.

A transparent ridge-logistic ranker trained on C32 and frozen on C64 can separate reused from unused bundles at the unweighted request level (ROC AUC 0.674), but it ranks the bytes that carry future value worse than random (byte-weighted AUC 0.472). Its top-ranked 25% of bytes contains only 19.7% of future key references. Training on the larger C64 corpus and testing on C32 reverses the result: request context reaches byte-weighted AUC 0.608 and captures 33.5% of future references in 25% of bytes.

Training directly on future references per block does not repair the asymmetry. C32→C64 remains below random by the byte-weighted and top-byte-budget objectives.

## Validity verdict

### Valid offline information diagnostic — no-go for policy replay

The trace reconstruction is already accepted and reproduces native movement with zero mismatches in both pressure conditions. This experiment reconstructs:

- exactly 42.0 C32 and 789.0 C64 external-target equivalents by fractional source attribution;
- exactly 36.440 and 803.642 seconds of native service attributed to the ordinary mirrors whose retention could have avoided that service;
- 368,265 C32 and 680,286 C64 ordinary-mirror bundle blocks.

Models are trained on one complete execution and frozen on the other. No future label is present in a feature.

This is not an independent workload-distribution validation: C32 and C64 share the AgentX dataset family and seed. That limitation strengthens the negative conclusion because the score fails even across two closely related pressure conditions.

No cache-policy replay was run. That is intentional: the predeclared information gate failed, so adding eviction mechanics would not make the ranking evidence stronger.

## Main takeaways

- **Measured:** C32→C64 request-context reuse AUC is 0.674, but byte-weighted AUC is 0.472.
- **Measured:** the same C32→C64 score captures 19.7% of future references in its top 25.0% of bytes, a 0.788× lift; the direct expected-value target captures 18.5%, a 0.740× lift.
- **Measured:** C64→C32 request context is positive: byte-weighted AUC is 0.608–0.616 and top-25%-byte value lift is 1.335–1.338×.
- **Measured:** adding bundle size, generation duration, ready delay, and observed CPU pressure makes held-out ranking worse in both directions than request context alone.
- **Inference:** the recorded features contain a weak pressure-dependent association, but not a stable placement score.
- **Inference:** an unweighted “reuse prediction” metric can be actively misleading: it rewards correctly ranking many small bundles while misranking the large bundles that dominate capacity and future references.
- **Decision:** do not implement this static score or replay it as an admission/eviction policy. Test a time-varying age-conditioned hazard next.

## Corpus and target reconstruction

| Metric | C32 | C64 |
|---|---:|---:|
| Source-request bundles | 891 | 1,258 |
| Ordinary-mirror bundle blocks | 368,265 | 680,286 |
| Reused bundles | 203 (22.78%) | 965 (76.71%) |
| Future external key references | 157,283 | 3,084,805 |
| Fractionally reconstructed external targets | 42.0 | 789.0 |
| Targets wholly attributable to one ordinary bundle | 12 | 104 |
| Attributed native service | 36.440 s | 803.642 s |

A bundle is scored after its last observed ordinary-mirror arrival, which is the offline proxy for completed-bundle scoring. Features are prompt tokens, maximum output tokens, whether the source request reused external KV, its external-key count, whether it was initially deferred, completed bundle size, duplicate fraction, generation duration, source ready delay, and observed CPU pressure.

The label attribution makes an important causal correction: a reactive native-demand promotion does **not** supersede the earlier ordinary mirror. The promotion exists because native placement failed to retain that mirror. A later ordinary mirror does supersede it because that later write provides another free placement opportunity.

Figure 1 shows the large label/base-rate shift that a deployable score must survive. Provenance: accepted C32 and C64 normalized traces at source-request-bundle granularity.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Bundle reuse prevalence changes sharply with pressure","width":620,"height":280,"data":{"values":[{"condition_outcome":"C32 · reused","outcome":"Reused later","bundles":203},{"condition_outcome":"C32 · not reused","outcome":"Not reused later","bundles":688},{"condition_outcome":"C64 · reused","outcome":"Reused later","bundles":965},{"condition_outcome":"C64 · not reused","outcome":"Not reused later","bundles":293}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"condition_outcome","type":"nominal","title":"Pressure condition and future outcome","sort":null,"axis":{"labelAngle":-18}},"y":{"field":"bundles","type":"quantitative","title":"Source-request bundles (count)","scale":{"zero":true}},"color":{"field":"outcome","type":"nominal","title":"Future outcome","scale":{"scheme":"category10"}},"tooltip":[{"field":"condition_outcome","type":"nominal","title":"Condition / outcome"},{"field":"bundles","type":"quantitative","title":"Bundles"}]}}
~~~

C64's reused-bundle prevalence is 76.71%, versus 22.78% at C32. This explains why calibration cannot be transported by merely freezing an intercept, but it does not excuse the below-random byte ordering: ROC AUC and value capture are rank metrics.

## Held-out ranking results

The first target is binary bundle reuse probability. The second directly regresses the constant-cost value density:

$$
V(b) = \frac{\text{future external key references attributed to } b}
{\text{resident blocks in } b}
$$

This corresponds to the research program's first cost label: equal reload cost per missing chunk. It does not claim that a reference equals exposed TTFT.

### Primary direction: train C32, freeze on C64

| Score target | Model | Request AUC | Byte-weighted AUC | Top-10% byte value capture | Top-25% byte value capture | Top-50% byte value capture |
|---|---|---:|---:|---:|---:|---:|
| Reuse probability | Constant | 0.500 | 0.500 | 8.92% | 25.00% | 45.51% |
| Reuse probability | Prompt only | 0.638 | 0.470 | 5.73% | 16.61% | 46.24% |
| Reuse probability | Request context | **0.674** | 0.472 | 6.84% | 19.69% | 48.42% |
| Reuse probability | Bundle context | 0.612 | 0.400 | 3.65% | 13.43% | 47.09% |
| References/block | Constant | 0.500 | 0.500 | 8.92% | 25.00% | 45.51% |
| References/block | Prompt only | 0.638 | 0.470 | 5.73% | 16.61% | 46.24% |
| References/block | Request context | **0.673** | 0.469 | 5.94% | 18.49% | 48.74% |
| References/block | Bundle context | 0.589 | 0.375 | 2.68% | 14.13% | 46.24% |

Figure 2 contrasts the metric that looks encouraging with the capacity-weighted metric that controls placement usefulness. Provenance: models trained on all C32 bundle examples and frozen on all C64 bundle examples.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — C32→C64 unweighted reuse signal does not rank valuable bytes","width":700,"height":300,"data":{"values":[{"model_metric":"Constant · request AUC","value":0.500,"model":"Constant"},{"model_metric":"Constant · byte AUC","value":0.500,"model":"Constant"},{"model_metric":"Prompt only · request AUC","value":0.638,"model":"Prompt only"},{"model_metric":"Prompt only · byte AUC","value":0.470,"model":"Prompt only"},{"model_metric":"Request context · request AUC","value":0.674,"model":"Request context"},{"model_metric":"Request context · byte AUC","value":0.472,"model":"Request context"},{"model_metric":"Bundle context · request AUC","value":0.612,"model":"Bundle context"},{"model_metric":"Bundle context · byte AUC","value":0.400,"model":"Bundle context"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"model_metric","type":"nominal","title":"Frozen model and metric","sort":null,"axis":{"labelAngle":-25}},"y":{"field":"value","type":"quantitative","title":"ROC AUC (fraction)","scale":{"zero":true,"domain":[0,0.75]}},"color":{"field":"model","type":"nominal","title":"Model","scale":{"scheme":"category10"}},"tooltip":[{"field":"model_metric","type":"nominal","title":"Model / metric"},{"field":"value","type":"quantitative","title":"AUC","format":".3f"}]}}
~~~

The request-context score recognizes some reused *bundles*, but the ordering is negatively correlated with valuable resident bytes. That is the wrong trade for a finite CPU tier.

### Reverse robustness check: train C64, freeze on C32

| Score target | Model | Request AUC | Byte-weighted AUC | Top-10% byte value capture | Top-25% byte value capture | Top-50% byte value capture |
|---|---|---:|---:|---:|---:|---:|
| Reuse probability | Request context | 0.577 | 0.608 | 13.89% | 33.46% | 65.92% |
| Reuse probability | Bundle context | 0.534 | 0.539 | 13.39% | 29.18% | 50.69% |
| References/block | Request context | 0.583 | **0.616** | 13.69% | 33.37% | 67.68% |
| References/block | Bundle context | 0.519 | 0.515 | 15.81% | 20.27% | 46.25% |

Figure 3 shows top-byte-budget value lift in both directions for the direct references-per-block target. A lift of 1.0 is the byte-proportional reference line. Provenance: bidirectional frozen evaluation; no per-test tuning.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3 — Expected-value ranking lift is asymmetric across pressure","width":720,"height":300,"data":{"values":[{"direction_budget":"C32→C64 · top 10%","lift":0.594,"direction":"C32→C64"},{"direction_budget":"C32→C64 · top 25%","lift":0.740,"direction":"C32→C64"},{"direction_budget":"C32→C64 · top 50%","lift":0.975,"direction":"C32→C64"},{"direction_budget":"C64→C32 · top 10%","lift":1.369,"direction":"C64→C32"},{"direction_budget":"C64→C32 · top 25%","lift":1.335,"direction":"C64→C32"},{"direction_budget":"C64→C32 · top 50%","lift":1.354,"direction":"C64→C32"}]},"layer":[{"mark":{"type":"bar"}},{"mark":{"type":"rule","color":"#555555","strokeDash":[5,4]},"encoding":{"y":{"datum":1.0}}}],"encoding":{"x":{"field":"direction_budget","type":"nominal","title":"Training→test direction and retained-byte budget","sort":null,"axis":{"labelAngle":-22}},"y":{"field":"lift","type":"quantitative","title":"Future-reference capture lift (× byte-proportional)","scale":{"zero":true}},"color":{"field":"direction","type":"nominal","title":"Frozen evaluation direction","scale":{"scheme":"category10"}},"tooltip":[{"field":"direction_budget","type":"nominal","title":"Evaluation"},{"field":"lift","type":"quantitative","title":"Lift","format":".3f"}]}}
~~~

The positive reverse result is evidence that the features are not pure noise. It is not sufficient to select a policy because the direction we would have used operationally—learn from C32, withstand C64 pressure—fails, and the coefficient magnitudes change materially across training conditions.

## Why the expected-value target did not rescue the score

The binary and direct-density scores induce almost the same held-out order. The currently available context predicts whether a source request belongs to a reusable region more readily than it predicts the magnitude and capacity cost of that reuse.

The additional completed-bundle features are not stable:

- their standardized coefficients change sign or magnitude across pressure conditions;
- bundle context reduces C32→C64 byte AUC from 0.469 to 0.375;
- bundle context also reduces reverse expected-value byte AUC from 0.616 to 0.515.

This is a falsification of the current **static feature set**, not of future-value placement. The equal-capacity next-use oracle remains strong. What failed is the attempt to approximate it with a timeless score built from prompt size, source external reuse, bundle size, runtime duration, and instantaneous pressure.

## Decision and next experiment

### Do not run this score through cache replay or implement it live

The policy gate was:

1. held-out byte-weighted ranking above random;
2. positive value capture at capacity-relevant byte budgets;
3. the same qualitative direction in C32→C64 and C64→C32.

The request-context and bundle-context scores fail gates 1–3.

### Next: age-conditioned reuse hazard at eviction time

The next offline experiment should test the most important missing dimension from the research program and the ATC'25-style workload-aware line: **value changes with age**.

At each recorded cache-pressure decision, estimate:

$$
P(\text{reuse in }[t,t+H] \mid \text{bundle survived to age }t,\ \text{context})
$$

Compare:

- aggregate empirical hazard;
- prompt-band-conditioned hazard;
- request-context-conditioned hazard;
- age plus prefix-position/value-density correction;
- static Experiment 7 score;
- LRU and clairvoyant next-use references.

Train all hazard tables on one execution and freeze them on the other. Evaluate pairwise arrival-versus-victim ranking first, then run a cache replay only if both directions show positive cost-weighted lift.

If age-conditioned ranking also fails, the trace does not contain the stable session/workflow identity the research program expects to carry future value. The next action would then be instrumentation—not a larger model: stable conversation/session ID, turn index, inter-turn time, active/suspended state, tool/DAG event, tenant/application class, and replica-routing identity.

## Reproduction and verification

~~~text
../vllm/.venv/bin/python -m tools.costar.run_value_ranking \
  /private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite \
  /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite

../vllm/.venv/bin/python -m pytest tests/tools/test_costar_*.py -q
ruff check tools/costar tests/tools/test_costar_*.py
git diff --check
~~~

Verification: 46 COSTAR tests passed; Ruff and diff checks passed. Nothing is committed.

## Source runs

- C32: [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- C64: [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow)

Related: [[06 - Experiment 6 C64 independent pressure validation]], [[../Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle]], and [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching]].