---
title: "Version 2 — Strategy and Re-sequencing"
date: "2026-08-18"
type: "strategy"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "active"
related: "[[../Methodology/01 - Experiment Definition]]"
---

# Version 2 — Strategy and Re-sequencing

This note records why the ABC program is re-sequenced as of 2026-08-18, what changes relative to the original four-phase plan in [[../Methodology/01 - Experiment Definition]], what is preserved, and where the defensible win is.

## Why re-sequence now

**1. V1 evidence says blind selection has hit its ceiling.** The V1 admission-time oracle (first-N, assume-resident) proved the wiring but not the policy:

- Concurrency 32 (AgentX Weka): 90.99% of attempted blocks redundant, 87.08% of submitted promotions load-failed, 98.50% late at first demand. Mean waiting depth < 0.25 gave almost no lead time.
- Concurrency 64: useful/attempted rose to 15.81% and late/promoted fell to 42.39% — queue pressure creates the lead time the policy needs — but mean/p95 TTFT were 3.19%/10.44% *worse* than control, with node confounding.
- The V1 index already anticipated this: "if all first-N settings remain mostly redundant or missing, stop increasing N and move selection to a later window or a reuse/residency heuristic."

The bottleneck is **selection quality and residency knowledge**, not the mechanism. Tuning a blind N further has low marginal value.

**2. The literature is converging on the V1 end-state.** TokenCake, KVFlow, PEEK, RelayCaching, and AgentKVShift (all 2025) collectively occupy the "event-driven KV management for agentic workloads" idea space. The research synthesis in [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization]] independently re-derived the ABC end-state from that literature. The window for claiming the *idea* is closing; the window for owning the *production integration* is still open.

**3. The deterministic insight collapses two phases.** The original plan spent Phase 2 on hand-tuned feature rules and Phase 3 on XGBoost temperature prediction. The synthesis doc's core argument — agentic execution is a deterministic state machine (inference → tool call → inference; multi-agent handoffs) — means the strongest access predictors are **orchestration events available for free**, not learned features. A deterministic event-driven temperature model plus a kinetic AET estimator for background blocks covers what the ML model was for, at a fraction of the implementation cost, with no training-data prerequisite, and with far better upstream reviewability (a serving-system reviewer will rightly ask "why does this need a model?").

## What changes

| Aspect | V1 plan | V2 |
|---|---|---|
| Temperature source | Phase 2 rules → Phase 3 XGBoost | Single deterministic model: orchestration events + session affinity + AET |
| Selection granularity | First-N prefix chunks (blind) | Scored chunks across the session working set, residency-checked |
| Residency | Assumed (bypassed lookup) | Verified via the real lookup machinery (fixes the 87% load-failure mode) |
| Migration gating | None in Phase 1 toy | Cost gate $\text{Benefit} > N \times \text{Cost}$ from the first heuristic |
| Background blocks | LRU (reactive) | AET-tracked, graceful demotion before hard eviction |
| ML | End goal | Optional ablation only |

## What is preserved

- The four-tier hierarchy (GPU HBM / CPU DRAM / NVMe / CephFS) and the cost-benefit migration gate.
- The AgentX Weka benchmark (`semianalysisai/cc-traces-weka-with-subagents-060826`) as the differentiating workload.
- The measurement discipline: paired repetitions, node balancing, MLflow registry, exact selection accounting (useful/attempted, load_failed/promoted, late/promoted, redundant/attempted) before any latency claim.
- The V1 mechanism assets: admission-time trigger wiring, `kv_transfer_params` event channel, Prometheus counter discipline, and the repaired manager/scheduler contract in the local vLLM tree.
- The credibility bar: V1 showed prefetch can *hurt* the tail when the policy is wrong. V2 leads with the cost gate as harm prevention and claims wins only where lead time exists.

## Where the defensible win is

Not the idea — the integration. Nobody upstream has: (1) temperature as a first-class, exportable signal consumed by the llm-d Endpoint Picker for routing and session-migration prefetch; (2) cost-gated multi-tier placement inside vLLM's `kv_offload` stack; (3) chunk-level (256-token) temperature semantics matching the transfer granularity. The proposition is the integrated system — vLLM connector + llm-d routing + cost model — validated on agentic traces, with an early upstream RFC as claim-staking (cf. the Mooncake connector RFC, vllm-project/vllm#38474).

## Explicitly parked

Research-grade items from the synthesis doc that are *not* in the V2 critical path: probe-guided residual correction for semantic drift (C^2KV/AgentKVShift), asymmetric FP8/INT4 tier quantization, cross-model prefix reuse via aLoRA, hardware-aware adaptive recomputation fallback (CacheTune-style). These are future-work sections in the RFC, not core claims.
