---
title: "COSTAR Continuation Readiness — Index"
date: "2026-08-27"
type: "research-index"
experiment: "COSTAR Continuation Readiness"
status: "active"
---

# COSTAR Continuation Readiness

## Research question

How much of the measured finite-capacity CPU-placement oracle can be recovered by bounded, continuation-aware, request-readiness-aware soft retention using information available before the next request reaches vLLM?

## Current status

**A3 complete — request-level grouping and shared-prefix marginal allocation are rejected for this workload. The unresolved problem is continuation eligibility/value, not set packing.**

A1–A3 now establish:

1. The previous turn's known working set almost exactly identifies next-continuation KV demand.
2. Continuation intent contains substantial gross placement value.
3. Unconditional or long-lived retention causes harmful substitution under pressure.
4. A 30-second soft TTL weakly dominates LRU but does not beat recorded placement at native capacity.
5. Whole-continuation and marginal-readiness allocation never beat independent block allocation at 192/256/320 GiB.
6. The next discriminating signal—lifecycle/tool/workflow state—is absent from the accepted traces.

Live retention remains gated. The next work is an A4 semantically enriched capture and information-value ladder, not a complex vLLM allocator.

## Experiment sequence

| ID | Experiment | Status | Latest decision |
|---|---|---|---|
| A0 | Semantic trace enrichment and replay certification | **Complete** | Existing traces support A1–A3; explicit semantics absent |
| A1 | Continuation-retention oracle | **Complete** | Strong signal go; whole-continuation live policy rejected |
| A2 | CPU retention TTL frontier | **Complete** | 30 s baseline-only conditional pass; no live go |
| A3 | Request-readiness-aware allocation | **Complete** | **KILL grouping allocator**; retain readiness as metric |
| A4 | Semantic information-value ladder | **Next; requires enriched trace for I2–I5** | Instrument lifecycle/tool/workflow state |
| A5 | Lightweight execution model | Gated | Only if A4 supports it |
| A6 | Route-to-data versus move-to-request | Requires multi-replica corpus | Not started |
| A7 | Tool/workflow-event prefetch feasibility | Requires semantic event timestamps | Not started |
| A8 | Combined action oracle | Gated on A6/A7 | Not started |
| B1–B7 | Live validation program | Gated on A4 evidence | Not started |

## Key documents

- [[Research/ABC/Continuation Readiness/04 - A3 request-readiness allocation|A3 request-readiness allocation]]
- [[Research/ABC/Continuation Readiness/03 - A2 bounded soft-TTL frontier|A2 bounded soft-TTL frontier]]
- [[Research/ABC/Continuation Readiness/02 - A1 continuation-retention oracle|A1 continuation-retention oracle]]
- [[Research/ABC/Continuation Readiness/01 - A0 semantic trace certification|A0 semantic trace certification]]
- [[Research/ABC/Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration]]
- [[Research/ABC/Future-Value Placement/06 - Experiment 6 C64 independent pressure validation]]
- [[Research/ABC/2026-08-21 - Independent research audit and redirection for speculative KV prefetching]]

## Accepted corpus registry

| Corpus | Run | Role |
|---|---|---|
| c32 | [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow) | Baseline continuation oracle corpus |
| c64 | [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow) | Independent higher-pressure corpus |

## A1 headline evidence

| Metric | c32 | c64 resume-only | c64 exact deadline |
|---|---:|---:|---:|
| Gross oracle service recovery | 80.42% | 61.29% | 65.44% |
| Recorded reads avoided | 10/12 | 90/212 | 95/212 |
| Net external miss delta | -6 | +79 | +64 |

A1 clears the information-value gate but rejects unconditional whole-continuation retention.

## A2 headline evidence at 256 GiB

| Metric | c32, 30 s | c64, 30 s |
|---|---:|---:|
| Complete external targets | 27/42 | 540/789 |
| Gross oracle service recovery | 15.42% | 26.13% |
| Net external miss delta | +3 | +37 |
| Protected GiB-hours | 18.04 | 34.24 |

Thirty seconds is the only tested common TTL that is never worse than LRU across both traces and 192/256/320 GiB.

## A3 headline evidence

| Corpus / capacity | TTL 30 s | Independent block | Whole continuation | Marginal readiness |
|---|---:|---:|---:|---:|
| c32 / 192 GiB | 18 | 31 | 31 | 31 |
| c32 / 256 GiB | 27 | 36 | 36 | 36 |
| c32 / 320 GiB | 34 | 40 | 40 | 40 |
| c64 / 192 GiB | 412 | **449** | 445 | 445 |
| c64 / 256 GiB | **540** | 509 | 504 | 504 |
| c64 / 320 GiB | **624** | 571 | 571 | 571 |

Values are complete external requests. Grouping provides zero positive cells and loses under c64 pressure.

## Next experiment

A4 requires a fresh matched AgentX capture with the existing KV oracle events plus:

- explicit lifecycle state;
- tool start/end and tool class;
- agent/workflow node;
- candidate successor set;
- explicit session terminal/close reason;
- early application event timestamps;
- stable root, continuation, turn, and request IDs.

Then evaluate the incremental decision value of:

- I0: ordinary key/history;
- I1: continuation identity and elapsed time;
- I2: lifecycle state;
- I3: tool/agent class;
- I4: workflow node/candidate successors;
- I5: exact application-known successor;
- I6: future oracle.

Primary metrics remain net external miss substitution and finite-oracle service recovery. Do not train a model or implement live eviction until one added information regime materially improves decisions beyond 30-second TTL.