---
title: "COSTAR-KV Experiment 4 — contextual request-bundle placement"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "COSTAR-KV future-value placement / Experiment 4"
status: "conditionally-valid-negative-result"
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
cache_cleaning_state: "offline replay of recorded native state"
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
corpus_sha256: "788adedcbc9850a64443681fbd8acf36ea62cdce950eeac6311f730f34cce687"
---

# COSTAR-KV Experiment 4 — contextual request-bundle placement

## Executive summary

Experiment 3 showed that exact-key history is unavailable for every valuable ordinary arrival, while future demand is strongly grouped by the request that created the KV. Experiment 4 tests the smallest corresponding policies:

- bundle FIFO and bundle LRU, with source request ID as the placement identity;
- source-external-reuse priority, forced-admit and bypass;
- post-hoc prompt-band priority with bypass;
- combined source-reuse and prompt-band priority with bypass.

All policies replay the same recorded arrivals at 131,072 slots. None beats recorded physical placement. The most aggressive contextual policy reaches a 67.53% key-hit ratio and eliminates 8/12 recorded reads, but completes only 21/42 external requests and creates 17 new misses: **a net regression of 9 requests**.

## Validity verdict — Conditionally valid negative result

The result is valid under common recorded-arrival semantics and exact equal capacity. It is not an endogenous simulator: newly created misses do not generate alternate timed demand fills, and their service cost is unknown.

The prompt boundaries—44,234, 62,413, and 90,398 tokens—were selected after inspecting this same C32 trace. That policy is hypothesis generation, not generalization evidence. Its failure in-sample is nevertheless sufficient to reject it as currently defined.

## Headline results

| Placement | Complete external targets | Key-hit rate | Recorded reads avoided | Gross service avoided | New misses | Net miss delta |
|---|---:|---:|---:|---:|---:|---:|
| Recorded physical | 30/42 (71.43%) | 69.20% | 0/12 | 0.000 s | 0 | 0 |
| Bundle FIFO | 27/42 (64.29%) | 61.88% | 3/12 | 8.035 s | 6 | **+3** |
| Bundle LRU | 26/42 (61.90%) | 61.02% | 2/12 | 5.620 s | 6 | **+4** |
| Source reuse · forced | 26/42 (61.90%) | 62.31% | 2/12 | 5.620 s | 6 | **+4** |
| Source reuse · bypass | 26/42 (61.90%) | 62.31% | 2/12 | 5.620 s | 6 | **+4** |
| Prompt band · bypass | 21/42 (50.00%) | 64.46% | 8/12 | 21.774 s | 17 | **+9** |
| Combined context · bypass | 21/42 (50.00%) | 67.53% | 8/12 | 21.774 s | 17 | **+9** |
| Clairvoyant next-use reference | 42/42 (100%) | 100% | 12/12 | 36.440 s | 0 | -12 |

Positive net miss delta is harmful. Gross service avoided cannot be netted against newly created reads because those counterfactual service times were not recorded.

## Main takeaways

- **Bundle identity alone is insufficient.** Bundle FIFO is identical to block FIFO on headline outcomes; bundle LRU is worse.
- **The sparse source-reuse signal does not change the trajectory.** Forced and bypass variants are identical, and the bypass variant rejects zero arrivals because a lower-scored incoming bundle never encounters a fully higher-scored resident population.
- **Prompt bands aggressively substitute victims rather than improve placement.** They recover 8 recorded misses but create 17 new ones.
- **Block-hit optimization remains misleading.** Combined context has the strongest practical key-hit result, 67.53%, while tying for the worst request completeness, 50%.
- **The current mechanisms still evict bundles one block at a time.** That preserves bundle identity in metadata but not whole-request completeness. Whole-bundle eviction is the next discriminating mechanism.

## Evidence

Figure 1 shows the primary request-level outcome. Provenance: exact C32 request targets under the common-arrival replay.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Complete external requests under contextual placement","width":720,"height":300,"data":{"values":[{"policy":"Recorded","complete_percent":71.43,"class":"Reference"},{"policy":"Bundle FIFO","complete_percent":64.29,"class":"Practical"},{"policy":"Bundle LRU","complete_percent":61.90,"class":"Practical"},{"policy":"Reuse forced","complete_percent":61.90,"class":"Practical"},{"policy":"Reuse bypass","complete_percent":61.90,"class":"Practical"},{"policy":"Prompt bypass","complete_percent":50.00,"class":"Practical"},{"policy":"Combined bypass","complete_percent":50.00,"class":"Practical"},{"policy":"Next-use oracle","complete_percent":100.0,"class":"Clairvoyant"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy","type":"nominal","title":"Placement policy","sort":null,"axis":{"labelAngle":-24}},"y":{"field":"complete_percent","type":"quantitative","title":"External targets CPU-complete (%)","scale":{"zero":true,"domain":[0,100]}},"color":{"field":"class","type":"nominal","title":"Policy class","scale":{"domain":["Reference","Practical","Clairvoyant"],"range":["#6b7280","#4c78a8","#e45756"]}},"tooltip":[{"field":"policy","type":"nominal","title":"Policy"},{"field":"complete_percent","type":"quantitative","title":"Complete targets (%)","format":".2f"}]}}
~~~

