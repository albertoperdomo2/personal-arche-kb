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

**V2.1 is implemented and has completed its first five-cell benchmark.** The decision/control plane and non-evicting safety behavior worked, but live mode submitted zero speculative blocks because warmup left no truly free CPU KV slots. Start with [[06 - 2026-08-19 - V2.1 Implementation Deep Dive|06]], then read [[Reports/2026-08-19 - V2.1 first five-cell comparison|the initial comparison report]].

## Documents

- [[01 - Strategy and Re-sequencing|01 — Strategy and Re-sequencing]] — V2 problem statement (no learned model), what changes versus V1, what is preserved, the narrowed novelty claim, and the nine standing corrections. **Revised 2026-08-19.**
- [[02 - Phased Plan|02 — Phased Plan]] — corrected plan: V2.0 characterization/calibration → V2.1 residency/deadline admission prefetch → V2.2 lifecycle-event prefetch via out-of-band control → V2.3 retention + placement within vLLM + RFC. llm-d scale-out is V2.4, post-proof and unscheduled. Gated V2.1 start, hypotheses H1–H5, terminal-partition accounting. **Revised 2026-08-19.**
- [[03 - Event-Driven Temperature Heuristic Implementation Guide|03 — Implementation Guide]] — vLLM `v0.27.0`-grounded build guide: ordered contiguous prefix bundles, async residency state machine, deadline + utility gate, non-evicting speculative allocation, shadow mode. **Revised 2026-08-19.** Superseded on three points by 06: the utility gate, the two-phase reservation, and same-step lookup consumability.
- [[04 - Theoretical Validation|04 — Theoretical Validation]] — the adversarial review: conditional validity verdict, code and research audit, blocking assumptions, corrected proposition, revised phases, and falsifiable hypotheses.
- [[05 - V2.1 Implementation Record|05 — V2.1 Implementation Record]] — first implementation note. **Superseded by 06**; retained as the record of the state before the code review and the self-calibrating cost model.
- [[06 - 2026-08-19 - V2.1 Implementation Deep Dive|06 — V2.1 Implementation Deep Dive]] — **the implementation reference.** What was built and why, the mechanism explained end to end with code and equations, observability, and the five-cell run plan.
- [[Reports/00 - Index|Reports]] — executed Version2 experiments and native-cadence telemetry. The first report is [[Reports/2026-08-19 - V2.1 first five-cell comparison|V2.1 first five-cell comparison]].

## Related

- [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization|06 — Deep Speculative Prefetching (research synthesis)]] — the literature foundation for V2.
- [[../Methodology/01 - Experiment Definition|01 — Experiment Definition (V1)]] — the original four-phase program.
- [[../00 - Index|ABC project index]] — V1 status, conclusions, and the MLflow run registry.

## Current status

- **Validity:** design remains sound, and the first benchmark validates the selector, async residency state machine, terminal partition, timing model, shadow isolation, and non-evicting safety. The live data plane is not yet validated because `admission_submitted=0`.
- **Corrected proposition:** an event- and queue-informed controller can reduce critical-path KV retrieval for reusable, contiguous session prefixes by scheduling residency-verified promotions only when predicted lead time exceeds calibrated transfer time and expected latency benefit exceeds contention and eviction cost.
- **Implementation:** 17 files, +4,425 lines, 82 new tests in the dedicated suites. Three commits on `experimental/v2-admission-prefetch`. Image `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v2` overlays the change onto `vllm/vllm-openai:v0.27.0` and serves every benchmark cell by config alone.
- **Cost model is self-calibrating:** transfer cost is fitted from real promotions, including demand-driven ones, so it is calibrated in shadow mode before any speculative byte moves. Four uncalibrated knobs and the inert utility gate were removed as a result.
- **Next step:** split capacity-skip reasons; expose true free/allocated/evictable/speculative CPU blocks; reserve a bounded speculative headroom; and rerun live with a required nonzero-submission correctness gate.
- **Observed timing:** the deadline was not binding—only one fully-resolved bundle per V2.1 cell failed it—but the calibrated 64-chunk cost finished near 0.53–0.63 s, not the earlier ~0.22 s assumption. Physical CPU free capacity was the binding constraint.
- **V1 state:** retained only as a negative safety control. In this batch it initiated 18,144 promotions, load-failed 5,739, wasted 10,751, used 1,641, and collapsed throughput.