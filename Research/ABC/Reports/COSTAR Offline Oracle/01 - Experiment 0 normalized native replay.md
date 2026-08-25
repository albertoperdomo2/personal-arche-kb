---
title: "COSTAR offline gate 0 — normalized native replay"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "ABC / COSTAR-KV offline gate 0"
status: "valid"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
vllm_version: "0.27.0"
tensor_parallelism: 8
concurrency: 32
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
secondary_tier: "filesystem-backed node-local NVMe"
workload: "AgentX Weka; semianalysisai/cc-traces-weka-062126"
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
oracle_trace_schema: 1
---

# COSTAR offline gate 0 — normalized native replay

## Aim

Prove that the accepted C32 oracle trace can be transformed into a queryable representation that reproduces native request, transfer, source-readiness, and physical CPU-occupancy invariants before any counterfactual policy is simulated.

## Verdict — Valid

The complete 2,241,218-event trace was streamed into a 2.3 GB normalized SQLite database. All 901 request lifecycles and 1,435 transfer lifecycles closed, reconstructed occupancy never exceeded capacity, and every replay consistency check passed.

## Results

| Check | Result |
|---|---:|
| Requests arrived / first lookup / ready / terminal | 901 / 901 / 901 / 901 |
| First attempt deferred / resolved | 891 / 10 |
| Transfer jobs / incomplete jobs | 1,435 / 0 |
| Peak / configured CPU occupancy | 131,072 / 131,072 chunks |
| Authoritative first-lookup working set | 4,012.81 chunks mean |
| Final matched external set | 3,604.08 chunks mean |
| Admission→first lookup | 7.48 ms p50; 25.15 ms p95 |
| First lookup→ready | 16.59 ms p50; 976.51 ms p95 |

Figure 1 shows the four native request-boundary quantiles reconstructed from the normalized corpus. Source: full-resolution C32 request events; no sampling or smoothing.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — C32 native request-boundary timing","width":620,"height":280,"data":{"values":[{"boundary":"Admission→lookup p50","latency_ms":7.483},{"boundary":"Admission→lookup p95","latency_ms":25.145},{"boundary":"Lookup→ready p50","latency_ms":16.593},{"boundary":"Lookup→ready p95","latency_ms":976.513}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"boundary","type":"nominal","title":"Request lifecycle boundary","axis":{"labelAngle":-20}},"y":{"field":"latency_ms","type":"quantitative","title":"Elapsed time (ms)","scale":{"zero":true}},"color":{"field":"boundary","type":"nominal","title":"Boundary","scale":{"scheme":"category10"}},"tooltip":[{"field":"boundary","type":"nominal","title":"Boundary"},{"field":"latency_ms","type":"quantitative","title":"Latency (ms)","format":".3f"}]}}
~~~

The admission window is normally milliseconds, while first-demand recovery has a long tail.

## Interpretation and decision

This gate establishes trace fidelity, not prefetch benefit. It licenses offline counterfactual work on C32. It also corrects the earlier approximately 4,062-chunk summary: 4,012.81 is the exact working-set snapshot referenced by each request's first connector lookup; the larger value came from a later/max snapshot.

Source run: [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow). See also [[Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration|the full corpus report]].

**Next gate:** hand-check single-request transfer feasibility before adding global contention.