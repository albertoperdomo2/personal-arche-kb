---
title: "Version 2 — Phased Plan"
date: "2026-08-19"
type: "experiment-plan"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "active"
supersedes: "2026-08-18 initial version"
revised-after: "[[04 - Theoretical Validation]]"
workload: "semianalysisai/cc-traces-weka-062126 (pinned)"
---

# Version 2 — Phased Plan

Shortest credible path to a demonstrable, deterministic prefetch heuristic with upstreamable shape. Revised 2026-08-19 to incorporate the theoretical validation in [[04 - Theoretical Validation]]: the phase sequence below is the corrected one, V2.1 start is gated, and all accounting uses the terminal partition (`useful/considered`).

**Corrected proposition.** An event- and queue-informed controller can reduce critical-path KV retrieval for reusable, contiguous session prefixes by scheduling residency-verified promotions only when predicted lead time exceeds calibrated transfer time and expected latency benefit exceeds contention and eviction cost.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Version 2 revised timeline (weeks from 2026-08-19)",
  "width": 700,
  "height": 170,
  "background": "white",
  "data": {
    "values": [
      {"phase": "V2.0 Characterization + calibration", "start": 0, "end": 1.5, "track": "bench"},
      {"phase": "V2.1 Residency/deadline admission prefetch", "start": 1.5, "end": 4, "track": "code"},
      {"phase": "V2.2 Lifecycle-event prefetch (out-of-band)", "start": 4, "end": 6.5, "track": "code"},
      {"phase": "V2.3 Retention + placement (vLLM) + RFC", "start": 2, "end": 7, "track": "upstream"}
    ]
  },
  "mark": {"type": "bar", "height": 18},
  "encoding": {
    "y": {"field": "phase", "type": "nominal", "title": null, "axis": {"labelLimit": 340}},
    "x": {"field": "start", "type": "quantitative", "title": "Week", "scale": {"domain": [0, 7]}},
    "x2": {"field": "end"},
    "color": {"field": "track", "type": "nominal", "title": "Track"}
  }
}
```

Figure 1 shows the four scheduled tracks — all single-engine vLLM work. The demonstration milestone is V2.1 at ~week 4. V2.1 start is **gated** (see below); the gate items are design/spec work, not benchmarks, so they overlap V2.0. V2.4 (llm-d scale-out) is intentionally absent from the timeline: it starts only after the single-engine proof exists.

## V2.0 — Characterization and calibration (week 0–1.5)

**Objective.** Establish whether an actionable prefetch window exists, and produce the calibrated constants the utility gate needs. This phase can reject the proposition for the tested environment — that is a feature.

**Method.**

1. **Pin the workload**: immutable revision `semianalysisai/cc-traces-weka-062126`, declared seed, prompt construction, and session mapping. Supersedes the `060826`/`061526` references in older docs.
2. Close V1 with the planned sweep (`N ∈ {0, 25, 50, 100, 200}`, ≥3 balanced interleaved reps, concurrency 32 and 64, node-swapped) — as the V1 closing report and the blind-policy baseline, not a performance chase.
3. Measure the **lead-time distributions**: queue waiting time per admitted request at each concurrency; reusable contiguous prefix sizes per session; secondary residency rates; batch transfer curves (NVMe→CPU promotion latency/throughput vs. batch size); CPU capacity and eviction effects.
4. Run a controlled resident-key microbenchmark: demand fetch vs. prefetched fetch of identical bundles, isolating critical-path savings from policy effects.
5. Evaluate event/session predictors **offline** against the trace (turn boundaries, session revisit structure) before any online claim.

**Exit criteria.** Declared distributions and constants support at least one feasible policy region (lead time H exceeds calibrated transfer time for a non-trivial share of admissions), or the proposition is rejected for the tested environment. Numerical acceptance bounds for H1–H5 (below) are declared here, before treatment results are inspected.

## V2.1 — Residency/deadline admission prefetch (week 1.5–4, gated start)

**Objective.** Isolate exact-residency and lead-time gating at admission. This is the demonstrable heuristic.

**Start gates (all eight, from the validation).**

1. Immutable workload revision and session semantics fixed (V2.0 item 1);
2. policy unit is an ordered contiguous prefix bundle;
3. async residency state machine and cancellation rules specified;
4. speculative destination allocation is non-evicting (or explicitly costed and bounded);
5. lead-time-aware utility function and measured constants defined (shadow mode until V2.0 calibration lands);
6. stable considered/submitted/useful accounting instrumented;
7. admission-only and true lifecycle-event claims separated;
8. single-engine scope confirmed — cluster/routing work is deferred to V2.4 (post-proof).

**Method.** Full build guide in [[03 - Event-Driven Temperature Heuristic Implementation Guide]]. Summary: contiguous prefix bundles; async residency state machine; deadline gate (promote bundle B only when predicted lead time H exceeds calibrated prefetch latency); non-evicting speculative reservation; terminal-partition accounting. Validate on the pinned workload at concurrency 32 and 64: V2.1 vs. V1 N=100 vs. reactive baseline, ≥3 paired repetitions, node-swapped.

**Exit criteria (all required).**

1. `late/considered` and `absent/submitted` strictly lower than the V1 concurrency-64 baseline (42.39% late/promoted, 37.78% load_failed/promoted — restated on the new accounting);
2. `useful/considered` materially higher than V1's `useful/attempted` (15.81%), with the denominator honestly including absent, gate-rejected, capacity-skipped, and unresolved bundles;
3. zero speculative-caused evictions of demand-useful blocks (or bounded and counted, if the bounded-budget variant is chosen);
4. TTFT/E2E within predeclared non-inferiority bounds vs. reactive baseline, with improvement where H is sufficient;
5. no correctness or request-completion failures.

## V2.2 — Lifecycle-event prefetch via out-of-band control (week 4–6.5)

**Objective.** Test whether early orchestration events add useful horizon beyond admission (hypothesis H3).

**Method.**

1. Out-of-band, session-addressed control API (tool-call start/end, handoff signals) — request-scoped `kv_transfer_params` cannot carry these early enough; the API is a V2.2 deliverable, with the AgentX harness or router as the event source.
2. Versioned session prefix registry: ordered, versioned prefix chains per session with lifecycle state — not a set-union of hashes.
3. Promote the last confirmed reusable prefix during the external-work window; demote/stop promoting on tool-call start.

**Exit criteria.** Better ready-at-demand and critical-path savings than the admission-only controller under matched workload and pressure; otherwise H3 is falsified and admission-only remains the claim.

## V2.3 — Retention and placement within vLLM + RFC (week 2–7, parallel)

**Objective.** Complete the single-engine control surfaces and stake the upstream claim. **Scope is vLLM only** — cluster routing is explicitly deferred to V2.4.

**Method.**

1. Retention: TTL/recency/inter-reuse-interval features first; AET-like global pressure only after trace validation.
2. Treat GPU placement, CPU retention, and secondary persistence as separate control surfaces with separate ownership and cost — do not imply a unified multi-tier placer.
3. RFC: design doc (deterministic prefetch policy, residency/deadline gating, non-evicting speculative allocation, pluggable policy idiom) circulated internally by ~week 4, posted to vllm-project by ~week 7.

**Exit criteria.** Each vLLM-side control surface demonstrates its incremental benefit under matched workload and capacity, with bounded eviction, bandwidth, and tail-latency cost; RFC posted to vllm-project.

## V2.4 — Scale-out to llm-d (post-proof, not yet scheduled)

**Objective.** Scale the proven single-engine mechanism to the cluster. **Starts only after V2.1/V2.2 demonstrate the single-engine win.**

**Method (sketch, to be expanded when scheduled).**

1. Export predicted-reuse/deadline summaries from the connector in a form the llm-d Endpoint Picker can consume.
2. Routing: combine predicted future reuse/deadlines with llm-d's existing exact prefix-residency + load signals; the baseline for any novelty claim is llm-d precise prefix-cache-aware routing, not naive load balancing (hypothesis H5).
3. Session-migration prefetch between replicas, building on the two-replica AgentX setup used in the Gemma two-replica runs.
4. llm-d RFC following the vLLM proof.

**Exit criteria.** Incremental benefit over exact-residency plus queue/load routing under matched placement, workload, and capacity, with bounded eviction, bandwidth, and tail-latency cost.

## Hypotheses and falsification criteria

Owned by [[04 - Theoretical Validation]]; acceptance bounds declared in V2.0.

- **H1** — Exact residency reduces wasted submissions (`load_failed/submitted` vs. V1) without scheduler regression.
- **H2** — Lead time is the primary benefit condition: ready-at-demand yield and critical-path savings grow when H exceeds calibrated bundle transfer time.
- **H3** — Lifecycle events add useful horizon beyond admission alone.
- **H4** — The utility/capacity gate protects the active workload (p95 TTFT and success within non-inferiority bounds across pressure levels).
- **H5** — *(V2.4, post-proof)* Predicted future reuse adds value beyond llm-d exact cache-aware routing in multi-replica deployment.

## Dependencies and critical path

- V2.1 start depends on the eight gates (design/spec items overlapping V2.0), and its *performance claims* depend on V2.0 calibration (shadow mode until then).
- V2.2 depends on V2.1's bundle machinery and on the out-of-band control API existing at all (harness work — check feasibility early).
- V2.3's RFC depends on V2.1's design settling, not its results.
- V2.4 (llm-d scale-out) depends on the V2.1/V2.2 single-engine proof; it is intentionally unscheduled.

## Risks

| Risk | Mitigation |
|---|---|
| No feasible policy region found in V2.0 | Reject the proposition for the tested environment and publish — a calibrated negative result is still a win against the crowded literature. |
| Out-of-band control API infeasible in harness/router | V2.2 blocked; admission-only (V2.1) remains the demonstrable claim. Decide feasibility by week 2. |
| Queue lead time too short at concurrency 32 | V1 showed C32 is a low-queue regime; validate at C64 and report both. Never claim wins where no queue exists. |
| Speculative eviction of demand-useful blocks | Non-evicting reservation API; telemetry counts capacity rejections and any speculative-caused eviction (target zero). |
| Accounting denominator drift | Terminal partition enforced in code; `useful/considered` reported alongside `useful/submitted`; transitions logged separately. |
| Scheduler-step latency regression | Bundle scoring O(chunks per admission); budget < 1 ms; monitor `vllm:kv_offload_tiering_lookup_sync_delay_seconds`. |
| Upstream API drift past v0.27.0 | Pin to commit `4bdc8a78`; re-inspect before rebasing. |