No contextual policy reaches the recorded reference.

Figure 2 places block-hit rate next to request completeness. Provenance: the same 157,283 external key references and 42 request targets.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — Block hits do not imply complete request targets","width":720,"height":300,"data":{"values":[{"policy_metric":"Recorded · key hits","percent":69.20,"policy":"Recorded"},{"policy_metric":"Recorded · complete requests","percent":71.43,"policy":"Recorded"},{"policy_metric":"Bundle FIFO · key hits","percent":61.88,"policy":"Bundle FIFO"},{"policy_metric":"Bundle FIFO · complete requests","percent":64.29,"policy":"Bundle FIFO"},{"policy_metric":"Combined · key hits","percent":67.53,"policy":"Combined context"},{"policy_metric":"Combined · complete requests","percent":50.00,"policy":"Combined context"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy_metric","type":"nominal","title":"Policy and outcome","sort":null,"axis":{"labelAngle":-25}},"y":{"field":"percent","type":"quantitative","title":"Outcome (%)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"scheme":"category10"}},"tooltip":[{"field":"policy_metric","type":"nominal","title":"Outcome"},{"field":"percent","type":"quantitative","title":"Percent","format":".2f"}]}}
~~~

Combined context nearly restores the recorded block-hit ratio while destroying request completeness. This is direct evidence for an all-required-block objective.

Figure 3 shows substitution explicitly. Provenance: baseline reads eliminated and newly incomplete baseline-hit requests.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3 — Recorded misses recovered versus new misses created","width":700,"height":290,"data":{"values":[{"policy_outcome":"Bundle FIFO · recovered","requests":3,"policy":"Bundle FIFO"},{"policy_outcome":"Bundle FIFO · created","requests":6,"policy":"Bundle FIFO"},{"policy_outcome":"Reuse priority · recovered","requests":2,"policy":"Reuse priority"},{"policy_outcome":"Reuse priority · created","requests":6,"policy":"Reuse priority"},{"policy_outcome":"Prompt band · recovered","requests":8,"policy":"Prompt band"},{"policy_outcome":"Prompt band · created","requests":17,"policy":"Prompt band"},{"policy_outcome":"Combined · recovered","requests":8,"policy":"Combined context"},{"policy_outcome":"Combined · created","requests":17,"policy":"Combined context"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy_outcome","type":"nominal","title":"Policy and request outcome","sort":null,"axis":{"labelAngle":-28}},"y":{"field":"requests","type":"quantitative","title":"External requests (count)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"scheme":"category10"}},"tooltip":[{"field":"policy_outcome","type":"nominal","title":"Outcome"},{"field":"requests","type":"quantitative","title":"Requests"}]}}
~~~

The prompt-derived policies are not close calls: every recovered baseline miss is accompanied by more than two newly created misses.

## Churn and overhead

| Policy | Admissions | Rejections | Evictions | Median Python ns/event |
|---|---:|---:|---:|---:|
| Bundle FIFO | 404,706 | 0 | 273,634 | 470 |
| Bundle LRU | 407,444 | 0 | 276,372 | 451 |
| Source reuse · forced | 407,750 | 0 | 276,678 | 1,566 |
| Source reuse · bypass | 407,750 | 0 | 276,678 | 1,387 |
| Prompt band · bypass | 296,182 | 85,617 | 165,110 | 1,042 |
| Combined context · bypass | 298,242 | 88,969 | 167,170 | 1,147 |

The prompt policies reduce churn materially, but that saving is purchased with worse request placement. Python replay timing is relative complexity evidence, not live scheduler latency.

## Decision

Reject the six policies as live candidates in their current form.

Retain these lessons:

1. group identity is necessary metadata but not sufficient policy;
2. source external reuse is a sparse high-precision hint, not a complete score;
3. C32-derived prompt bands must not be promoted;
4. victim selection must optimize complete working sets and reload cost, not isolated key coverage.

## Next experiment

Test **whole-bundle eviction** as a mechanism diagnostic. When capacity binds, evict a complete low-priority source bundle rather than one block from many bundles. Measure:

- complete external targets and net new misses;
- average and tail unused CPU capacity caused by coarse eviction;
- whole-target preservation versus fragmentation;
- churn and victim regret.

This is intentionally separate because it trades utilization for completeness. If it cannot beat blockwise FIFO on C32, bundle identity alone is exhausted and the research requires richer session/workflow signals or a request-level cost oracle.

## Reproduction

- `tools/costar/contextual_policy_benchmark.py`
- `tools/costar/run_contextual_policy_benchmark.py`
- `tests/tools/test_costar_contextual_policy_benchmark.py`

~~~text
../vllm/.venv/bin/python -m tools.costar.run_contextual_policy_benchmark   /private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite
~~~

Focused verification: 7 tests passed; Ruff passed.

Source corpus: [MLflow C32 run f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow).