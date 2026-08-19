---
title: "Version 2 — Strategy and Re-sequencing"
date: "2026-08-19"
type: "strategy"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "active"
supersedes: "2026-08-18 initial version"
revised-after: "[[04 - Theoretical Validation]]"
related: "[[../Methodology/01 - Experiment Definition]]"
---

# Version 2 — Strategy and Re-sequencing

This note records why the ABC program is re-sequenced, what changes relative to the original four-phase plan in [[../Methodology/01 - Experiment Definition]], what is preserved, and where the defensible win is. Revised 2026-08-19 after the theoretical validation in [[04 - Theoretical Validation]]; the standing corrections from that review are listed at the end and are incorporated here and in [[02 - Phased Plan]] and [[03 - Event-Driven Temperature Heuristic Implementation Guide]].

## Problem statement (V2)

Existing KV cache tier management in vLLM is reactive: blocks residing on secondary tiers are fetched only on demand, during admission-time lookup, and eviction happens under capacity pressure. Under agentic workloads — where sessions repeatedly revisit overlapping context and KV cache hit rates are high — this produces avoidable TTFT stalls whenever reusable context sits on a slow tier at the moment it is needed.

The goal of ABC Version 2 is **proactive speculative prefetching**: moving KV cache blocks across storage tiers before they are demanded, driven by deterministic heuristics over signals the serving stack already has — queue state and lead time, exact block residency, session lifecycle, and measured transfer costs — so that critical-path retrieval latency falls without harming the active workload. Prediction is derived from orchestration structure, not from any learned model.

## Why re-sequence now

**1. V1 evidence says blind selection has hit its ceiling.** The V1 admission-time oracle (first-N, assume-resident) proved the wiring but not the policy:

- Concurrency 32 (AgentX Weka): 90.99% of attempted blocks redundant, 87.08% of submitted promotions load-failed, 98.50% late at first demand. Mean waiting depth < 0.25 gave almost no lead time.
- Concurrency 64: useful/attempted rose to 15.81% and late/promoted fell to 42.39% — queue pressure creates the lead time the policy needs — but mean/p95 TTFT were 3.19%/10.44% *worse* than control, with node confounding.
- The V1 index already anticipated this: "if all first-N settings remain mostly redundant or missing, stop increasing N and move selection to a later window or a reuse/residency heuristic."

The bottleneck is **selection quality, residency knowledge, and lead-time gating**, not the mechanism. Tuning a blind N further has low marginal value.

**2. The literature is converging on this end-state.** TokenCake, KVFlow, PEEK, RelayCaching, and AgentKVShift collectively occupy the "event-driven KV management for agentic workloads" idea space. The research synthesis in [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization]] independently re-derived the ABC end-state from that literature. The window for claiming the *idea* is closing; the window for owning the *production integration* is still open.

**3. The heuristic approach is the end-state, not a stepping stone.** The V1 plan ended at a learned temperature model. That phase is **removed, not deferred**: a learned model in the serving hot path is not upstreamable, and the V1 evidence plus the synthesis doc both indicate the strongest access predictors are structural — queue lead time, exact residency, session lifecycle — not learned features. A deterministic heuristic is cheaper to build, falsifiable, reviewable upstream, and it is the deliverable itself.

## What changes

| Aspect | V1 plan | V2 |
|---|---|---|
| Prediction source | Phase 2 rules → Phase 3 learned model | Deterministic heuristics only: admission/queue lead time, exact residency, session lifecycle, measured costs |
| Selection unit | First-N prefix chunks (blind) | Ordered, contiguous **prefix bundles**; the policy chooses bundle length and which bundles earn budget |
| Residency | Assumed (bypassed lookup) | Verified through the async lookup state machine (fixes the 87% load-failure mode) |
| Speculative allocation | Demand path (can evict useful blocks) | Non-evicting reservation or explicitly bounded speculative budget |
| Migration gating | None in Phase 1 toy | Lead-time-aware utility gate in common units (milliseconds) |
| Background retention | LRU (reactive) | TTL / last-access / inter-reuse interval first; AET-like global pressure only after trace validation |
| Learned model | End goal | **Removed from the program** |

## What is preserved

- The tier hierarchy (GPU HBM / CPU DRAM / NVMe / CephFS) as an *aspiration*; V2.1 scope is honestly limited to secondary→CPU promotion (see corrections below).
- The AgentX Weka benchmark, **pinned to the immutable revision `semianalysisai/cc-traces-weka-062126`** used by the V1 evidence (the `060826`/`061526` references elsewhere are superseded for ABC continuity; V2.0 declares the pin formally).
- The measurement discipline: paired repetitions, node balancing, MLflow registry, and terminal-partition accounting (`considered = primary_redundant + secondary_absent + gate_reject + capacity_skip + submitted + lookup_unresolved`; `useful/considered` as the stable yield metric) before any latency claim.
- The V1 mechanism assets: admission-time trigger wiring, the `kv_transfer_params` channel, Prometheus counter discipline, and the repaired manager/scheduler contract in the local vLLM tree.
- The credibility bar: V1 showed prefetch can *hurt* the tail when the policy is wrong. V2 leads with the utility gate and non-evicting allocation as harm prevention, and claims wins only where lead time exists.

## Where the defensible win is

Not the idea, and not cache-aware routing — llm-d already performs precise prefix-cache-aware routing (exact, event-driven, per-pod KV residency indexing combined with load). The defensible delta is narrower and stronger: **predicted future reuse and deadlines derived from session lifecycle state**, combined with exact residency and load.

**Sequencing decision (2026-08-19): vLLM first, llm-d later.** The program proves the mechanism inside a single vLLM engine — residency-verified, deadline-gated promotion decisions made on lead time — before any cluster-level routing or migration work. Only once the single-engine proof exists do we scale out: exporting predictions for llm-d routing and session-migration prefetch becomes the follow-on phase, evaluated against llm-d's existing exact-residency/load baseline. The upstream RFC targets vLLM first (cf. the Mooncake connector RFC, vllm-project/vllm#38474); the llm-d RFC follows the proof.

## Standing corrections from the theoretical validation

Incorporated from [[04 - Theoretical Validation]] (2026-08-19); that note is the canonical reference:

1. Selection unit is an ordered contiguous prefix bundle, never a discontinuous key set (vLLM block hashes are prefix-chained; the demand scan breaks at the first MISS).
2. Residency verification uses an explicit async state machine (`PENDING_LOOKUP → RESIDENT | ABSENT → GATE → SUBMITTED → READY | LATE | FAILED`) with deadlines, cancellation on request completion/preemption, duplicate suppression, and bounded pending state.
3. Speculative promotion must not evict demand-useful CPU blocks: non-evicting reservation API or bounded speculative budget, with telemetry on capacity rejections and any speculative-caused eviction.
4. AET is not a per-key eviction countdown; it is retained, if at all, as a global pressure/retention-horizon signal after trace validation.
5. The gate is a common-unit (milliseconds), lead-time-aware expected utility; V2.1 runs in shadow mode until V2.0 calibration lands.
6. Tool-window lifecycle events cannot flow through request-scoped `kv_transfer_params`; V2.2 requires an out-of-band, session-addressed control API.
7. V2.1 claims secondary→CPU promotion only; independent Hot/Warm/Cool/Cold placement is split into later control surfaces.
8. The workload revision is pinned immutably (`cc-traces-weka-062126`) with declared seed, prompt construction, and session mapping.
9. Accounting uses the terminal partition with `useful/considered` as the stable yield metric; state transitions are logged separately from terminal accounting.

## Explicitly parked

Research-grade items from the synthesis doc that are not in the V2 critical path: probe-guided residual correction for semantic drift (C^2KV/AgentKVShift), asymmetric FP8/INT4 tier quantization, cross-model prefix reuse via aLoRA, hardware-aware adaptive recomputation fallback (CacheTune-style). These are future-work sections in the RFC, not core claims. The V1 plan's learned-model phase is not parked — it is removed.
