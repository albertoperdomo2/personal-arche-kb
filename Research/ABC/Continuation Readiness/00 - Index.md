---
title: "COSTAR Continuation Readiness — Index"
date: "2026-08-26"
type: "research-index"
experiment: "COSTAR Continuation Readiness"
status: "active"
---

# COSTAR Continuation Readiness

## Research question

How much of the measured finite-capacity CPU-placement oracle can be recovered by bounded, continuation-aware, request-readiness-aware soft retention using information available before the next request reaches vLLM?

## Current status

**A0 complete — existing c32/c64 artifacts are certified for A1–A3.**

The authoritative continuation identity is `x_correlation_id`; `conversation_id` is reusable source/content identity and can occur in multiple concurrent replay instances. All 1,838 observed continuation edges are consecutive and ordered under `x_correlation_id`.

Explicit lifecycle, tool, and workflow events are absent. A4's richer semantic regimes and A7 require a new capture, but that capture is not required before the continuation oracle, TTL frontier, and readiness-allocation experiments.

## Experiment sequence

| ID | Experiment | Status | Latest decision |
|---|---|---|---|
| A0 | Semantic trace enrichment and replay certification | **Complete** | Existing traces support A1–A3; explicit semantics absent |
| A1 | Continuation-retention oracle | **Next** | Run offline on c32/c64 |
| A2 | CPU retention TTL frontier | Pending | Existing traces sufficient with censoring |
| A3 | Request-readiness-aware allocation | Pending | Existing traces sufficient |
| A4 | Semantic information-value ladder | Blocked on enriched trace for I2–I5 | I0–I1 possible now |
| A5 | Lightweight execution model | Gated | Only if A4 supports it |
| A6 | Route-to-data versus move-to-request | Requires multi-replica corpus | Not started |
| A7 | Tool/workflow-event prefetch feasibility | Requires semantic event timestamps | Not started |
| A8 | Combined action oracle | Gated on A6/A7 | Not started |
| B1–B7 | Live validation program | Gated on offline evidence | Not started |

## Key documents

- [[COSTAR Continuation Readiness A0 — semantic trace certification]]
- [[Research/ABC/Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration]]
- [[Research/ABC/Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle]]
- [[Research/ABC/Future-Value Placement/01 - Experiment 1 matched next-use admission decomposition]]
- [[Research/ABC/2026-08-21 - Independent research audit and redirection for speculative KV prefetching]]

## Accepted corpus registry

| Corpus | Run | Role |
|---|---|---|
| c32 | [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow) | Baseline continuation oracle corpus |
| c64 | [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow) | Independent higher-pressure corpus |

## Next experiment

Implement A1 as a finite-capacity, mandatory-admission, retention-only replay:

- no proactive reads;
- no bypass;
- no hard reserve or lease;
- `x_correlation_id` continuation identity;
- observed adjacent turn as oracle continuation;
- terminal turns right-censored;
- aborted/unidentified server requests retained as unknown/native-priority physical traffic;
- primary metric: fraction of finite next-use oracle secondary-service seconds recovered.