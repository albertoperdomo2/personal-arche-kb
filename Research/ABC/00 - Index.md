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
- [[02 - Phase 1 Naive Prefetch Implementation Guide|02 — Phase 1 Naive Prefetch Implementation Guide (historical)]] — original post-miss N-chunk read-ahead design; its Phase 1 policy was rejected by the NVMe validation and is superseded by guide 04.
- [[03 - Phase 2 Heuristic Prefetch Implementation Guide|03 — Phase 2 Heuristic Prefetch Implementation Guide (tentative)]] — adaptive N controller, feature-based block selection, and sliding-window group support; tentative pending Phase 1 proof.
- [[04 - Phase 1 Queued-Request Oracle Prefetch Implementation Guide|04 — Phase 1 Queued-Request Oracle Prefetch Implementation Guide]] — implementation-ready design for blind first-N promotion at request admission, with request-level warm-up gating, direct NVMe→CPU loads, failure attribution, and tests.

## Reports

- [[2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report|2026-08-10 — Nemotron no-offload versus CPU-offload KV lookup report]]
- [[2026-08-14 - Phase 1 CPU prefetch validation|2026-08-14 — Phase 1 CPU prefetch validation]] — rejected Phase 1 batch: `prefetch_chunks=100` was enabled but every attempted chunk was skipped because no secondary tier was configured.
- [[2026-08-14 - Phase 1 NVMe prefetch validation|2026-08-14 — Phase 1 NVMe prefetch validation]] — mechanism diagnosis with an active NVMe tier: demand lookup worked, but every post-miss prefetch candidate was skipped.
- [[2026-08-14 - Phase 1 queued-request oracle prefetch plan|2026-08-14 — Phase 1 queued-request oracle prefetch plan]] — blind first-N queued-request experiment to isolate the performance value of correctly timed NVMe→CPU promotion.

## Current conclusion

The original Phase 1 post-miss read-ahead policy is closed as rejected. vLLM hashes are prefix-chained and the filesystem tier is append-like, so a stored later chunk normally implies its predecessor was also stored. Selecting later keys after a resolved terminal miss therefore produces candidates that are normally absent. The old helper also collapses an asynchronous filesystem `RETRY` into a generic skip, but correcting that state alone would not rescue the candidate policy.

Phase 1 now means the queued-request oracle proof of concept in guide 04. The new path will build the first `N` keys at request admission, bypass secondary membership lookup, and directly submit assumed-resident NVMe→CPU promotions while the request waits. The benchmark controls residency by construction and uses `kv_transfer_params.abc_admission_prefetch` to disable prefetch during NVMe population and enable it only for measured requests.

The old scheduler hook, `_try_promote()`, and its membership-driven `prefetch()` contract should be removed. The underlying CPU reservation, batched secondary load, completion, reactive fallback, and outcome metrics remain required and should be retained.

## MLflow run registry

- No-offload reference: [c2c2e87883324898995c3ca1639db3b1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/c2c2e87883324898995c3ca1639db3b1?workspace=benchflow)
- 256 GiB CPU-offload control: [5f57165d7d464cee8514645215c526c7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5f57165d7d464cee8514645215c526c7?workspace=benchflow)
- Nominal `prefetch_chunks=100` without secondary tier (rejected): [d5bace21821648ec96bcb7f6efdb3077](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d5bace21821648ec96bcb7f6efdb3077?workspace=benchflow)
- NVMe control, `prefetch_chunks=0`: [988f03995bb745659749110472019c6b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/988f03995bb745659749110472019c6b?workspace=benchflow)
- NVMe nominal prefetch, `prefetch_chunks=100` (rejected for effect/tuning; all candidates skipped): [96d01b33a71f4f1bbb2d55a53a8aaacd](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/96d01b33a71f4f1bbb2d55a53a8aaacd?workspace=benchflow)

## Next experiment

Implement guide 04 on `experimental/naive-proactive-prefetching`:

1. replace `prefetch_chunks` with `admission_prefetch_chunks`;
2. delete the post-miss scheduler hook and membership-probing Phase 1 helper;
3. add the request-gated, assumed-resident admission path from secondary tier 0;
4. add oracle-failure and late-prefetch metrics;
5. mark measured GuideLLM requests and explicitly leave pre-warmup unmarked;
6. compare at least five paired `N=0` and `N=100` repetitions.

Accept the mechanism only with real `1:fs` promotions, zero oracle-load failures, a useful/effective-promoted ratio of at least 0.9, and prefetch completion before target demand for most eligible blocks. Then evaluate paired target TTFT and aggregate pipeline throughput.