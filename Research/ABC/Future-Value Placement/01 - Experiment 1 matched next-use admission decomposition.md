---
title: "COSTAR-KV Experiment 1 — matched next-use admission decomposition"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "COSTAR-KV future-value placement / Experiment 1"
status: "conditionally-valid"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
vllm_version: "v0.27.0-derived oracle instrumentation"
code_base_revision: "e50f7d36960980c0c89651ffd0ce281a9fb8a466 plus uncommitted tools/costar experiment code"
tensor_parallelism: 8
concurrency: 32
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
secondary_tier: "filesystem-backed node-local NVMe"
workload: "AgentX/Weka C32"
random_seed: "deterministic; no randomized policy"
cache_cleaning_state: "offline replay of recorded native state"
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
corpus_sha256: "788adedcbc9850a64443681fbd8acf36ea62cdce950eeac6311f730f34cce687"
---

# COSTAR-KV Experiment 1 — matched next-use admission decomposition

## Executive summary

This experiment asks whether the accepted finite-CPU result required the ability to reject mirrored arrivals, or whether better victim selection was sufficient while still admitting every arrival.

Four policies replayed the accepted C32 request targets and recorded CPU-ready arrival stream at the same 131,072-chunk capacity:

1. recorded physical residency;
2. always-admit LRU;
3. forced-admit clairvoyant next-use placement;
4. bypass-capable clairvoyant next-use placement.

**Measured result:** forced admission and bypass both retained every authoritative external key reference, made all 42 nonempty external request targets CPU-complete, and avoided all 12 native secondary reads / 36.440 seconds of measured device service. Bypass provided no additional movement benefit, but reduced persistent admissions by 177,891 and evictions by 177,891 relative to the forced-admit policy.

## Validity verdict — Conditionally valid diagnostic

The comparison is valid for the declared **common recorded-arrival, equal-size next-use replay semantics**. It directly falsifies the claim that bypass was necessary to obtain the prior 12-read result under those semantics.

It is not a global optimality or live-performance proof:

- the next-use policies use perfect future request knowledge;
- nearest-next-use is not globally optimal for all repeated-demand sequences when arrivals are exogenous and counterfactual demand fills are not simulated;
- all policies receive the recorded CPU-ready stream, including 48,444 block arrivals produced by native demand jobs;
- the replay does not enforce live ref-count/protected-block eligibility;
- bypass assumes transient secondary cascade, so rejecting persistent CPU residency does not remove secondary availability;
- measured device-service seconds are not necessarily exposed TTFT.

## Hypothesis

**H1:** A bypass-capable future-aware policy will materially outperform an otherwise matched forced-admit future-aware victim policy on native-read avoidance or request-level CPU completeness.

If supported, admission rejection must become a first-class vLLM policy operation. If contradicted, practical victim ranking should be the immediate research priority, while bypass remains a possible churn optimization.

## Main takeaways

- **Observation:** recorded residency made 30/42 nonempty external targets CPU-complete and initiated 12 native reads.
- **Observation:** always-admit LRU made 27/42 targets complete, yet happened to avoid 2/12 recorded native-read requests. Key-hit ratio alone does not determine which batched requests move data.
- **Observation:** forced-admit next-use placement achieved 42/42 complete targets, 157,283/157,283 external key hits, and 12/12 native reads avoided.
- **Observation:** bypass-capable next-use placement produced exactly the same request and movement outcome.
- **Observation:** bypass reduced admissions from 368,265 to 190,374, evictions from 237,193 to 59,302, and admission-plus-eviction churn from 605,458 to 249,676.
- **Inference:** within this replay, future-aware victim selection carries 100% of the measured request/service opportunity. Bypass carries 0% additional movement benefit but 58.8% of the matched policy's churn reduction.
- **Conclusion:** redirect the next experiment toward practical future-value victim selection. Evaluate explicit admission filters afterward or alongside it for overhead/churn, not because C32 establishes that rejection is required for read avoidance.

## Headline results

| Policy | Complete external targets | External key hits | Native reads avoided | Service avoided | Admissions | Rejections | Evictions |
|---|---:|---:|---:|---:|---:|---:|---:|
| Recorded physical | 30/42 (71.43%) | 108,839/157,283 (69.20%) | 0/12 | 0.000 s | 416,709 | 0 | 285,637 |
| Always-admit LRU | 27/42 (64.29%) | 99,955/157,283 (63.55%) | 2/12 | 5.620 s | 405,217 | 0 | 274,145 |
| Forced-admit next-use | 42/42 (100%) | 157,283/157,283 (100%) | 12/12 | 36.440 s | 368,265 | 0 | 237,193 |
| Bypass-capable next-use | 42/42 (100%) | 157,283/157,283 (100%) | 12/12 | 36.440 s | 190,374 | 177,891 | 59,302 |

The 901-request denominator is not the useful headline because 859 requests have empty external targets. Across all requests, the corresponding CPU-complete counts are 889, 886, 901, and 901.

## Evidence

