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

**A4 instrumentation baseline complete — accepted exports support I0–I1 only; I2–I5 require producer-side semantic capture.**

A1–A3 establish:

1. The previous turn's known working set almost exactly identifies next-continuation KV demand.
2. Continuation intent contains substantial gross placement value.
3. Unconditional or long-lived retention causes harmful substitution under pressure.
4. A 30-second soft TTL weakly dominates LRU but does not beat recorded placement at native capacity.
5. Whole-continuation and marginal-readiness allocation never beat independent block allocation at 192/256/320 GiB.
6. The next discriminating signal—lifecycle/tool/workflow state—is absent from the accepted traces.

Live retention remains gated. A4 now defines the strict capture contract and proves that existing exports cannot answer the semantic question.

## Experiment sequence

| ID | Experiment | Status | Latest decision |
|---|---|---|---|
| A0 | Semantic trace enrichment and replay certification | **Complete** | Existing traces support A1–A3; explicit semantics absent |
| A1 | Continuation-retention oracle | **Complete** | Strong signal go; whole-continuation live policy rejected |
| A2 | CPU retention TTL frontier | **Complete** | 30 s baseline-only conditional pass; no live go |
| A3 | Request-readiness-aware allocation | **Complete** | **KILL grouping allocator**; retain readiness as metric |
| A4 | Semantic information-value ladder | **Instrumentation baseline complete** | Add producer-side I2–I5 fields, then matched c32/c64 capture |
| A5 | Lightweight execution model | Gated | Only if A4 supports it |
| A6 | Route-to-data versus move-to-request | Requires multi-replica corpus | Not started |
| A7 | Tool/workflow-event prefetch feasibility | Requires semantic event timestamps | Not started |
| A8 | Combined action oracle | Gated on A6/A7 | Not started |
| B1–B7 | Live validation program | Gated on A4 evidence | Not started |

## Key documents

- [[Research/ABC/Continuation Readiness/05 - A4 semantic information capture audit|A4 semantic information capture audit]]
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

## A4 baseline

| Corpus | Records | I0 | I1 | I2 | I3 | I4 | I5 |
|---|---:|---:|---:|---:|---:|---:|---:|
| c32 | 897 | 897 | 897 | 0 | 0 | 0 | 0 |
| c64 | 1,262 | 1,262 | 1,262 | 0 | 0 | 0 | 0 |

I2–I5 are not measurable until lifecycle/tool/workflow fields are captured at the producer boundary and preserved in the client export.