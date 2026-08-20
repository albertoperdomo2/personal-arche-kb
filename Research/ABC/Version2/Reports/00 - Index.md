---
title: "ABC Version 2 reports"
type: "index"
---

# ABC Version 2 reports

## Retention-lease live failure

- [2026-08-20 - V2.1 retention-lease Weka failure](2026-08-20%20-%20V2.1%20retention-lease%20Weka%20failure.md)
  - Verdict: **invalid for benefit analysis; retention implementation rejected.**
  - The v5 image's 512-block physical reserve and one-bundle lease were active: promotions and leases happened. But only 4.71% of integrated outcomes were useful while 93.29% were wasted; demand-critical work reclaimed leased blocks frequently. Service remained in the same severe failure regime as the preceding no-lease run (0.261 requests/s, 48.1 s mean TTFT).
  - Do not run an N/concurrency sweep. First make speculative retention durable enough to produce a materially useful outcome mix without blocking reactive demand.

## Bounded-reserve allocator validation

- [2026-08-20 - V2.1 bounded-reserve live validation](2026-08-20%20-%20V2.1%20bounded-reserve%20live%20validation.md)
  - Interim live-only checkpoint; a matched shadow run is pending.
  - Verdict: corrected V2.1 64/64/64 configuration reached residency-verified live allocation, but submitted zero promotions again. The CPU reserve was logically reported as 64 free blocks while physical free blocks were zero after warmup, so the implementation does not yet preserve physical speculative headroom.

## Initial V2.1 comparison

- [2026-08-19 - V2.1 first five-cell comparison](2026-08-19%20-%20V2.1%20first%20five-cell%20comparison.md)
  - Verdict: the V2.1 decision/control plane and non-evicting safety behavior worked, but the live cell submitted zero speculative blocks because no truly free CPU KV slots remained after warmup.
  - Native 15-second telemetry is split into the report's companion directory.