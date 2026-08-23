---
title: "ABC Methodology and Implementation"
date: "2026-08-18"
type: "research-methodology-index"
experiment: "ABC"
status: "active"
---

# ABC Methodology and Implementation

Definitions, experiment plans, implementation tutorials, and design discussions live here.

## Core methodology

- [[Methodology/01 - Experiment Definition|01 — Experiment Definition]] — research question and phased program.
- [[Methodology/05 - Initial versus Admission-Time Proactive Prefetching|05 — Initial versus Admission-Time Proactive Prefetching]] — concise explanation of how the current implementation works and why the original policy failed.
- [[Methodology/2026-08-14 - Phase 1 queued-request oracle prefetch plan|2026-08-14 — Queued-request oracle prefetch plan]] — experimental plan that isolates proactive timing from candidate prediction.

## Research synthesis

- [[Methodology/06 - Deep Speculative Prefetching and Temperature Characterization|06 — Deep Speculative Prefetching and Temperature Characterization]] — literature synthesis motivating the event-driven temperature design.

- [[07 - Dynamic admission and cross-scope prefetch roadmap|07 — Dynamic admission and cross-scope prefetch roadmap]] — model-neutral byte/deadline admission policy and the additional components required for cross-vLLM, temperature, and cross-session prefetch.

## Implementation guides

- [[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide|02 — Phase 1 Naive Prefetch Implementation Guide (historical)]] — rejected post-miss same-request read-ahead implementation, retained in two linked parts.
- [[Methodology/04 - Phase 1 Queued-Request Oracle Prefetch Implementation Guide|04 — Phase 1 Queued-Request Oracle Prefetch Implementation Guide]] — current admission-time, assume-resident proof-of-concept tutorial.
- [[Methodology/03 - Phase 2 Heuristic Prefetch Implementation Guide|03 — Phase 2 Heuristic Prefetch Implementation Guide (tentative)]] — future adaptive and speculative direction, pending the performance proof.

Return to [[00 - Index|ABC project index]].