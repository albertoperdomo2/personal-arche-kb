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

## Documents

- [[01 - Strategy and Re-sequencing|01 — Strategy and Re-sequencing]] — V2 problem statement (no learned model), what changes versus V1, what is preserved, the narrowed novelty claim, and the nine standing corrections. **Revised 2026-08-19.**
- [[02 - Phased Plan|02 — Phased Plan]] — corrected plan: V2.0 characterization/calibration → V2.1 residency/deadline admission prefetch → V2.2 lifecycle-event prefetch via out-of-band control → V2.3 retention + placement within vLLM + RFC. llm-d scale-out is V2.4, post-proof and unscheduled. Gated V2.1 start, hypotheses H1–H5, terminal-partition accounting. **Revised 2026-08-19.**
- [[03 - Event-Driven Temperature Heuristic Implementation Guide|03 — Implementation Guide]] — vLLM `v0.27.0`-grounded build guide: ordered contiguous prefix bundles, async residency state machine, deadline + utility gate, non-evicting speculative allocation, shadow mode. **Revised 2026-08-19.**
- [[04 - Theoretical Validation|04 — Theoretical Validation]] — the adversarial review: conditional validity verdict, code and research audit, blocking assumptions, corrected proposition, revised phases, and falsifiable hypotheses.

## Related

- [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization|06 — Deep Speculative Prefetching (research synthesis)]] — the literature foundation for V2.
- [[../Methodology/01 - Experiment Definition|01 — Experiment Definition (V1)]] — the original four-phase program.
- [[../00 - Index|ABC project index]] — V1 status, conclusions, and the MLflow run registry.

## Current status

- **Validity:** conditionally valid; the nine blocking corrections from [[04 - Theoretical Validation|04]] are incorporated into 01–03. V2.1 implementation may begin only when the eight start gates in [[02 - Phased Plan|02]] are satisfied.
- **Corrected proposition:** an event- and queue-informed controller can reduce critical-path KV retrieval for reusable, contiguous session prefixes by scheduling residency-verified promotions only when predicted lead time exceeds calibrated transfer time and expected latency benefit exceeds contention and eviction cost.
- **Next step:** V2.0 characterization/calibration — pin the immutable workload (`semianalysisai/cc-traces-weka-062126`), measure lead-time and transfer distributions, run the resident-key microbenchmark, quantify residency/capacity/eviction behavior, evaluate event predictors offline, and declare H1–H5 acceptance bounds.
- **Code grounding:** upstream `vllm-project/vllm` @ `v0.27.0` (commit `4bdc8a78`, GitHub Connector, 2026-08-18); local `experimental/naive-proactive-prefetching` branch (theoretical review, 2026-08-19).
- **V1 state:** mechanism proven; performance inconclusive; blind first-N selection reached its useful ceiling and remains a baseline, not the Version2 policy.
