---
title: "ABC KV-cache prefetching — short research brief for feedback"
date: "2026-08-23"
type: "research-brief"
experiment: "Activity-Based KV Cache Tier Placement"
status: "current"
scope: "FP8 Nemotron 253B; vLLM tiered CPU/NVMe offload; GuideLLM and AgentX/Weka"
---

# ABC KV-cache prefetching — short research brief for feedback

## Question

Can vLLM move reusable KV from NVMe toward the GPU **before demand**, hide transfer behind useful work, and improve TTFT or throughput under realistic continuous batching?

## Short answer

**The opportunity is real, and proactive NVMe→CPU movement works mechanically. We have not yet demonstrated a repeatable production benefit.** The main limitation is no longer identifying correct blocks: it is making a request's **complete external working set** ready before first demand without destroying equally valuable CPU-resident KV.

## What we tried

| Strategy | Logic and implementation | Experimental evidence | Decision |
|---|---|---|---|
| Reactive baseline | vLLM scans ordered prompt keys; CPU miss triggers async secondary lookup and NVMe→CPU promotion | External lookup P50 ≈2.2 s; P90 ≈5 s; P99 reached the 10 s histogram ceiling | Opportunity established |
| Post-miss read-ahead | After the first demand MISS, submit the next N request keys | All proactive candidates skipped: prefix-chained hashes mean later keys normally cannot exist after an earlier terminal miss | **Killed** |
| Blind admission first-N | At request admission, assume the first N keys exist on NVMe and submit them directly | Serialized GuideLLM: 25,344/25,344 eventually useful and 92.97% ready in time. AgentX C32: 90.99% redundant, 87.08% load-failed, 98.50% late | Mechanism baseline only |
| Residency/deadline bundles (V2.1) | Start at CPU frontier; verify one contiguous NVMe prefix asynchronously; submit only when queue horizon exceeds fitted transfer cost | Shadow selected 10,832 keys; live initially submitted zero because the 256 GiB CPU cache was physically full | Control plane valid; capacity unresolved |
| Physical reserve + lease | Preserve speculative CPU slots and temporarily protect a 64-chunk bundle | Live promotion worked, but 93.29% was wasted; throughput −61%; mean TTFT +472% vs healthy stock | **Killed as implemented** |
| JIT/demand-safe and continuous batching (V3/V7) | One deadline owner, demand-idle submission, demand-priority I/O, 8-chunk micro-batches | V7: 48.1% of submitted chunks useful, yet throughput −55%, mean TTFT +126%, ITL +180% | **V7 killed** |
| Clean fixed-N | Minimal exact admission plans, full-cache eviction, then first-demand cutoff and deadline ordering | v2 eliminated stale/late work, but 64 chunks covered only 2.49% of the average external prefix; mean TTFT +2.14% | Fixed-N rejected |
| One-owner working set | Stage up to the complete 8,192-chunk prompt set for the earliest request; measure request readiness | 676,388 promoted; 99.55% eventually useful—but only 20/2,638 requests fully ready at first lookup | Negative for current implementation; true oracle still open |

## Evidence 1 — queue lead time changes timing

Figure 1 uses exact profiling totals from the blind first-N AgentX runs. Concurrency 64 created a real waiting population; it reduced lateness and increased useful yield. This supports the overlap intuition, not an end-to-end benefit claim.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Queue pressure improves prefetch timing, but not enough","width":650,"height":260,"data":{"values":[{"observation":"C32 · late/promoted","percent":98.50,"metric":"Late"},{"observation":"C64 · late/promoted","percent":42.39,"metric":"Late"},{"observation":"C32 · useful/attempted","percent":1.16,"metric":"Useful yield"},{"observation":"C64 · useful/attempted","percent":15.81,"metric":"Useful yield"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"observation","type":"nominal","sort":null,"title":"AgentX configuration and ratio","axis":{"labelAngle":-20}},"y":{"field":"percent","type":"quantitative","title":"Share of chunks (%)","scale":{"zero":true,"domain":[0,100]}},"color":{"field":"metric","type":"nominal","title":"Metric","scale":{"scheme":"category10"}},"tooltip":[{"field":"observation","type":"nominal"},{"field":"percent","type":"quantitative","format":".2f","title":"Percent"}]}}
~~~

**Meaning:** admission timing is useful only when a request waits long enough. Queueing is a diagnostic source of horizon, but creating overload is not a production strategy.

## Evidence 2 — useful blocks can coexist with severe service harm