Figure 1 compares the movement-specific outcome. Source: exact full-resolution C32 external targets and measured native job service; no aggregation beyond the policy-level experiment result.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Native secondary-read service avoided by placement policy","width":650,"height":300,"data":{"values":[{"policy":"Recorded physical","service_percent":0.0},{"policy":"Always-admit LRU","service_percent":15.424},{"policy":"Forced-admit next-use","service_percent":100.0},{"policy":"Bypass-capable next-use","service_percent":100.0}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy","type":"nominal","title":"CPU placement policy","axis":{"labelAngle":-20}},"y":{"field":"service_percent","type":"quantitative","title":"Measured native-read service avoided (%)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"scheme":"category10"}},"tooltip":[{"field":"policy","type":"nominal","title":"Policy"},{"field":"service_percent","type":"quantitative","title":"Service avoided (%)","format":".3f"}]}}
~~~

Both future-aware policies recover the complete movement result. The absence of a gap between forced admission and bypass contradicts H1 for this metric and trace.

Figure 2 shows the action cost required to obtain that outcome. Source: all 416,709 native CPU-ready generations at their exact recorded ordering; categories are policy/action totals.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — CPU placement actions under matched replay","width":720,"height":320,"data":{"values":[{"policy_action":"Recorded · admissions","events":416709,"policy":"Recorded"},{"policy_action":"Recorded · evictions","events":285637,"policy":"Recorded"},{"policy_action":"LRU · admissions","events":405217,"policy":"Always-admit LRU"},{"policy_action":"LRU · evictions","events":274145,"policy":"Always-admit LRU"},{"policy_action":"Forced · admissions","events":368265,"policy":"Forced-admit next-use"},{"policy_action":"Forced · evictions","events":237193,"policy":"Forced-admit next-use"},{"policy_action":"Bypass · admissions","events":190374,"policy":"Bypass-capable next-use"},{"policy_action":"Bypass · rejections","events":177891,"policy":"Bypass-capable next-use"},{"policy_action":"Bypass · evictions","events":59302,"policy":"Bypass-capable next-use"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy_action","type":"nominal","title":"Policy action","axis":{"labelAngle":-28}},"y":{"field":"events","type":"quantitative","title":"Placement events (count)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"scheme":"category10"}},"tooltip":[{"field":"policy_action","type":"nominal","title":"Action"},{"field":"events","type":"quantitative","title":"Events","format":","}]}}
~~~

The bypass policy exchanges 177,891 forced admissions and their 177,891 victim evictions for 177,891 rejections. Evictions fall 75.0%, admissions fall 48.3%, and combined admission-plus-eviction churn falls 58.8% without changing request completeness.

## Arrival provenance and counterfactual limitation

The normalized corpus contains:

| CPU-ready provenance | Block arrivals |
|---|---:|
| Ordinary GPU-mirror/persistence path | 368,265 |
| Native secondary-to-CPU demand promotion | 48,444 |
| Unclassified | 0 |

All 48,444 demand-generated arrivals join exactly to the 12 recorded native demand jobs. They become duplicate arrivals for both next-use policies because those policies already retained the keys and therefore claim the corresponding native reads were avoidable.

This preserves comparability with the preceding finite-retention diagnostic, but it means the replay is not a fully endogenous alternate execution. A later simulator should synthesize policy-induced demand fills and suppress baseline fills whose reads were avoided.

## Important methodological correction

The initial experiment design proposed proving that bypass-capable nearest-next-use placement was the global offline optimum on toy traces. Exhaustive validation falsified that claim.

Counterexample with capacity one:

1. keys 0, 1, and 2 arrive before demand;
2. future demand is 0, 1, 1, 1;
3. greedy nearest-next-use retains key 0 and obtains one hit;
4. retaining key 1 obtains three hits.

Therefore this report uses **matched clairvoyant next-use policy**, not “optimal Belady oracle,” in its conclusion. The matched comparison still isolates the effect of allowing the arrival to be rejected under the same scoring rule. It does not establish the globally optimal request- or service-cost placement.

## Missing-key attribution

At the authoritative first-demand snapshots:

| Policy | Missing external references | Attributed to rejection | Attributed to eviction | Never observed |
|---|---:|---:|---:|---:|
| Recorded physical | 48,444 | 0 | 48,444 | 0 |
| Always-admit LRU | 57,328 | 0 | 57,328 | 0 |
| Forced-admit next-use | 0 | 0 | 0 | 0 |
| Bypass-capable next-use | 0 | 0 | 0 | 0 |

No bypassed arrival was later required under this matched next-use trajectory. That is a perfect-future result, not evidence that an online rejection policy will have zero false negatives.

## Verification

Implementation:

- `tools/costar/oracle_decomposition.py`
- `tools/costar/run_oracle_decomposition.py`
- provenance extension in `tools/costar/finite_retention.py`
- `tests/tools/test_costar_oracle_decomposition.py`

Focused verification:

~~~text
../vllm/.venv/bin/python -m pytest   tests/tools/test_costar_oracle_decomposition.py   tests/tools/test_costar_finite_retention.py -q

8 passed
~~~

Ruff and `git diff --check` pass for the touched COSTAR files.

Reproduction command:

~~~text
../vllm/.venv/bin/python -m tools.costar.run_oracle_decomposition   /private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite
~~~

Source MLflow run: [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow).

## Decision

H1 is **contradicted for read avoidance, external request completeness, and measured service under the declared C32 replay**.

The prior wording “the oracle's main action is refusal” remains true as an action-count description of the bypass policy, but not as a causal description of why the 12 native reads disappear. A matched forced-admit future-aware victim policy produces the same movement outcome.

Admission bypass remains potentially valuable because it sharply reduces churn and may avoid unnecessary data movement or metadata work in a live transient-cascade design. Those costs are not modeled here.

## Next experiment

Run a practical **victim-policy benchmark** at exact capacity before building a new live vLLM admission path. Compare at least:

- always-admit LRU;
- vLLM ARC;
- aged LFU/LRFU;
- 2Q/LIRS-style reuse-distance ranking;
- an ATC'25-style age/category/prefix-offset baseline where the trace exposes the required category.

Primary outcomes remain 42-request external completeness, 12-read / 36.440-second avoidance, key coverage, victim regret, metadata, and replay cost. Add S3-FIFO/W-TinyLFU as a parallel churn/admission axis, not as the assumed source of movement benefit.

Before live enforcement, extend the simulator with policy-induced demand fills, protected-block eligibility, and a selected secondary-cascade semantic.