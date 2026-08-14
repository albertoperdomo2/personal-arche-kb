---
title: "ABC KV lookup experiments"
date: "2026-08-14"
type: "research-index"
experiment: "ABC CPU offload KV lookup comparison"
status: "active"
---

# ABC

## Definition

- [[01 - Experiment Definition|01 — Experiment Definition]] — problem statement, proposed end-state framework, and the four-phase path from reactive fetching to speculative prefetching.
- [[02 - Phase 1 Naive Prefetch Implementation Guide|02 — Phase 1 Naive Prefetch Implementation Guide]] — step-by-step implementation of the toy N-chunk read-ahead prefetcher in `vllm/v1/kv_offload`, grounded in the vLLM `main` codebase.
- [[03 - Phase 2 Heuristic Prefetch Implementation Guide|03 — Phase 2 Heuristic Prefetch Implementation Guide (tentative)]] — adaptive N controller, feature-based block selection, and sliding-window group support; tentative pending Phase 1 sweep results.

## Reports

- [[2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report|2026-08-10 — Nemotron no-offload versus CPU-offload KV lookup report]]
- [[2026-08-14 - Phase 1 CPU prefetch validation|2026-08-14 — Phase 1 CPU prefetch validation]] — rejected Phase 1 batch: `prefetch_chunks=100` was enabled but every attempted chunk was skipped because no secondary tier was configured.

## Current conclusion

Phase 1 remains open. The 2026-08-14 batch confirms the scheduler prefetch hook and telemetry executed, but it does not confirm working prefetch: attempted and skipped rates were identical at every native 15-second sample, with no promoted/useful/wasted series. The rendered deployment had a 256 GiB CPU primary tier and no `secondary_tiers`, while the toy can only promote secondary→CPU.

The small latency differences between the 256 GiB control and nominal prefetch profile are not attributable to prefetch and must not be used to tune $N$.

## MLflow run registry

- No-offload reference: [c2c2e87883324898995c3ca1639db3b1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/c2c2e87883324898995c3ca1639db3b1?workspace=benchflow)
- 256 GiB CPU-offload control: [5f57165d7d464cee8514645215c526c7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5f57165d7d464cee8514645215c526c7?workspace=benchflow)
- Nominal `prefetch_chunks=100` (rejected for mechanism conclusions): [d5bace21821648ec96bcb7f6efdb3077](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d5bace21821648ec96bcb7f6efdb3077?workspace=benchflow)

## Next experiment

Add and populate a real secondary tier, then repeat the plumbing validation with `prefetch_chunks=100` so the configuration fix is isolated. Require nonzero promoted and resolved useful/wasted samples before comparing latency. After that gate passes, run at least three paired repetitions across `N ∈ {0, 32, 64, 100, 128, 256}`.