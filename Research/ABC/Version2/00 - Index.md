---
title: "ABC Version 2 — Proactive Speculative Prefetching"
date: "2026-08-19"
type: "research-index"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "active"
---

# ABC Version 2

Version 2 pursues **proactive speculative prefetching** of KV cache blocks across storage tiers, driven by deterministic heuristics over signals the serving stack already has — queue state and lead time, exact block residency, session lifecycle, and measured transfer costs. No learned model is involved anywhere in the program; prediction is derived from orchestration structure, not from training.

On 2026-08-19 an independent theoretical and code-grounded review ([[04 - Theoretical Validation|04]]) found the initial V2.1 design conditionally valid but not implementation-ready, and issued nine blocking corrections. **All nine were accepted and documents 01–03 were revised the same day.** The program now follows the corrected sequence in document 02.

**V2.1 is implemented** on the fork branch [`albertoperdomo2/vllm @ experimental/v2-admission-prefetch`](https://github.com/albertoperdomo2/vllm/tree/experimental/v2-admission-prefetch), shadow mode by default, not yet benchmarked. Start with [[06 - 2026-08-19 - V2.1 Implementation Deep Dive|06]].

## Documents

- [[01 - Strategy and Re-sequencing|01 — Strategy and Re-sequencing]] — V2 problem statement (no learned model), what changes versus V1, what is preserved, the narrowed novelty claim, and the nine standing corrections. **Revised 2026-08-19.**
- [[02 - Phased Plan|02 — Phased Plan]] — corrected plan: V2.0 characterization/calibration → V2.1 residency/deadline admission prefetch → V2.2 lifecycle-event prefetch via out-of-band control → V2.3 retention + placement within vLLM + RFC. llm-d scale-out is V2.4, post-proof and unscheduled. Gated V2.1 start, hypotheses H1–H5, terminal-partition accounting. **Revised 2026-08-19.**
- [[03 - Event-Driven Temperature Heuristic Implementation Guide|03 — Implementation Guide]] — vLLM `v0.27.0`-grounded build guide: ordered contiguous prefix bundles, async residency state machine, deadline + utility gate, non-evicting speculative allocation, shadow mode. **Revised 2026-08-19.** Superseded on three points by 06: the utility gate, the two-phase reservation, and same-step lookup consumability.
- [[04 - Theoretical Validation|04 — Theoretical Validation]] — the adversarial review: conditional validity verdict, code and research audit, blocking assumptions, corrected proposition, revised phases, and falsifiable hypotheses.
- [[05 - V2.1 Implementation Record|05 — V2.1 Implementation Record]] — first implementation note. **Superseded by 06**; retained as the record of the state before the code review and the self-calibrating cost model.
- [[06 - 2026-08-19 - V2.1 Implementation Deep Dive|06 — V2.1 Implementation Deep Dive]] — **the current reference.** What was built and why, the mechanism explained end to end with code and equations, the KB-derived constants, the code-review findings and fixes, observability, the five-cell run plan, the falsifiable expectation, and related work. **New 2026-08-19.**

## Related

- [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization|06 — Deep Speculative Prefetching (research synthesis)]] — the literature foundation for V2.
- [[../Methodology/01 - Experiment Definition|01 — Experiment Definition (V1)]] — the original four-phase program.
- [[../00 - Index|ABC project index]] — V1 status, conclusions, and the MLflow run registry.

## Current status

- **Validity:** sound after three review rounds (2026-08-19): nine blocking corrections, six follow-up refinements, and a code-grounded re-validation against the local tree. See the addenda in [[04 - Theoretical Validation|04]].
- **Corrected proposition:** an event- and queue-informed controller can reduce critical-path KV retrieval for reusable, contiguous session prefixes by scheduling residency-verified promotions only when predicted lead time exceeds calibrated transfer time and expected latency benefit exceeds contention and eviction cost.
- **Implementation:** 17 files, +4,425 lines, 82 new tests in the dedicated suites. Three commits on `experimental/v2-admission-prefetch`. Image `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v2` overlays the change onto `vllm/vllm-openai:v0.27.0` and serves every benchmark cell by config alone.
- **Cost model is self-calibrating:** transfer cost is fitted from real promotions, including demand-driven ones, so it is calibrated in shadow mode before any speculative byte moves. Four uncalibrated knobs and the inert utility gate were removed as a result.
- **Next step:** the five-cell sequence in [[06 - 2026-08-19 - V2.1 Implementation Deep Dive|06]] §12. Cell 3 (shadow) doubles as V2.0 characterization and is a decision gate for cells 4 and 5.
- **Falsifiable expectation:** Little's law on the C64 run gives ~9.2 s of queue wait against ~220 ms of transfer for a 100-chunk bundle, so the deadline gate should rarely reject and residency should be the binding constraint.
- **V1 state:** mechanism proven; performance inconclusive; blind first-N selection reached its useful ceiling and remains a baseline, not the Version2 policy.