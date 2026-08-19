---
title: "ABC Version 2 — Event-Driven Temperature Prefetching"
date: "2026-08-18"
type: "research-index"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "active"
---

# ABC Version 2

Version 2 re-sequences the ABC program around a deterministic, event-driven temperature heuristic. It collapses the original Phase 2 (hand-tuned rules) and Phase 3 (XGBoost ML) of the V1 program into a single deterministic phase, on the evidence that in agentic workloads the strongest access predictors are orchestration events, not learned features.

## Documents

- [[01 - Strategy and Re-sequencing|01 — Strategy and Re-sequencing]] — why V2 exists, what changes versus the original four-phase plan, what is preserved, and where the defensible win is.
- [[02 - Phased Plan|02 — Phased Plan]] — compressed four-track plan (V2.0–V2.3) with objectives, exit criteria, timeline, and the critical path to a demonstrable heuristic.
- [[03 - Event-Driven Temperature Heuristic Implementation Guide|03 — Implementation Guide]] — vLLM `v0.27.0`-grounded build guide: exact files, classes, line numbers, event schema, config surface, metrics, and validation plan.

## Related

- [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization|06 — Deep Speculative Prefetching (research synthesis)]] — the literature foundation for V2.
- [[../Methodology/01 - Experiment Definition|01 — Experiment Definition (V1)]] — the original four-phase program.
- [[../00 - Index|ABC project index]] — V1 status, conclusions, and the MLflow run registry.

## Current status

- V2.0 (close V1 + cost curve) and V2.1 (temperature-gated admission prefetch) are ready to start.
- Code grounding: `vllm-project/vllm` @ tag `v0.27.0`, commit `4bdc8a788d2e2ce9165d552b3d4d8b72604626bf`, inspected via the GitHub Connector on 2026-08-18. Upstream `vllm/v1/kv_offload` contains **no** prefetch code — the design space is unoccupied upstream.
- V1 state: mechanism proven (repaired image, 2026-08-18); performance inconclusive; blind first-N selection ceiling reached (90.99% redundant at concurrency 32; 15.81% useful at concurrency 64).
