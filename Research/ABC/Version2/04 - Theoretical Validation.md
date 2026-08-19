---
title: "ABC Version 2 — Theoretical Validation"
date: "2026-08-19"
type: "research-validation"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "resolved-incorporated"
scope: "Version2 proposition, research basis, implementation assumptions, and hypotheses"
codebase: "local vLLM experimental/naive-proactive-prefetching branch, inspected 2026-08-19"
---

# ABC Version 2 — Theoretical Validation

## Executive summary

This checkpoint asks whether the Version 2 proposition in [[01 - Strategy and Re-sequencing]], [[02 - Phased Plan]], and [[03 - Event-Driven Temperature Heuristic Implementation Guide]] is logically sound enough to implement and benchmark.

The broad direction is sound: orchestration state and queue lead time can provide advance knowledge that reactive cache replacement does not have, and deterministic policies are the correct first step before learned prediction. Version 1 also established that admission-time NVMe→CPU promotion is mechanically possible and that useful yield improves when requests have enough queue lead time.

The current V2.1 design is **not implementation-ready**, however. Its proposed scoring path makes every admission candidate hot, does not actually use `abc_event` to rank candidates, lacks the asynchronous lookup state machine needed by the existing filesystem tier, can evict useful CPU blocks despite claiming that prefetch will not evict, and treats Average Eviction Time (AET) as a per-key predictor even though the cited method estimates population-level cache behavior. The cost gate also needs a single-unit, lead-time-aware definition, and tool-window events require an out-of-band control path rather than request-scoped metadata.

No benchmark result is claimed in this note. This is a theoretical and code-grounded validation of the proposition and its assumptions.

## Validity verdict

# Conditionally valid — revision required before implementation

| Question | Verdict | Reason |
|---|---|---|
| Can workflow events improve proactive KV decisions? | Supported as a research hypothesis | Prior work shows that orchestration/workflow signals can outperform purely reactive cache policies, and V1 confirms queue lead time matters. |
| Is deterministic-before-ML sequencing appropriate? | Yes | It isolates mechanism and policy effects and avoids training on an unstable telemetry contract. |
| Is the documented V2.1 heuristic an event-driven temperature policy? | No | At admission it assigns the same hot state to every candidate and the score does not use the event type. |
| Can the documented one-shot lookup path work with the current async filesystem tier? | No | The first lookup can return retry/pending and completes only after the scheduler hook has passed. |
| Does the current implementation guarantee that speculative promotion never evicts demand-useful CPU KV? | No | CPU destination allocation can evict its LRU victim. |
| Is AET valid as a per-block eviction countdown? | No | The cited AET mechanism estimates aggregate eviction behavior/MRCs, not an exact per-key time-to-eviction. |
| Is V2 ready for implementation benchmarking as written? | No-go | Revise V2.0–V2.3 and the implementation guide first. |

## Evidence basis

This review consumed all 59 articles under `Research/ABC`, including the V1 methodology, implementation guides, validation failures, repaired-image mechanism proof, AgentX/Weka pressure experiments, and the Version2 documents.

The code check used the local `experimental/naive-proactive-prefetching` branch. The relevant behavior is:

- request block keys are constructed in `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`;
- the existing admission-prefetch hook is in that scheduler;
- lookup, promotion submission, promotion flushing, store completion, and schedule-end processing are in `vllm/v1/kv_offload/tiering/manager.py`;
- filesystem lookup is asynchronous and may first return retry in `vllm/v1/kv_offload/tiering/async_lookup.py`;
- CPU evictable blocks are held in an `OrderedDict` LRU in `vllm/v1/kv_offload/cpu/policies/lru.py`;
- CPU destination allocation in `vllm/v1/kv_offload/cpu/manager.py` can evict blocks to make room;
- store completion currently cascades newly stored blocks through all configured secondary tiers rather than selecting among Hot/Warm/Cool/Cold placements.

