---
title: "ABC Experiment Reports"
date: "2026-08-18"
type: "research-reports-index"
experiment: "ABC"
status: "active"
---

# ABC Experiment Reports

Executed experiments and their evidence live here. Failed and inconclusive runs are retained alongside accepted validations.

## Reports

- [[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report|2026-08-10 — Nemotron no-offload versus CPU-offload KV lookup]] — baseline comparison of no offload and CPU offload.
- [[Reports/2026-08-14 - Phase 1 CPU prefetch validation|2026-08-14 — Phase 1 CPU prefetch validation]] — rejected: prefetch enabled without a secondary tier.
- [[Reports/2026-08-14 - Phase 1 NVMe prefetch validation|2026-08-14 — Phase 1 NVMe prefetch validation]] — rejected original post-miss policy: all continuation candidates skipped.
- [[Reports/2026-08-17 - Phase 1 admission prefetch first execution report|2026-08-17 — Phase 1 admission prefetch first execution]] — invalid: configuration reached a private manager field but not the scheduler-visible property.
- [[Reports/2026-08-18 - Phase 1 admission prefetch repaired-image validation|2026-08-18 — Phase 1 admission prefetch repaired-image validation]] — mechanism accepted: 25,344 promotions, 100% eventually useful, 92.97% ready at first demand; performance still provisional.
- [[Reports/2026-08-18 - AgentX Weka admission prefetch first exploratory run|2026-08-18 — AgentX Weka admission prefetch first exploratory run]] — concurrency-32 mechanism executed but N=100 was ineffective: 90.99% redundant, 87.08% of submissions load-failed, 98.50% late, and no positive latency signal.
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64|2026-08-18 — AgentX Weka admission prefetch at concurrency 64]] — queue-sensitivity supported: useful/attempted rose to 15.81% and late/promoted fell to 42.39%; performance remained mixed and inconclusive.

- [[Reports/2026-08-21 - Clean-prefetch v1 AgentX first comparison|2026-08-21 — Clean-prefetch v1 AgentX first comparison]] — conditionally valid mechanism result: full-cache admission worked with no failures or observed regret, but 98.44% of promotions were late and performance was neutral; concurrent cells and incomplete artifacts prevent a benefit claim.
- [[Reports/2026-08-22 - Clean-prefetch v1 AgentX concurrency 64 comparison|2026-08-22 — Clean-prefetch v1 AgentX concurrency-64 comparison]] — natural queueing reduced late/useful to 44.67%, but the 64-chunk footprint saturated, 256 promotions were wasted, and 586/966 CPU victims incurred eviction regret; performance is mixed and inconclusive.
- [[Reports/2026-08-22 - Clean-prefetch v1 repeat and attempted v2 invalidation|2026-08-22 — Clean-prefetch v1 repeat and attempted v2 invalidation]] — both cells accidentally reused the exact old v1 digest; invalid for the cutoff/order fix, but a useful repeat showing a 740-second ready-delay p90 and eviction regret above useful outcomes.
- [[Reports/2026-08-22 - Clean-prefetch v2 AgentX concurrency 64 comparison|2026-08-22 — Clean-prefetch v2 AgentX concurrency-64 comparison]] — valid negative policy result: v2 eliminated stale and late work, but 64 chunks covered only 2.49% of the average external prefix, 57.5% of evictions were regretted, and TTFT/throughput did not improve.

- [[Reports/2026-08-23 - Working-set oracle AgentX first comparison|2026-08-23 — Working-set oracle AgentX first comparison]] — mechanism active but the intended oracle condition failed: only 20/2,638 intents were fully ready at first lookup; performance was mixed and near-neutral.

- [[Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration|2026-08-25 — COSTAR Experiment 0 AgentX oracle-corpus calibration]] — conditionally valid: the C32 trace passed 2.24M-event lifecycle, transfer, source, and CPU-capacity validation; the manually salvaged 6.96 GB C64 trace is present but awaits source-side certification because MLflow proxy retrieval times out.

## Detailed evidence appendices

The 2026-08-10, 2026-08-17, and both AgentX Weka report directories contain native-cadence mechanism, scheduler, latency, storage, transfer, and saturation appendices linked from their parent reports.

Return to [[00 - Index|ABC project index]].