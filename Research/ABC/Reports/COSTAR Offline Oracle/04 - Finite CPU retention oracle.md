---
title: "COSTAR offline gate 3 — finite-CPU retention oracle"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "ABC / COSTAR-KV offline gate 3"
status: "valid-diagnostic"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
tensor_parallelism: 8
concurrency: 32
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
policies: ["recorded residency", "always-admit LRU", "clairvoyant next-use admission/eviction"]
secondary_tier: "filesystem-backed node-local NVMe"
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
---

# COSTAR offline gate 3 — finite-CPU retention oracle

## Aim

Test whether better CPU admission and eviction can avoid native secondary reads at the configured 131,072-slot capacity, before adding any proactive movement. All policies receive the same recorded CPU-ready arrivals. The comparison is recorded physical residency versus an always-admit LRU replay and a clairvoyant next-use policy that may reject low-value arrivals.

## Verdict — Valid diagnostic

For all 898 non-aborted requests, reconstructed CPU target completeness agrees exactly with whether a native secondary→CPU read occurred. This validates the movement-specific oracle. Clairvoyant next-use eliminates all 12 native reads at the same physical capacity.

This is not an end-to-end performance prediction. The policy has perfect future knowledge, uses baseline arrivals as free counterfactual inputs, and does not model the residual asynchronous secondary existence check.

## Results

| Policy | CPU-complete requests | External key hits | Admissions | Rejected arrivals | Evictions |
|---|---:|---:|---:|---:|---:|
| Recorded physical residency | 889/901, 98.67% | 108,839/157,283, 69.20% | 416,709 | 0 | 285,637 |
| Simulated always-admit LRU | 886/901, 98.34% | 99,955/157,283, 63.55% | 405,217 | 0 | 274,145 |
| Clairvoyant next-use | 901/901, 100% | 157,283/157,283, 100% | 190,374 | 177,891 | 59,302 |

Movement outcome:

- Native secondary→CPU read requests: **12**.
- Clairvoyant-avoidable reads: **12/12**.
- Measured native read service time: **36.440 seconds**.
- Clairvoyant-avoidable measured service time: **36.440 seconds**.
- Ground-truth movement mismatches: **0/898 non-aborted requests**.

Figure 1 compares request-level CPU completeness and key-level hit coverage. Source: exact full-resolution C32 CPU generation and first-lookup target replay.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Finite-CPU retention outcomes","width":680,"height":300,"data":{"values":[{"policy_metric":"Recorded · CPU-complete requests","percent":98.668,"policy":"Recorded"},{"policy_metric":"Recorded · external key hits","percent":69.199,"policy":"Recorded"},{"policy_metric":"LRU · CPU-complete requests","percent":98.335,"policy":"Always-admit LRU"},{"policy_metric":"LRU · external key hits","percent":63.551,"policy":"Always-admit LRU"},{"policy_metric":"Clairvoyant · CPU-complete requests","percent":100.0,"policy":"Clairvoyant next-use"},{"policy_metric":"Clairvoyant · external key hits","percent":100.0,"policy":"Clairvoyant next-use"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy_metric","type":"nominal","title":"Policy and outcome","axis":{"labelAngle":-25}},"y":{"field":"percent","type":"quantitative","title":"Coverage (%)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"scheme":"category10"}},"tooltip":[{"field":"policy_metric","type":"nominal","title":"Outcome"},{"field":"percent","type":"quantitative","title":"Coverage (%)","format":".3f"}]}}
~~~

The request-complete difference looks numerically small because only 12 requests moved data, but those misses trigger large batched reads. Key-hit rate alone is also misleading: recorded residency hits 69.2% of externally reused keys yet still causes 12 multi-GiB movements.

Figure 2 shows how the clairvoyant policy changes cache churn. Source: recorded generation events and equal-capacity policy replay.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — Finite-CPU cache actions","width":680,"height":300,"data":{"values":[{"policy_action":"Recorded · admissions","events":416709,"policy":"Recorded"},{"policy_action":"Recorded · evictions","events":285637,"policy":"Recorded"},{"policy_action":"LRU · admissions","events":405217,"policy":"Always-admit LRU"},{"policy_action":"LRU · evictions","events":274145,"policy":"Always-admit LRU"},{"policy_action":"Clairvoyant · admissions","events":190374,"policy":"Clairvoyant next-use"},{"policy_action":"Clairvoyant · rejected","events":177891,"policy":"Clairvoyant next-use"},{"policy_action":"Clairvoyant · evictions","events":59302,"policy":"Clairvoyant next-use"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy_action","type":"nominal","title":"Policy action","axis":{"labelAngle":-25}},"y":{"field":"events","type":"quantitative","title":"Cache events (count)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"scheme":"category10"}},"tooltip":[{"field":"policy_action","type":"nominal","title":"Action"},{"field":"events","type":"quantitative","title":"Events","format":","}]}}
~~~

The oracle's main action is refusal: it prevents low-future-value mirrored arrivals from displacing nearer-use state. This cuts eviction churn by 79.2% relative to recorded residency.

## Critical semantic correction

**matched_key_counts** is the total cached prefix, including GPU-local chunks. It is not the external CPU target. The authoritative external target is the trailing **matched_tokens / tokens_per_chunk** segment ending at each group's matched count.

After correction:

- External references: **157,283**, not 3,247,280.
- Unique external target keys: **116,409**, not 289,266.
- Mean external target: **174.56 chunks across all requests**; median and p95 are zero because only 42/901 requests reuse external tokens.
- A first lookup can defer solely for an asynchronous terminal filesystem existence probe. Therefore 891 deferred first attempts do not mean 891 CPU misses. Only 12 requests initiate native movement.

The earlier total-prefix interpretation was rejected and the preceding offline reports were corrected.

## Interpretation and decision

This experiment establishes meaningful oracle headroom for **admission/retention policy**: the configured CPU capacity is sufficient to retain all 116,409 unique externally reused keys, but ordinary mirroring introduces 368,265 unique keys overall and causes valuable reuse state to be displaced. Perfect next-use admission avoids all measured native reloads without increasing capacity.

It does not establish that an online predictor can identify those keys, that all 36.44 seconds are exposed TTFT, or that proactive NVMe reads are needed. The next experiment should measure information value: how early and accurately must a practical signal rank future external reuse to recover a useful fraction of the clairvoyant retention result?

Recommended baselines are category/session-aware admission, reuse-frequency/recency ranking, and predicted next-use ranking. Evaluate each by recovered oracle fraction, avoided native read service, false-retention capacity cost, and robustness—not by first-attempt deferred count.