The external research basis was checked against primary sources listed in [Research references](#research-references).

## What is sound

### 1. Admission and workflow events provide information unavailable to reactive LRU

A reactive replacement policy sees past access. An agent runtime can additionally expose future-oriented signals: a request has entered a queue, a tool call has started, a handoff is expected, or a session is likely to resume. TokenCake, KVFlow, and PEEK support the broader premise that workflow state, reuse relationships, and proactive movement can improve cache decisions.

This establishes plausibility, not a performance result. Benefit still depends on lead time, exact residency, reusable prefix length, bandwidth contention, and CPU capacity.

### 2. V1 supports the lead-time mechanism

The repaired-image V1 run proved that queued-request promotion can execute correctly. The AgentX/Weka comparison then showed a large policy-yield change when concurrency increased: useful/attempted rose from 1.16% at concurrency 32 to 15.81% at concurrency 64, while late/promoted fell from 98.50% to 42.39%. That is consistent with the hypothesis that queue lead time is a primary control variable.

It does **not** prove an end-to-end latency benefit: the V1 comparisons were not sufficiently controlled or repeated, and the concurrency-64 treatment did not improve headline TTFT.

### 3. Deterministic policy before ML is the right sequence

The mechanism, accounting, residency contract, event semantics, and cost model must stabilize before training data is trustworthy. A transparent heuristic also makes failures falsifiable and provides a baseline that any learned policy must beat.

### 4. The defensible novelty is narrower than “cache-aware scheduling”

llm-d already performs exact, event-driven, per-pod KV indexing and combines longest-prefix cache locality with load-aware routing. Version2 should therefore claim novelty in **predicted future reuse and deadlines derived from session lifecycle state**, combined with exact residency and load—not in cache awareness alone.

AgentKVShift and RelayCaching address semantic reuse/shift across agent contexts. They strengthen the importance of agent structure but do not by themselves establish the proposed tier-placement controller.

## Blocking design issues and required corrections

### 1. The V2.1 scorer degenerates at admission

The documented admission path marks each candidate as Hot. The proposed score does not incorporate `abc_event` in a way that differentiates candidates. Session identity and AET therefore cannot change the ordering. As written, V2.1 is a residency-checked admission loader, not an event-driven temperature policy.

The unit of selection must also be an **ordered, contiguous prefix bundle**, not an unordered union of block keys. vLLM block hashes are prefix-chained; selecting discontinuous blocks breaks the relationship between reusable prefixes and demand lookup.

Required correction:

- define `B = [k_0, …, k_j]` as an ordered contiguous prefix bundle;
- distinguish event-specific policies and deadlines explicitly;
- use admission-only residency/deadline gating as V2.1;
- defer actual lifecycle-event temperature changes to V2.2.

### 2. Async residency needs a re-driven state machine

Filesystem lookup can return retry while a batched lookup is pending, and the lookup batch is flushed at schedule end. A one-shot admission callback cannot consume the eventual result unless pending candidates are retained and revisited.

Required state machine:

`PENDING_LOOKUP → RESIDENT | ABSENT → GATE → SUBMITTED → READY | LATE | FAILED`

The controller must define ownership, deadlines, cancellation on request completion/preemption, duplicate suppression, and bounded pending state. Lookup pending must never be silently counted as secondary absence.

### 3. “Prefetch never evicts” is false with current CPU allocation

The promotion path prepares writes into the primary CPU tier using the same destination allocation behavior as normal storage. That path may evict LRU blocks. A capacity check taken before submission is not enough because allocations race with demand and other promotions.

At least one of the following is required before speculative promotion:

- a destination API that reserves only currently free blocks and refuses to evict;
- a separately bounded speculative region/budget;
- an atomic allocation preview/reservation with a no-evict flag.

Telemetry must count capacity rejections and any victim evicted by speculative work. If speculative eviction is allowed in a later phase, its expected cost belongs in the decision function.

### 4. AET is misapplied

The AET paper derives population-average eviction time and miss-ratio information from reuse-time distributions. It does not supply an exact, cheap time-to-eviction for an individual cache key. The current CPU LRU is an `OrderedDict`, so deriving an arbitrary key's rank by scan would also be O(n).

Use AET, if retained, as a **global pressure or retention-horizon signal**. Start per-bundle retention with observable TTL, last access, inter-reuse interval, and exact residency. Add rank-based features only after the data structure and cost are explicitly designed.

### 5. The cost gate is dimensionally and causally incomplete

The documented gate mixes fetch and transfer terms, leaves `N(load)` ambiguous, omits eviction cost, and does not account for how much transfer can be hidden behind predicted lead time.

Use a common-unit expected utility for a contiguous prefix bundle `B`:

$$
U(B) =
p_{\mathrm{use}}(B)\,\mathrm{saved\_critical\_path\_ms}(B)
- \Delta Q_{\mathrm{active}}(B)
- \mathbb{E}[C_{\mathrm{eviction}}(B)]
- C_{\mathrm{failure}}(B)
$$

where

$$
\mathrm{saved\_critical\_path\_ms}(B)
=
\max\left(
0,\,
L_{\mathrm{demand\ fetch}}(B)
-
\max(0, L_{\mathrm{prefetch}}(B)-H)
\right)
$$

and `H` is predicted lead time until demand. Every term must be calibrated in milliseconds or converted to an explicitly declared equivalent. V2.1 cannot be independent of V2.0 constants unless it first runs in shadow mode.

### 6. Request-scoped event metadata cannot expose the complete tool window

`kv_transfer_params` arrives with a request. A `tool_call_start` signal is known only after the model emits the tool request, and `tool_call_end` is attached to a later resumed request—too late to exploit most of the external tool interval.

V2.2 needs an out-of-band, session-addressed control API or equivalent runtime integration. The session registry must store ordered, versioned prefix chains and their lifecycle, not a set-union of hashes.

### 7. “Multi-tier placement” exceeds the current control surface

The current manager cascades completed stores to every secondary tier and promotes into the primary tier. That is sufficient to study secondary→CPU movement, but it does not implement independent Hot/Warm/Cool/Cold placement, GPU admission, CPU retention, and secondary persistence.

Those are separate decisions with different ownership and cost. They must be split into later control surfaces rather than implied by V2.1.

### 8. Workload provenance is inconsistent

The Version2 documents reference multiple AgentX dataset/profile dates (`060826`, `061526`), while the V1 evidence uses `062126`. A benchmark cannot be interpreted as a controlled continuation until a single immutable workload revision, seed, prompt construction, and session mapping are declared.

### 9. The attempted denominator can hide lookup misses

If only submitted promotions count as attempted, adding a residency filter can mechanically improve useful/attempted by moving absent keys outside the denominator. That would confuse selection quality with denominator choice.

Count every considered bundle or block exactly once and preserve a terminal partition:

$$
\mathrm{considered}
=
\mathrm{primary\ redundant}
+
\mathrm{secondary\ absent}
+
\mathrm{gate\ reject}
+
\mathrm{capacity\ skip}
+
\mathrm{submitted}
+
\mathrm{lookup\ unresolved}
$$

Report `useful / considered` as the stable policy-yield metric. Also report `useful / submitted`, readiness at first demand, bytes moved, transfer time, evictions, queue delay, TTFT, E2E latency, errors, preemptions, and completed sessions. Record state transitions separately from the terminal accounting partition so asynchronous retries are not double-counted.

## Corrected proposition

> An event- and queue-informed controller can reduce critical-path KV retrieval for reusable, contiguous session prefixes by scheduling residency-verified promotions only when predicted lead time exceeds calibrated transfer time and expected latency benefit exceeds contention and eviction cost.

This proposition is narrower, measurable, and falsifiable. It does not assume that every event is predictive, that secondary residency is known synchronously, or that speculative work is free.

## Revised phase sequence

| Phase | Purpose | Minimum implementation | Exit criterion |
|---|---|---|---|
| V2.0 — Characterization and calibration | Establish whether an actionable window exists | Standardize workload revision; measure queue/tool/handoff lead times, reusable contiguous prefix sizes, secondary residency, batch transfer curves, CPU capacity/eviction effects; run a controlled resident-key microbenchmark; evaluate event/session predictors offline | Declared distributions and constants support at least one feasible policy region, or the proposition is rejected for the tested environment |
| V2.1 — Residency/deadline admission prefetch | Isolate exact-residency and lead-time gating | Contiguous prefix bundles, async lookup state machine, deadline gate, no-evict speculative capacity, stable accounting | Lower late/considered and absent/submitted than V1 without violating correctness or predeclared latency/resource non-inferiority bounds |
| V2.2 — True lifecycle-event prefetch | Test whether early orchestration events add useful horizon | Out-of-band tool/handoff API; versioned session prefix registry; promote the last confirmed reusable prefix during the external-work window | Better ready-at-demand and critical-path savings than admission-only under matched workload and pressure |
| V2.3 — Retention, placement, and routing | Test cross-request and multi-replica policy value | TTL/recency first; AET-like global pressure only after trace validation; combine predicted future reuse with llm-d exact residency/load; treat GPU placement, CPU retention, and secondary persistence separately | Incremental benefit over exact-residency plus queue/load routing, with bounded eviction, bandwidth, and tail-latency cost |

The existing [[01 - Strategy and Re-sequencing]], [[02 - Phased Plan]], and [[03 - Event-Driven Temperature Heuristic Implementation Guide]] remain historical design inputs but require revision to this sequence before implementation begins.

## Hypotheses and falsification criteria

### H1 — Exact residency reduces wasted submissions

**Hypothesis:** asynchronous exact-residency resolution will reduce `load_failed / submitted` relative to V1 blind first-N selection without material scheduler regression.

**Falsified or unsupported if:** lookup completion misses the useful scheduling window, scheduler overhead breaches the predeclared bound, or absent/failed work merely moves outside the accounting denominator.

### H2 — Lead time is the primary benefit condition

**Hypothesis:** useful-ready-at-demand yield and saved critical-path time increase when `H` exceeds calibrated bundle transfer time.

**Falsified or unsupported if:** matched runs show no relationship after controlling for bundle size, residency, queue pressure, and CPU capacity.

### H3 — Lifecycle events add value beyond admission

**Hypothesis:** tool-start or handoff events expose more useful lead time than request admission alone for resumed sessions.

**Falsified or unsupported if:** event precision, reusable-prefix availability, or horizon is insufficient to improve ready-at-demand versus an admission-only controller.

### H4 — The cost gate protects the active workload

**Hypothesis:** the utility/capacity gate holds p95 TTFT and request success within predeclared non-inferiority bounds across low, medium, and high pressure while retaining some useful promotions.

**Falsified or unsupported if:** speculative work causes unbounded eviction, transfer contention, preemption, scheduler cost, or tail-latency regression.

### H5 — Predicted future reuse adds value beyond exact cache-aware routing

**Hypothesis:** in a multi-replica deployment, future-reuse/deadline state improves outcomes beyond llm-d exact prefix residency plus queue/load signals.

**Falsified or unsupported if:** the enhanced scorer does not beat that baseline under matched placement, workload, and capacity.

Numerical acceptance bounds must be declared after V2.0 calibration and before treatment results are inspected.

## Go/no-go decision

**No-go for implementing the current V2.1 guide verbatim.**

Proceed with V2.0 characterization and a guide revision. V2.1 may begin only after these gates are satisfied:

1. the immutable workload revision and session semantics are fixed;
2. the policy unit is an ordered contiguous prefix bundle;
3. the async residency state machine and cancellation rules are specified;
4. speculative destination allocation is non-evicting or explicitly costed and bounded;
5. the lead-time-aware utility function and measured constants are defined;
6. stable considered/submitted/useful accounting is instrumented;
7. admission-only and true lifecycle-event claims are separated;
8. the llm-d exact-residency/load baseline is included where routing novelty is evaluated.

## Research references

- [TokenCake: agent workflow-aware KV cache management](https://arxiv.org/abs/2510.18586)
- [KVFlow: efficient KV cache management for agentic workflows](https://arxiv.org/abs/2507.07400)
- [PEEK: proactive KV movement for agent workloads](https://arxiv.org/abs/2607.02525)
- [Average Eviction Time (USENIX ATC 2016)](https://www.usenix.org/conference/atc16/technical-sessions/presentation/hu)
- [AgentKVShift](https://arxiv.org/abs/2607.21604)
- [RelayCaching](https://arxiv.org/abs/2603.13289)
- [llm-d precise prefix-cache-aware routing](https://github.com/llm-d/llm-d/blob/main/guides/precise-prefix-cache-aware/README.md)
- [llm-d KV-cache management architecture](https://github.com/llm-d/llm-d/blob/main/docs/architecture/advanced/kv-management/README.md)
- [vLLM KV offloading documentation](https://github.com/vllm-project/vllm/blob/main/docs/features/kv_offloading_usage.md)

## Related internal evidence

- [[../Reports/2026-08-18 - Phase 1 admission prefetch repaired-image validation|V1 repaired-image mechanism validation]]
- [[../Reports/2026-08-18 - AgentX Weka admission prefetch first exploratory run|AgentX/Weka concurrency-32 result]]
- [[../Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64|AgentX/Weka concurrency-64 result]]
- [[../Methodology/06 - Deep Speculative Prefetching and Temperature Characterization|Deep speculative prefetching research synthesis]]
- [[00 - Index|Version2 index]]
- [[../00 - Index|ABC project index]]

## Resolution addendum (2026-08-19)

**Status update: the revisions required by this review are complete.** Documents 01–03 were revised on 2026-08-19 and all nine blocking corrections were incorporated. The original review above is preserved unmodified as the audit record; this addendum is the resolution record.

### Resolution of the nine blocking corrections

1. Degenerate admission scorer → selection redefined over ordered contiguous prefix bundles; admission-only residency/deadline gating is V2.1; lifecycle-event temperature deferred to V2.2 (docs 01, 02, 03 §2).
2. Async residency → explicit state machine `PENDING_LOOKUP → RESIDENT | ABSENT → GATE → SUBMITTED → READY | LATE | FAILED` with deadlines, cancellation, duplicate suppression, and bounded pending state (03 §3.2).
3. Non-eviction → `try_reserve_no_evict` reservation consumed by `_initiate_promotion(..., reserved=…)` with exactly one allocation per key; cancellation via `complete_write(success=False)` (03 §3.4).
4. AET → removed as a per-key predictor; retained only as a candidate global pressure signal after trace validation (01, 02 V2.3).
5. Cost gate → common-unit, lead-time-aware utility $U(B)$; shadow mode until V2.0 calibration lands (03 §3.3).
6. Tool-window events → out-of-band session-addressed control API in V2.2; request-scoped metadata limited to admission (02 V2.2, 03 §3.6).
7. Multi-tier overclaim → V2.1 scoped to secondary→CPU promotion; placement split into separate control surfaces (01, 02 V2.3).
8. Workload provenance → pinned `semianalysisai/cc-traces-weka-062126` (01, 02, 03).
9. Denominator → terminal-partition accounting with `useful/considered` as the stable yield metric (02, 03 §3.5).

### Follow-up refinements (second review round, 2026-08-19)

Six further issues were raised after the first revision and incorporated the same day:

1. This document's status became stale → this addendum.
2. Bounded budget ≠ non-eviction → reservation-into-promotion flow specified without double allocation (03 §3.4).
3. Bundle semantics → candidates begin at the primary-residency frontier: $B = [k_m…k_j]$ with $[k_0…k_{m-1}]$ already resident; explicit O(n) selection algorithm (03 §2, §3.2).
4. Mixed denominators/units in V2.1 exit criteria → V1 baseline restated per offload key (`attempted ≡ considered`, `promoted ≡ submitted`); `load_failed/submitted` for submission failure, `secondary_absent/considered` for candidate composition, `late/submitted` for readiness (02 V2.1 exit criteria).
5. V2.2 event-timeline ambiguity → single policy: `tool_call_start` cancels the session's pending prefetch and, when predicted duration $D > L_{\text{prefetch}}$, immediately triggers resume-prefix promotion inside the window; "demotion" means loss of scheduling priority, never active data movement (02 V2.2).
6. Async-loop timing → remaining lead time recomputed at every re-drive; precise `on_schedule_end` ordering specified (03 §3.2, §3.7).

**Verdict after resolution:** the proposition is sound and the phase structure is accepted for implementation planning. V2.0 characterization remains the next step.

## Independent re-validation addendum (2026-08-19, third round)

An independent review of the revised documents 01–03 re-checked the logic, the V1 evidence chain, and — new in this round — the load-bearing code assumptions directly against the local `experimental/naive-proactive-prefetching` tree. **Verdict: the revised proposition and phase structure are confirmed sound; no new blocking issues.** The corrected proposition targets exactly the two failure modes V1 measured (blind residency → 87.08% load-fail at C32; missing lead time → 98.50% late at C32 versus 42.39% at C64), and every correction from the first two rounds is genuinely present in the revised documents.

### Code assumptions verified (measured observations, local tree)

- `CPUOffloadingManager.prepare_store` evicts LRU victims to make room and returns `None` only when eviction cannot free enough (`cpu/manager.py`) — the non-evicting reservation requirement (03 §3.4) is real, not hypothetical.
- In-flight stores are tracked at `ref_cnt = -1` and excluded from evictable accounting (`cpu/manager.py`) — the reservation-is-allocation idiom in 03 §3.4 builds on an existing mechanism.
- `complete_store(success=False)` removes and frees not-yet-ready blocks — the cancellation/release path in 03 §3.4 exists as claimed.
- `AsyncLookupManager.lookup` returns `None` (retry) with batched flush at schedule end — the state-machine requirement (03 §3.2) is genuine; a one-shot admission callback would miss results.
- `_maximal_prefix_lookup` breaks at the first `MISS` — the ordered-contiguous-prefix-bundle constraint is provably necessary; discontinuous prefetch is unusable by the demand path.
- `TieringOffloadingManager.on_schedule_end` runs `_maybe_process_finished_jobs()` first and `_flush_pending_promotions()` later — the §3.7 policy-hook insertion point exists exactly where the guide places it.

### Minor notes (non-blocking, for the V2.0/V2.1 revision)

1. **Same-step lookup consumability is optimistic.** `AsyncLookupManager.flush()` posts the batch to the worker at schedule end and results are drained no earlier than the next step, so a lookup issued at step k resolves at step k+1 at best. 03 §3.7's guarantee "lookup results from this step's flush are consumable in the same step" should be restated as a minimum one-step lookup latency. The design already absorbs this (`H_remaining` recomputation; "lookup delay consumes lead time"), so this is a wording/expectation fix, not a design flaw.
2. **Refresh the numerical exit-criteria anchors from the V2.0 rerun.** The V2.1 exit thresholds in 02 (≤10% vs 37.78%, ≤21% vs 42.39%, >15.81%) are anchored on single-repetition, node-confounded V1 runs. Since V2.0 item 2 reruns the V1 sweep with ≥3 balanced repetitions, the final bounds should be declared against those rerun values, consistent with 02's own rule that bounds are declared before treatment results are inspected.
3. **The tested pressure points may straddle the feasible region.** C32 gave essentially no lead time (mean waiting < 0.25) and C64 is a saturation regime (~91–92% mean GPU KV occupancy, elevated preemptions) where prefetch benefit can be masked by the HBM bottleneck. The intermediate-concurrency cell that 02 lists as optional should be treated as likely necessary for V2.0 lead-time characterization and for locating the H2 benefit region.

The AET characterization in correction 4 was also re-verified against the ATC 2016 kinetic-modeling paper: AET is derived from reuse-time distributions as a population-level quantity, so its removal as a per-key countdown is correct. V2.0 characterization remains the next step.