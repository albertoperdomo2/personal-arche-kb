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
- [[2026-08-14 - Phase 1 queued-request oracle prefetch plan|2026-08-14 — Phase 1 queued-request oracle prefetch plan]] — proposed blind first-N queued-request experiment to isolate the performance value of correctly timed NVMe→CPU promotion.

## Current conclusion

Phase 1 remains open. The CPU-only batch first showed that the hook and counters executed without a secondary tier. The controlled NVMe pair now shows the deeper problem: the secondary tier and reactive NVMe lookup were active, but attempted and skipped prefetch rates were identical at every native 15-second sample, with no promoted/useful/wasted series.

Code inspection confirms the root cause. vLLM hashes are prefix-chained and the filesystem tier is append-like, so a stored later chunk implies its predecessor was also stored. The hook runs only after a resolved terminal miss and selects later keys; under normal operation, that candidate set is necessarily absent. The prefetch helper also collapses asynchronous secondary-tier `RETRY` into a generic skip. No latency difference is attributable to prefetch, and $N$ must not be tuned until the candidate source and deferred lookup handling are redesigned.

## MLflow run registry

- No-offload reference: [c2c2e87883324898995c3ca1639db3b1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/c2c2e87883324898995c3ca1639db3b1?workspace=benchflow)
- 256 GiB CPU-offload control: [5f57165d7d464cee8514645215c526c7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5f57165d7d464cee8514645215c526c7?workspace=benchflow)
- Nominal `prefetch_chunks=100` without secondary tier (rejected): [d5bace21821648ec96bcb7f6efdb3077](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d5bace21821648ec96bcb7f6efdb3077?workspace=benchflow)
- NVMe control, `prefetch_chunks=0`: [988f03995bb745659749110472019c6b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/988f03995bb745659749110472019c6b?workspace=benchflow)
- NVMe nominal prefetch, `prefetch_chunks=100` (rejected for effect/tuning; all candidates skipped): [96d01b33a71f4f1bbb2d55a53a8aaacd](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/96d01b33a71f4f1bbb2d55a53a8aaacd?workspace=benchflow)

## Next experiment

Implement the queued-request oracle PoC: pre-warm target prefixes on persistent NVMe, restart with cold GPU/CPU caches, and blindly promote the first `N=100` keys of a marked waiting request while a cover request occupies the GPU. The experimental path should bypass secondary membership lookup and treat the warmed tier as authoritative, while reporting load failures as oracle violations.

Compare at least five paired `N=0` and `N=100` repetitions. Accept the mechanism only with real `1:fs` promotions, a useful/promoted ratio of at least 0.9, and prefetch completion before target demand. Then evaluate paired target TTFT and aggregate pipeline throughput.