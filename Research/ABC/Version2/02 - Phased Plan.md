---
title: "Version 2 — Phased Plan"
date: "2026-08-18"
type: "experiment-plan"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "active"
---

# Version 2 — Phased Plan

Shortest path to a demonstrable, complex, event-driven prefetch heuristic with upstreamable shape. Four tracks, two of which run in parallel from day one. The critical path to the demonstrable heuristic is **V2.1 (~3 weeks)**; the full proposition (heuristic + cost gate + EPP export + RFC) lands in **~6 weeks**.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Version 2 compressed timeline (weeks from 2026-08-18)",
  "width": 700,
  "height": 170,
  "background": "white",
  "data": {
    "values": [
      {"phase": "V2.0 Close V1 + cost curve", "start": 0, "end": 1, "track": "bench"},
      {"phase": "V2.1 Temperature-gated admission prefetch", "start": 0.5, "end": 3, "track": "code"},
      {"phase": "V2.2 Cost gate + tool-call events", "start": 3, "end": 5, "track": "code"},
      {"phase": "V2.3 EPP export + upstream RFC", "start": 2, "end": 6, "track": "upstream"}
    ]
  },
  "mark": {"type": "bar", "height": 18},
  "encoding": {
    "y": {"field": "phase", "type": "nominal", "title": null, "axis": {"labelLimit": 320}},
    "x": {"field": "start", "type": "quantitative", "title": "Week", "scale": {"domain": [0, 6]}},
    "x2": {"field": "end"},
    "color": {"field": "track", "type": "nominal", "title": "Track"}
  }
}
```

Figure 1 shows the four tracks. V2.0 is benchmark operations only and never blocks code; V2.1 starts as soon as the branch is cut and is the demonstration milestone; V2.3's RFC is drafted early deliberately — in upstream OSS the RFC itself is the claim.

## V2.0 — Close V1 and extract the cost curve (week 0–1, bench ops, parallel)

**Objective.** Discharge the remaining V1 obligation and convert it into V2 inputs. Not a performance chase.

**Method.**

1. Run the planned sweep `N ∈ {0, 25, 50, 100, 200}`, ≥3 balanced/interleaved repetitions, concurrency 32 (low pressure) and 64 (queue pressure), swapping the `…-6kl5z` / `…-mt46x` node assignments to break the node/treatment confound.
2. From the sweep, extract the **transfer-cost constants** the V2 cost gate needs: per-chunk NVMe→CPU promotion latency distribution, CPU→GPU load latency, promotion throughput vs. N, and the redundancy/load-failure curves as a function of N.
3. Publish the sweep as the V1 closing report; record the blind-first-N ceiling as the documented motivation for V2 selection.

**Exit criteria.** Cost curve published; V1 formally closed with mechanism accepted; no further blind-N tuning.

## V2.1 — Temperature-gated admission prefetch (week 0.5–3, the demonstrable heuristic)

**Objective.** Replace blind first-N with scored, residency-checked, cost-gated chunk selection at admission time, and demonstrate the benefit on AgentX Weka at queue pressure.

**Method.** Full build guide in [[03 - Event-Driven Temperature Heuristic Implementation Guide]]. Summary:

1. New `temperature.py` module in `vllm/v1/kv_offload/tiering/`: `SessionRegistry`, `AETTracker`, `TemperatureScorer`, `CostGate`.
2. Hook `TieringOffloadingManager.on_new_request` / `on_schedule_end` to score the admitted request's full (already-known) offload key list and submit promotions through the **existing** `_initiate_promotion` / `_flush_pending_promotions` machinery.
3. Verify residency through the real per-tier `lookup` (async where the tier provides `AsyncLookupManager`) instead of V1's assume-resident bypass.
4. Event channel: `kv_transfer_params` (`abc_session_id`, `abc_turn`, `abc_event`, `abc_prefetch`), consistent with the V1 `abc_admission_prefetch` convention.
5. Validate on AgentX Weka at concurrency 32 and 64: V2 heuristic vs. V1 N=100 vs. reactive baseline, 3 paired repetitions, node-swapped.

**Exit criteria (all required).**

1. useful/attempted ≥ 3× the V1 concurrency-64 value (15.81%) — i.e. ≥ ~47%;
2. late/promoted ≤ half the V1 concurrency-64 value (42.39%) — i.e. ≤ ~21%;
3. load_failed/promoted ≤ 10% (vs. 37.78% in V1 at concurrency 64) — the residency check must work;
4. repeatable TTFT improvement over the reactive baseline at concurrency 64 (mean ± CI over ≥3 paired reps, node-balanced), with p95 no worse than baseline;
5. no correctness or request-completion failures.

## V2.2 — Cost gate dynamics + tool-call events (week 3–5)

**Objective.** Make the gate load-aware and exploit the agentic execution loop beyond admission.

**Method.**

1. Dynamic $N$ in $\text{Benefit} > N \times \text{Cost}$ from live queue depth, waiting-request count, and transfer telemetry (the scheduler already exposes waiting depth; tier job queues are visible in `TieringOffloadingManager`).
2. `tool_call_start` event: mark the session's chunks idle; stop promoting them; let AET-driven demotion cascade them toward secondary tiers, freeing primary capacity for active sessions.
3. `tool_call_end` / expected-resume event: pre-position the session's working set NVMe→CPU during the tool window (the multi-hop promotion chain from the synthesis doc), so admission only needs the fast CPU→GPU hop.
4. AET-driven graceful demotion: blocks whose AET trajectory predicts imminent eviction are cascaded asynchronously before hard capacity pressure forces synchronous eviction.

**Exit criteria.** p95 TTFT never worse than reactive baseline under pressure (harm prevention demonstrated); measurable primary-tier capacity relief from tool-call demotion; multi-hop pre-positioning shown to convert NVMe-resident hits into CPU-resident hits before admission.

## V2.3 — EPP export + upstream RFC (week 2–6, parallel)

**Objective.** Stake the claim and wire the routing story.

**Method.**

1. Export temperature summaries from the connector (via `get_kv_connector_stats` / KV cache events) in a form the llm-d Endpoint Picker can consume.
2. Prototype the EPP composite scorer (queue depth + prefix match + resident temperature) against the two-replica AgentX setup used in the Gemma two-replica runs.
3. Draft the RFC (design doc: event-driven temperature, cost gate, multi-tier placement, EPP export) and circulate internally by week 4; post upstream to vllm-project and llm-d by week 6.

**Exit criteria.** RFC posted; EPP routing demonstrably influenced by exported temperature in a ≥2-replica bench.

## Dependencies and critical path

- V2.1 depends only on the local V1 tree state (repaired manager property, event channel) — not on V2.0.
- V2.2 depends on V2.1's scorer and on V2.0's cost constants (for gate calibration; placeholder constants from the repaired-image run suffice until then).
- V2.3's RFC depends on V2.1's design being settled, not on its results; the EPP bench depends on V2.1 code.

## Risks

| Risk | Mitigation |
|---|---|
| AgentX harness cannot emit tool-call events | V2.1 needs only admission events, which the harness already provides; tool-call events are V2.2. Check trace metadata for turn boundaries before V2.2. |
| Queue lead time too short at concurrency 32 | V1 showed C32 is a low-queue regime; validate at C64 and report both. Do not claim wins where no queue exists. |
| Primary-tier capacity contention from prefetch | `prepare_write` returns `None` when full — count as `capacity_skip`, never block; the cost gate throttles admission of prefetch work. |
| Scheduler-step latency regression from scoring | Scorer must be O(chunks per request); budget < 1 ms per admission; measure with the existing `LOOKUP_SYNC_DELAY` histogram. |
| Upstream API drift past v0.27.0 | Pin the guide to commit `4bdc8a78`; re-inspect before rebasing. |
