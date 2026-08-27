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

**A1 complete — continuation identity is a strong signal, but unconditional whole-continuation retention is unsafe at c64.**

A1 recovers 80.42% of the c32 and 61.29% of the c64 matched next-use oracle's avoidable measured service using only oracle knowledge that a concrete continuation will resume. Exact next-turn deadline raises c64 recovery to 65.44%.

The previous turn's known reusable working set covers 100% of c32 and 99.99997% of c64 next-turn external key references. Candidate discovery is therefore not the main problem.

The naïve policy is not ready for vLLM: deadline-ordered c64 retention avoids 95 recorded reads but creates 159 new request misses, a net regression of 64. Protected occupancy reaches the entire CPU cache. The next gate is bounded TTL and request-readiness-aware allocation, not a live whole-session policy.

## Experiment sequence

| ID | Experiment | Status | Latest decision |
|---|---|---|---|
| A0 | Semantic trace enrichment and replay certification | **Complete** | Existing traces support A1–A3; explicit semantics absent |
| A1 | Continuation-retention oracle | **Complete** | Strong signal go; whole-continuation live policy rejected |
| A2 | CPU retention TTL frontier | **Next** | Reduce c64 false protection and quantify GiB-hour efficiency |
| A3 | Request-readiness-aware allocation | Pending | Required because deadline-only retention still regresses c64 |
| A4 | Semantic information-value ladder | Blocked on enriched trace for I2–I5 | I0–I1 possible now |
| A5 | Lightweight execution model | Gated | Only if A4 supports it |
| A6 | Route-to-data versus move-to-request | Requires multi-replica corpus | Not started |
| A7 | Tool/workflow-event prefetch feasibility | Requires semantic event timestamps | Not started |
| A8 | Combined action oracle | Gated on A6/A7 | Not started |
| B1–B7 | Live validation program | Gated on A2/A3 evidence | Not started |

## Key documents

- [[Research/ABC/Continuation Readiness/02 - A1 continuation-retention oracle|A1 continuation-retention oracle]]
- [[Research/ABC/Continuation Readiness/01 - A0 semantic trace certification|A0 semantic trace certification]]
- [[Research/ABC/Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration]]
- [[Research/ABC/Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle]]
- [[Research/ABC/Future-Value Placement/01 - Experiment 1 matched next-use admission decomposition]]
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
| Average protected CPU share | 31.44% | 49.59% | 48.13% |
| Peak protected CPU share | 91.25% | 100% | 100% |

The formal A1 strong-go threshold is met, but the positive c64 net miss delta prevents a live go.

## Next experiment

Implement A2 as a finite-capacity, mandatory-admission, retention-only TTL frontier:

- static soft TTLs: 1, 3, 10, 30, 60, 120, 300, 600, and 1,800 seconds;
- no proactive reads, bypass, reserve, or lease;
- right-censored terminal handling;
- equal capacity and common recorded arrivals;
- report each trace separately;
- primary safety metric: net external miss delta;
- primary opportunity metric: gross oracle service recovery;
- efficiency: protected GiB-hours and service saved per protected GiB-hour.

A2 should determine whether bounded retention preserves most continuation value without letting the protected domain consume the whole cache. A3 follows even if A2 succeeds, because request-level allocation is needed to reduce partial-working-set waste.