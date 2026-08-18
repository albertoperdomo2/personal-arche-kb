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

## Detailed evidence appendices

The 2026-08-10 and 2026-08-17 report directories contain the native-cadence comparison plots and request-level appendices linked from their parent reports.

Return to [[00 - Index|ABC project index]].