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
- [[2026-08-14 - Phase 1 NVMe prefetch validation|2026-08-14 — Phase 1 NVMe prefetch validation]] — mechanism diagnosis with an active NVMe tier: demand lookup worked, but every post-miss prefetch candidate was skipped.

## Current conclusion

Phase 1 remains open. The CPU-only batch first showed that the hook and counters executed without a secondary tier. The controlled NVMe pair now shows the deeper problem: the secondary tier and reactive NVMe lookup were active, but attempted and skipped prefetch rates were identical at every native 15-second sample, with no promoted/useful/wasted series.

The first-miss hook selects later cumulative prefix keys after the reactive scan has reached its terminal miss. In the observed workload, none of those candidates existed in the secondary tier. No latency difference is attributable to prefetch, and $N$ must not be tuned until candidate discovery produces real promotions.

## MLflow run registry

- No-offload reference: [c2c2e87883324898995c3ca1639db3b1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/c2c2e87883324898995c3ca1639db3b1?workspace=benchflow)
- 256 GiB CPU-offload control: [5f57165d7d464cee8514645215c526c7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5f57165d7d464cee8514645215c526c7?workspace=benchflow)
- Nominal `prefetch_chunks=100` without secondary tier (rejected): [d5bace21821648ec96bcb7f6efdb3077](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d5bace21821648ec96bcb7f6efdb3077?workspace=benchflow)
- NVMe control, `prefetch_chunks=0`: [988f03995bb745659749110472019c6b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/988f03995bb745659749110472019c6b?workspace=benchflow)
- NVMe nominal prefetch, `prefetch_chunks=100` (rejected for effect/tuning; all candidates skipped): [96d01b33a71f4f1bbb2d55a53a8aaacd](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/96d01b33a71f4f1bbb2d55a53a8aaacd?workspace=benchflow)

## Next experiment

Correct the Phase 1 trigger/candidate construction so it discovers keys positively known to exist in the secondary tier before the terminal prefix miss. Keep `prefetch_chunks=100` for the first post-fix plumbing validation to maximize signal while isolating the code change. Require nonzero promoted and resolved useful/wasted samples before comparing latency. Only after that gate passes, run at least three paired repetitions across `N ∈ {0, 32, 64, 100, 128, 256}`.