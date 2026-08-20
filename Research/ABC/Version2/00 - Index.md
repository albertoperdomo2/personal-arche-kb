---
title: "ABC Version 2 — Proactive Speculative Prefetching"
date: "2026-08-20"
type: "research-index"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "active"
---

# ABC Version 2

Version 2 pursues **proactive speculative prefetching** of KV cache blocks across storage tiers, driven by deterministic heuristics over signals the serving stack already has — queue state and lead time, exact block residency, session lifecycle, and measured transfer costs. No learned model is involved anywhere in the program; prediction is derived from orchestration structure, not from training.

On 2026-08-19 an independent theoretical and code-grounded review ([[04 - Theoretical Validation|04]]) found the initial V2.1 design conditionally valid but not implementation-ready, and issued nine blocking corrections. **All nine were accepted and documents 01–03 were revised the same day.** The program now follows the corrected sequence in document 02.

**V2.1 has completed its initial five-cell benchmark and a bounded-reserve live-only rerun.** The decision/control plane, exact terminal accounting, and non-evicting safety behavior work. The first live run had no truly free CPU slots; the follow-up proved that the configured 64-block reserve remains logical rather than physical through warmup, so live mode again submitted zero speculative blocks. Start with [[06 - 2026-08-19 - V2.1 Implementation Deep Dive|06]], then read the two executed reports.

## Documents

- [[01 - Strategy and Re-sequencing|01 — Strategy and Re-sequencing]] — V2 problem statement (no learned model), what changes versus V1, what is preserved, the narrowed novelty claim, and the nine standing corrections. **Revised 2026-08-19.**
- [[02 - Phased Plan|02 — Phased Plan]] — corrected plan: V2.0 characterization/calibration → V2.1 residency/deadline admission prefetch → V2.2 lifecycle-event prefetch via out-of-band control → V2.3 retention + placement within vLLM + RFC. llm-d scale-out is V2.4, post-proof and unscheduled. Gated V2.1 start, hypotheses H1–H5, terminal-partition accounting. **Revised 2026-08-19.**
- [[03 - Event-Driven Temperature Heuristic Implementation Guide|03 — Implementation Guide]] — vLLM v0.27.0-grounded build guide: ordered contiguous prefix bundles, async residency state machine, deadline gate, non-evicting speculative allocation, and shadow mode. Superseded on three points by 06: the utility gate, the two-phase reservation, and same-step lookup consumability.
- [[04 - Theoretical Validation|04 — Theoretical Validation]] — adversarial review: conditional validity verdict, code and research audit, blocking assumptions, corrected proposition, revised phases, and falsifiable hypotheses.
- [[05 - V2.1 Implementation Record|05 — V2.1 Implementation Record]] — first implementation note. **Superseded by 06**; retained as the record before the code review and self-calibrating cost model.
- [[06 - 2026-08-19 - V2.1 Implementation Deep Dive|06 — V2.1 Implementation Deep Dive]] — **implementation reference.** End-to-end mechanism, code rationale, observability, and the five-cell run plan.
- [[Reports/00 - Index|Reports]] — executed Version2 experiments. Read [[Reports/2026-08-19 - V2.1 first five-cell comparison|initial comparison]] then [[Reports/2026-08-20 - V2.1 bounded-reserve live validation|bounded-reserve validation]].

## Related

- [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization|06 — Deep Speculative Prefetching (research synthesis)]] — literature foundation.
- [[../Methodology/01 - Experiment Definition|01 — Experiment Definition (V1)]] — original four-phase program.
- [[../00 - Index|ABC project index]] — V1 status, conclusions, and MLflow registry.

## Current status

- **Validity:** selector, async residency state machine, terminal partition, timing model, request gate, and non-evicting safety are validated. The live data plane remains unvalidated because admission_submitted=0 in both completed live batches.
- **New allocator finding:** a configured speculative reserve is insufficient unless ordinary cache persistence physically preserves it. After warmup, the V2.1 live cells reported reserve-free=64 while physical free_blocks=0 and fill=100%, then correctly refused every speculative allocation.
- **Corrected proposition:** an event- and queue-informed controller can reduce critical-path KV retrieval for reusable, contiguous session prefixes only when its destination capacity is physically protected long enough to overlap transfer with queued lead time.
- **Implementation status:** V2.1 has the corrected policy accounting, true CPU occupancy gauges, bounded reserve metrics, and allocator modes. The next implementation task is to distinguish normal persistence, speculative promotion, and reactive demand allocation so only the last can borrow reserved capacity as a last resort.
- **Next experiment:** add a scheduler-path warmup regression that proves N actual free CPU blocks survive, then rerun a node-paired reactive-overlay/live 64/64/64 pair. A matched shadow run for the current live-only checkpoint is pending.
- **V1 state:** retained only as a negative safety control; do not use it as a viable performance competitor.