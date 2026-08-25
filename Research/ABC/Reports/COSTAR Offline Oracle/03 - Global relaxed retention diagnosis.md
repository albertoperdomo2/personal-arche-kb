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
cpu_policy_in_model: "infinite retention; no eviction"
secondary_tier: "filesystem-backed node-local NVMe"
service_model: "global EDF at constant 2.5 GiB/s"
horizons_seconds: [0.5, 1, 2, 5, 10, 30]
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
---

# COSTAR offline gate 2 — global relaxed-retention diagnosis

## Aim

Determine whether a global, perfect-horizon scheduler exposes a bandwidth/prefetch opportunity when shared keys, real source-ready times, and recorded native CPU-ready arrivals are reconstructed, while deliberately relaxing CPU eviction.

## Verdict — Conditionally valid diagnostic; not physically deployable

The corrected replay finds a large relaxed opportunity, but it is entirely retention-driven. Native execution already brings the required shared keys into CPU. Keeping them indefinitely eliminates almost all first-demand deferrals without performing a proactive secondary read, but requires 2.21× real CPU capacity.

## Results

| Quantity | Result |
|---|---:|
| Request→key references | 3,247,280 |
| Unique target keys | 289,266 |
| Shared references | 2,958,014 |
| Real CPU capacity | 131,072 keys |
| Relaxed retained footprint | 289,266 keys / 2.207× capacity |
| Baseline deferred requests | 891 / 901 |
| Ready under infinite retention | 898 / 901 |
| Prevented deferrals | 889 / 891 |
| Inferred recoverable lookup→ready stall | 346.81 / 410.88 s, 84.41% |
| Proactive secondary→CPU reads | 0 keys / 0 bytes |
| Horizon sensitivity, 0.5–30 s | None |

Figure 1 compares the apparent benefit with the physical footprint required. Source: normalized full C32 corpus; all horizons produced the same corrected result.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Relaxed benefit versus required CPU footprint","width":580,"height":280,"data":{"values":[{"measure":"Deferred requests prevented","percent":99.776},{"measure":"Lookup→ready stall recovered","percent":84.407},{"measure":"Retained / physical CPU capacity","percent":220.692}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"measure","type":"nominal","title":"Relaxed-oracle measure","axis":{"labelAngle":-20}},"y":{"field":"percent","type":"quantitative","title":"Fraction (%)","scale":{"zero":true}},"color":{"field":"measure","type":"nominal","title":"Measure","scale":{"scheme":"category10"}},"tooltip":[{"field":"measure","type":"nominal","title":"Measure"},{"field":"percent","type":"quantitative","title":"Percent","format":".2f"}]}}
~~~

The 220.69% capacity bar is the critical validity constraint: the attractive readiness outcome cannot be realized by the configured tier without choosing victims.

## Interpretation and decision

The main opportunity in this C32 trace is not “read the secondary tier earlier.” It is “retain the correct heavily shared keys after native mirroring or reactive loading.” The fixed-native-read reservation variant is identical because the relaxed solution schedules no proactive reads.

An earlier intermediate replay incorrectly treated every target key as requiring a new read and produced horizon-dependent readiness. That interpretation was rejected. Once recorded native CPU-ready arrivals are included, horizon disappears and retention dominates.

This is evidence for placement/eviction co-design, not proof that a practical policy improves TTFT. The inferred 346.81 stall-seconds assumes that eliminating connector deferral removes the recorded lookup→ready delay; a finite replay must test that counterfactual more carefully.

**Next gate:** enforce 131,072 physical slots, compare native/LRU retention against clairvoyant next-use victim selection, charge reload consequences, and add proactive reads only when retention cannot satisfy a deadline.