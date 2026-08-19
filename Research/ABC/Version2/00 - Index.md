---
title: "ABC Version 2 — Event-Driven Temperature Prefetching"
date: "2026-08-19"
type: "research-index"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "active-revision-required"
---

# ABC Version 2

Version 2 explores deterministic, event- and queue-informed KV promotion before learned prediction. The direction is conditionally valid, but the 2026-08-19 theoretical and code-grounded review found that the current V2.1 design is not implementation-ready. Documents 01–03 are retained as design inputs and must be revised to the validated sequence in document 04.

## Documents

- [[01 - Strategy and Re-sequencing|01 — Strategy and Re-sequencing]] — why V2 exists, what changes versus the original four-phase plan, what is preserved, and where the defensible win is.
- [[02 - Phased Plan|02 — Phased Plan]] — compressed four-track plan (V2.0–V2.3) with objectives, exit criteria, timeline, and the critical path to a demonstrable heuristic.
- [[03 - Event-Driven Temperature Heuristic Implementation Guide|03 — Implementation Guide]] — original vLLM `v0.27.0`-grounded build guide; retained as a design input and marked for revision before implementation.
- [[04 - Theoretical Validation|04 — Theoretical Validation]] — conditional validity verdict, code and research audit, blocking assumptions, corrected proposition, revised phases, and falsifiable hypotheses.

## Related

- [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization|06 — Deep Speculative Prefetching (research synthesis)]] — the literature foundation for V2.
- [[../Methodology/01 - Experiment Definition|01 — Experiment Definition (V1)]] — the original four-phase program.
- [[../00 - Index|ABC project index]] — V1 status, conclusions, and the MLflow run registry.

## Current status

- **Validity:** conditionally valid; revision required before implementation. The working decision is [[04 - Theoretical Validation|04 — Theoretical Validation]], not the earlier “ready to start” statement.
- **Supported premises:** workflow and queue signals can provide useful advance information; V1 proves admission-time promotion mechanically and supports lead time as a control variable; deterministic-before-ML remains the correct sequence.
- **No-go blockers in the current V2.1 design:** admission scoring degenerates to uniform Hot candidates; candidate selection is not defined as an ordered contiguous prefix bundle; async filesystem lookup lacks a re-driven state machine; speculative CPU allocation can evict despite the no-evict claim; AET is misused as a per-key countdown; the cost gate omits lead-time hiding and eviction cost; request-scoped metadata cannot expose the full tool window; and the metric denominator can hide residency misses.
- **Corrected next step:** V2.0 characterization/calibration—standardize the immutable workload, measure lead-time and transfer distributions, run a controlled resident-key microbenchmark, quantify residency/capacity/eviction behavior, and evaluate event predictors offline. Revise documents 01–03 before implementing V2.1.
- **Code grounding:** the theoretical review inspected the local `experimental/naive-proactive-prefetching` branch on 2026-08-19 and checked current primary research plus vLLM/llm-d documentation.
- **V1 state:** mechanism proven; performance inconclusive; blind first-N selection reached its useful ceiling and remains a baseline, not the Version2 policy.