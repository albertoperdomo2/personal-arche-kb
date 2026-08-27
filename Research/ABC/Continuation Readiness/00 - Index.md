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

**A2 complete — a 30-second soft TTL weakly dominates LRU across the capacity sweep, but no static TTL is safe versus recorded placement at the native 256 GiB capacity.**

A1 proved that continuation intent contains substantial oracle value. A2 shows that elapsed time alone captures only a small, pressure-sensitive fraction of it:

- 30 s is neutral versus LRU on c32 and improves c64 by 6/3/10 complete requests at 192/256/320 GiB;
- at native capacity, 30 s still has +3 c32 and +37 c64 net misses versus recorded placement;
- long TTLs are dangerous: c64 1,800 s creates 502 net new misses;
- the best per-trace point differs sharply: 300 s at c32 and 10 s at c64.

The next gate is A3 request-readiness-aware allocation. Thirty seconds is retained as a robust block-level baseline only; live vLLM work remains gated.

## Experiment sequence

| ID | Experiment | Status | Latest decision |
|---|---|---|---|
| A0 | Semantic trace enrichment and replay certification | **Complete** | Existing traces support A1–A3; explicit semantics absent |
| A1 | Continuation-retention oracle | **Complete** | Strong signal go; whole-continuation live policy rejected |
| A2 | CPU retention TTL frontier | **Complete** | 30 s baseline-only conditional pass; no live go |
| A3 | Request-readiness-aware allocation | **Next** | Must beat 30 s on readiness and net miss substitution |
| A4 | Semantic information-value ladder | Blocked on enriched trace for I2–I5 | I0–I1 possible now |
| A5 | Lightweight execution model | Gated | Only if A4 supports it |
| A6 | Route-to-data versus move-to-request | Requires multi-replica corpus | Not started |
| A7 | Tool/workflow-event prefetch feasibility | Requires semantic event timestamps | Not started |
| A8 | Combined action oracle | Gated on A6/A7 | Not started |
| B1–B7 | Live validation program | Gated on A3 evidence | Not started |

## Key documents

- [[Research/ABC/Continuation Readiness/03 - A2 bounded soft-TTL frontier|A2 bounded soft-TTL frontier]]
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

A1 clears the information-value gate but rejects unconditional whole-continuation retention.

## A2 headline evidence at 256 GiB

| Metric | c32, 30 s | c32 best request point, 300 s | c64, 30 s | c64 best request point, 10 s |
|---|---:|---:|---:|---:|
| Complete external targets | 27/42 | 29/42 | 540/789 | 545/789 |
| Gross oracle service recovery | 15.42% | 21.65% | 26.13% | 23.06% |
| Net external miss delta | +3 | +1 | +37 | +32 |
| Protected GiB-hours | 18.04 | 88.11 | 34.24 | 16.70 |

Thirty seconds is the only tested common TTL that is never worse than LRU across both traces and 192/256/320 GiB.

## Next experiment

Implement A3 as an offline, equal-capacity, request-readiness-aware allocation comparison:

- no-TTL LRU;
- 30 s A2 TTL baseline;
- A1 positive-only continuation oracle;
- matched finite next-use reference;
- independent block score;
- whole-continuation value density;
- prefix-frontier density;
- marginal-readiness allocation;
- shared-prefix incremental capacity accounting.

Primary outcome: complete external requests and net miss substitution. Gross service and key-hit rate remain diagnostics. Do not begin live retention unless A3 materially improves request readiness over 30 s without creating a new c64 regression.