Figure 2 shows the largest observed negative treatments. Values are relative to each report's reactive reference. V2 reserve/lease comparisons used a historical healthy stock run; V7 was one cross-node pair. They are no-go magnitudes, not precise causal effect estimates.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — Major no-go treatments: observed service harm","width":650,"height":280,"data":{"values":[{"observation":"V2 physical reserve · throughput loss","harm_percent":61.0,"metric":"Throughput loss"},{"observation":"V2 physical reserve · TTFT increase","harm_percent":459.5,"metric":"TTFT increase"},{"observation":"V2 retention lease · throughput loss","harm_percent":61.0,"metric":"Throughput loss"},{"observation":"V2 retention lease · TTFT increase","harm_percent":471.6,"metric":"TTFT increase"},{"observation":"V7 continuous batching · throughput loss","harm_percent":55.0,"metric":"Throughput loss"},{"observation":"V7 continuous batching · TTFT increase","harm_percent":125.8,"metric":"TTFT increase"}]},"mark":{"type":"bar"},"encoding":{"y":{"field":"observation","type":"nominal","sort":null,"title":null},"x":{"field":"harm_percent","type":"quantitative","title":"Observed performance harm (%)","scale":{"zero":true}},"color":{"field":"metric","type":"nominal","title":"Metric","scale":{"scheme":"category10"}},"tooltip":[{"field":"observation","type":"nominal"},{"field":"harm_percent","type":"quantitative","format":".1f","title":"Harm (%)"}]}}
~~~

**Meaning:** adding residency checks, reserves, leases, and demand priority fixed real correctness problems, but speculative traffic and cache placement still harmed or failed to improve the serving objective.

## Evidence 3 — the central metric mismatch

The latest working-set treatment is the clearest diagnosis. Figure 3 uses exact run totals or native rolling averages from [treatment 39a70a1b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/39a70a1b52e241bcb48abe5338d56110?workspace=benchflow).

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3 — Correct chunks did not make complete requests ready","width":620,"height":250,"data":{"values":[{"metric":"Promoted chunks eventually useful","percent":99.55},{"metric":"Average working-set coverage","percent":25.46},{"metric":"Requests ready at first lookup","percent":0.76}]},"mark":{"type":"bar","color":"#1f77b4"},"encoding":{"x":{"field":"metric","type":"nominal","sort":null,"title":"Outcome","axis":{"labelAngle":-20}},"y":{"field":"percent","type":"quantitative","title":"Share (%)","scale":{"zero":true,"domain":[0,100]}},"tooltip":[{"field":"metric","type":"nominal"},{"field":"percent","type":"quantitative","format":".2f","title":"Percent"}]}}
~~~

Latest treatment versus control:

| Metric | Delta |
|---|---:|
| Request throughput | +0.16% |
| Mean TTFT | −1.87% |
| p95 TTFT | +1.11% |
| p99 TTFT | +6.51% |
| Requests still deferred at first lookup | **99.24%** |

The high chunk-usefulness ratio answers “did demand eventually touch the copy?” It does **not** answer “did prefetch remove a request-visible stall?” Any missing required chunk still sends the request through reactive promotion.

## What the evidence now supports

- External-tier latency is large enough that a sufficiently early, complete prefetch could matter.
- vLLM already knows exact prompt keys after request arrival; the hard problem is **deadline and working-set readiness**, not classic next-block classification.
- Useful work must be evaluated against transfer contention and CPU eviction regret.
- Ordinary HTTP admission is often too late. The strongest agentic opportunity may be a tool/workflow/session event that occurs before the continuation request.
- Current prefetch stops at CPU; CPU→GPU transfer remains exposed.

## Feedback requested

1. Should the next oracle stage the complete working set only into CPU, or include CPU→GPU readiness?
2. Is the better research contribution scheduler/deadline staging, an agent-runtime→vLLM early-event interface, or joint placement/eviction?
3. Should success mean complete request readiness, maximum contiguous-prefix advancement, or measured stall-time saved?
4. What result should terminate the direction?

## Proposed next gate

Create a **true perfect-residency oracle**: prepopulate and fingerprint the complete source working set, stage it before first lookup, measure full-ready/deferred requests and saved reactive stall, and run replicated same-node crossovers.

Continue only if it produces:

- ≥50% fewer externally deferred requests; and
- ≥5% lower mean/p95 TTFT **or** ≥3% higher throughput; and
- positive benefit after eviction regret, with no ITL or demand-I/O regression.

Full evidence: [[2026-08-21 - Independent research audit and redirection for speculative KV prefetching]], [[Reports/2026-08-23 - Working-set oracle AgentX first comparison]], and [[00 - Index]].