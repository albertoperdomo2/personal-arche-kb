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
workload: "largest authoritative C32 request target"
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
---

# COSTAR offline gate 1 — L0 single-request transfer feasibility

## Aim

Verify the transfer arithmetic and establish the minimum future-knowledge horizon for one real request under deliberately simple conditions: perfect request knowledge, real source-ready time, unlimited CPU staging, one constant-bandwidth link, and no competing traffic.

## Verdict — Conditionally valid

The model passes hand-solvable feasible, impossible-deadline, late-source, unavailable-source, and shared-key cases. The real-request result is a feasibility diagnostic, not an end-to-end oracle, because bandwidth is constant and other requests are absent.

## Results

The largest C32 authoritative target contains 7,741 unique chunks: 15.12 GiB at 2 MiB/chunk. Its source state was already ready. At 2.5 GiB/s, pure service time is 6.048 seconds.

| Reveal regime | Available horizon | Completion slack | Ready by first lookup? |
|---|---:|---:|---|
| Recorded HTTP admission | 0.014 s | -6.033 s | No |
| Fixed horizon | 0.5 s | -5.548 s | No |
| Fixed horizon | 5 s | -1.048 s | No |
| Fixed horizon | 10 s | +3.952 s | Yes |
| Fixed horizon | 30 s | +23.952 s | Yes |

Figure 1 shows completion slack; zero is the first-demand deadline. Source: exact C32 target size and source-ready state, with the explicitly supplied 2.5 GiB/s L0 service rate.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Largest-request L0 completion slack","width":620,"height":280,"data":{"values":[{"horizon":"Admission (0.014 s)","slack_s":-6.033},{"horizon":"0.5 s","slack_s":-5.548},{"horizon":"5 s","slack_s":-1.048},{"horizon":"10 s","slack_s":3.952},{"horizon":"30 s","slack_s":23.952}]},"layer":[{"mark":{"type":"bar"}},{"mark":{"type":"rule","color":"black","strokeDash":[4,4]},"encoding":{"y":{"datum":0}}}],"encoding":{"x":{"field":"horizon","type":"ordinal","title":"Perfect-information horizon"},"y":{"field":"slack_s","type":"quantitative","title":"Completion slack (s)","scale":{"zero":true}},"color":{"condition":{"test":"datum.slack_s >= 0","value":"#2ca02c"},"value":"#d62728"},"tooltip":[{"field":"horizon","type":"ordinal","title":"Horizon"},{"field":"slack_s","type":"quantitative","title":"Slack (s)","format":".3f"}]}}
~~~

## Interpretation and decision

For this large request, admission-time knowledge is physically too late even before contention and eviction are considered. A roughly six-second signal is the minimum under the assumed rate. This does not imply every request requires ten seconds, and it does not establish workload benefit.

The C32 service corpus contains only 10 single-read and 2 two-read-concurrency observations, so 2.5 GiB/s is an explicit conservative input rather than a fully calibrated stochastic model.

**Next gate:** schedule all requests together, deduplicate shared keys, and distinguish proactive movement from retaining native CPU arrivals.