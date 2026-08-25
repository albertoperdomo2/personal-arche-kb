---
title: "COSTAR offline gate 2 — global relaxed-retention diagnosis"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "ABC / COSTAR-KV offline gate 2"
status: "conditionally-valid"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
tensor_parallelism: 8
concurrency: 32
cpu_blocks: 131072
cpu_policy_in_model: "retain all externally reused keys; no eviction"
secondary_tier: "filesystem-backed node-local NVMe"
service_model: "global EDF at constant 2.5 GiB/s"
horizons_seconds: [0.5, 1, 2, 5, 10, 30]
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
---

# COSTAR offline gate 2 — global relaxed-retention diagnosis

## Aim

Determine whether a global perfect-horizon scheduler needs proactive secondary reads after reconstructing corrected external targets, shared keys, real source-ready times, and recorded native CPU-ready arrivals, while allowing ideal retention of externally reused state.

## Verdict — Conditionally valid diagnostic

The corrected replay finds that retention dominates movement. Every externally reused key appeared in CPU natively before its relevant deadline. Retaining those keys avoids all 12 native read jobs without scheduling a proactive read.

This is not yet a deployable policy: it uses perfect future knowledge to retain externally reused keys while ignoring 251,856 other unique native CPU arrivals.

## Results

| Quantity | Result |
|---|---:|
| External request→key references | 157,283 |
| Unique external target keys | 116,409 |
| Shared external references | 40,874 |
| Requests with nonzero external target | 42/901 |
| Real CPU capacity | 131,072 keys |
| Ideal externally reused footprint | 116,409 keys / 88.81% capacity |
| Native secondary→CPU read requests | 12 |
| Avoidable under ideal retention | 12/12; 36.440 s measured service |
| Proactive secondary→CPU reads | 0 |
| Horizon sensitivity, 0.5–30 s | None |

Figure 1 compares the corrected reusable footprint and movement result. Source: normalized full C32 corpus; every tested horizon produced the same result.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Corrected global relaxed-retention result","width":580,"height":280,"data":{"values":[{"measure":"External target / CPU capacity","percent":88.813},{"measure":"Native read jobs avoided","percent":100.0},{"measure":"Native read service avoided","percent":100.0}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"measure","type":"nominal","title":"Relaxed-retention measure","axis":{"labelAngle":-20}},"y":{"field":"percent","type":"quantitative","title":"Fraction (%)","scale":{"zero":true}},"color":{"field":"measure","type":"nominal","title":"Measure","scale":{"scheme":"category10"}},"tooltip":[{"field":"measure","type":"nominal","title":"Measure"},{"field":"percent","type":"quantitative","title":"Percent","format":".3f"}]}}
~~~

## Critical correction

The initial version incorrectly treated total matched group counts as external demand and reported 3,247,280 references, 289,266 unique keys, and a 2.21× capacity requirement. That result is rejected.

In vLLM, group matched counts include GPU-local chunks. The external target is the trailing matched-token contribution. Correct extraction yields 157,283 references and 116,409 unique keys—small enough to fit within the configured CPU tier if unrelated arrivals can be rejected intelligently.

Likewise, a first lookup can defer for the asynchronous terminal filesystem probe even when its external target is already CPU-complete. This report therefore claims avoided native movement, not elimination of all 891 first-attempt deferrals or all lookup→ready time.

## Interpretation and decision

C32 does contain physically recoverable value, but the immediate mechanism is admission/retention rather than earlier NVMe movement. Ordinary mirroring introduces 368,265 unique CPU keys overall, while only 116,409 are reused externally. The next gate must enforce capacity across all arrivals and determine whether future-aware rejection and victim choice can preserve the reusable subset.

**Next gate:** compare recorded residency and always-admit LRU against a clairvoyant next-use admission/eviction policy at exactly 131,072 slots.