---
title: "COSTAR offline gate 1 — L0 single-request transfer feasibility"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "ABC / COSTAR-KV offline gate 1"
status: "conditionally-valid"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
tensor_parallelism: 8
concurrency: 32
cpu_capacity: "unlimited staging relaxation"
secondary_tier: "filesystem-backed node-local NVMe"
service_model: "constant 2.5 GiB/s; zero setup; no competing traffic"
workload: "largest authoritative external C32 request target"
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
---

# COSTAR offline gate 1 — L0 single-request transfer feasibility

## Aim

Verify transfer arithmetic and establish the minimum future-knowledge horizon for one real external target under deliberately simple conditions: perfect knowledge, real source-ready time, unlimited CPU staging, one constant-bandwidth link, and no competing traffic.

## Verdict — Conditionally valid

The model passes hand-solvable feasible, impossible-deadline, late-source, unavailable-source, and shared-key cases. The real-request result is only a feasibility diagnostic because bandwidth is constant and other requests are absent.

A post-run semantic audit corrected the target definition: total matched group counts include GPU-local chunks. L0 now selects only the trailing external portion defined by matched tokens.

## Results

The largest authoritative external target contains 7,311 unique chunks: 14.28 GiB at 2 MiB/chunk. Its source state was already ready. At 2.5 GiB/s, pure service time is 5.712 seconds.

| Reveal regime | Available horizon | Completion slack | Ready by first lookup? |
|---|---:|---:|---|
| Recorded HTTP admission | 0.005 s | -5.707 s | No |
| Fixed horizon | 0.5 s | -5.212 s | No |
| Fixed horizon | 5 s | -0.712 s | No |
| Fixed horizon | 10 s | +4.288 s | Yes |
| Fixed horizon | 30 s | +24.288 s | Yes |

Figure 1 shows completion slack; zero is the first-demand deadline. Source: exact corrected C32 external target and source-ready state, with the explicitly supplied 2.5 GiB/s L0 service rate.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Largest external-target L0 completion slack","width":620,"height":280,"data":{"values":[{"horizon":"Admission (0.005 s)","slack_s":-5.707},{"horizon":"0.5 s","slack_s":-5.212},{"horizon":"5 s","slack_s":-0.712},{"horizon":"10 s","slack_s":4.288},{"horizon":"30 s","slack_s":24.288}]},"layer":[{"mark":{"type":"bar"}},{"mark":{"type":"rule","color":"black","strokeDash":[4,4]},"encoding":{"y":{"datum":0}}}],"encoding":{"x":{"field":"horizon","type":"ordinal","title":"Perfect-information horizon"},"y":{"field":"slack_s","type":"quantitative","title":"Completion slack (s)","scale":{"zero":true}},"color":{"condition":{"test":"datum.slack_s >= 0","value":"#2ca02c"},"value":"#d62728"},"tooltip":[{"field":"horizon","type":"ordinal","title":"Horizon"},{"field":"slack_s","type":"quantitative","title":"Slack (s)","format":".3f"}]}}
~~~

## Interpretation and decision

For the largest truly external request, admission-time knowledge is physically too late even before contention and eviction. Roughly 5.7 seconds is the minimum under the assumed rate. This does not imply every request requires that horizon: only 42/901 C32 requests have any externally matched tokens.

The service corpus contains only 10 single-read and 2 two-read-concurrency observations, so 2.5 GiB/s is an explicit conservative input rather than a fully calibrated stochastic model.

**Next gate:** replay all corrected external targets, native CPU arrivals, and shared keys before deciding whether movement or retention dominates.