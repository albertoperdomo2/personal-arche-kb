---
title: "ABC KV lookup experiments"
date: "2026-08-17"
type: "research-index"
experiment: "ABC CPU offload KV lookup comparison"
status: "active"
---

# ABC

## Definition

- [[01 - Experiment Definition|01 — Experiment Definition]] — problem statement, proposed end-state framework, and the four-phase path from reactive fetching to speculative prefetching.
- [[02 - Phase 1 Naive Prefetch Implementation Guide|02 — Phase 1 Naive Prefetch Implementation Guide (historical)]] — original post-miss N-chunk read-ahead design; its Phase 1 policy was rejected by the NVMe validation and is superseded by guide 04.
- [[03 - Phase 2 Heuristic Prefetch Implementation Guide|03 — Phase 2 Heuristic Prefetch Implementation Guide (tentative)]] — adaptive N controller, feature-based block selection, and sliding-window group support; tentative pending Phase 1 proof.
- [[04 - Phase 1 Queued-Request Oracle Prefetch Implementation Guide|04 — Phase 1 Queued-Request Oracle Prefetch Implementation Guide]] — implementation design for blind first-N promotion at request admission, with request-level warm-up gating, direct NVMe→CPU loads, failure attribution, and tests.

## Reports

- [[2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report|2026-08-10 — Nemotron no-offload versus CPU-offload KV lookup report]]
- [[2026-08-14 - Phase 1 CPU prefetch validation|2026-08-14 — Phase 1 CPU prefetch validation]] — rejected Phase 1 batch: `prefetch_chunks=100` was enabled but every attempted chunk was skipped because no secondary tier was configured.
- [[2026-08-14 - Phase 1 NVMe prefetch validation|2026-08-14 — Phase 1 NVMe prefetch validation]] — mechanism diagnosis with an active NVMe tier: demand lookup worked, but every post-miss prefetch candidate was skipped.
- [[2026-08-14 - Phase 1 queued-request oracle prefetch plan|2026-08-14 — Phase 1 queued-request oracle prefetch plan]] — blind first-N queued-request experiment to isolate the performance value of correctly timed NVMe→CPU promotion.
- [[2026-08-17 - Phase 1 admission prefetch first execution report|2026-08-17 — Phase 1 admission prefetch first execution report]] — invalid/inconclusive first execution: N=100 was rendered, but a manager/scheduler attribute mismatch reduced the scheduler-visible limit to zero, so no proactive work ran. Includes native request, offload, queue, transfer, and NVMe evidence.

## Current conclusion

The original Phase 1 post-miss read-ahead policy remains closed as rejected. vLLM hashes are prefix-chained and the filesystem tier is append-like, so a stored later chunk normally implies its predecessor was also stored. Selecting later keys after a resolved terminal miss therefore produces candidates that are normally absent.

Phase 1 now means the queued-request oracle proof of concept in guide 04: build the first `N` keys at request admission, bypass secondary membership lookup, and directly submit assumed-resident NVMe→CPU promotions while the request waits. The benchmark controls residency by construction and uses `kv_transfer_params.abc_admission_prefetch` to disable prefetch during NVMe population and enable it only for measured requests.

The first live execution on 2026-08-17 did **not** exercise that mechanism. The manager stored the parsed value as `_admission_prefetch_chunks`, while the scheduler read `manager.admission_prefetch_chunks` with a zero fallback. All nine prefetch metric queries were empty in the N=100 cell. The performance result is therefore invalid/inconclusive for proactive prefetch; the slower treatment aggregate is run variability between effectively non-prefetching cells.

The deployment and workload scaffolding otherwise worked: all three cells completed 256/256 requests without errors, both custom cells used the same immutable image digest, warm-up sent the request gate as false and measurement as true, the server rendered N=0 versus N=100 correctly, one sequence ran with roughly 6–7 waiting, and normal reactive NVMe offload remained active.

## MLflow run registry

- No-offload reference: [c2c2e87883324898995c3ca1639db3b1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/c2c2e87883324898995c3ca1639db3b1?workspace=benchflow)
- 256 GiB CPU-offload control: [5f57165d7d464cee8514645215c526c7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5f57165d7d464cee8514645215c526c7?workspace=benchflow)
- Nominal `prefetch_chunks=100` without secondary tier (rejected): [d5bace21821648ec96bcb7f6efdb3077](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d5bace21821648ec96bcb7f6efdb3077?workspace=benchflow)
- NVMe control, `prefetch_chunks=0`: [988f03995bb745659749110472019c6b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/988f03995bb745659749110472019c6b?workspace=benchflow)
- NVMe nominal prefetch, `prefetch_chunks=100` (rejected for effect/tuning; all candidates skipped): [96d01b33a71f4f1bbb2d55a53a8aaacd](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/96d01b33a71f4f1bbb2d55a53a8aaacd?workspace=benchflow)
- Admission-prefetch official-image control: [23b7f315a6a54c08b484b113037abccc](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/23b7f315a6a54c08b484b113037abccc?workspace=benchflow)
- Admission-prefetch custom-image N=0 control: [3ee22e3ae07144039b83d9e6b8dfcbf0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/3ee22e3ae07144039b83d9e6b8dfcbf0?workspace=benchflow)
- Admission-prefetch configured N=100 treatment (invalid; scheduler observed zero): [b6bce02143a0431baa9935731cbe8b23](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/b6bce02143a0431baa9935731cbe8b23?workspace=benchflow)

## Next experiment

Repair and prove real configuration wiring before another full benchmark:

1. expose a read-only `TieringOffloadingManager.admission_prefetch_chunks` property returning the private field;
2. add a real spec→manager→scheduler regression test so a mocked public field cannot hide the mismatch;
3. rebuild under a new immutable image tag/digest;
4. run an 8–16 request live smoke test and require nonzero `attempted{tier="1:fs"}`, the attempted partition identity, zero load failures, and no old aggregate `tier="prefetch"` label;
5. eliminate measured-phase `_swap_blocks_kernel` JIT;
6. verify the actual accelerator SKU and pin or balance node placement;
7. only then compare at least five paired custom-image N=0 and N=100 repetitions with measured request flag true and warm-up flag false in both cells.

Accept the mechanism only with real `1:fs` promotions, zero oracle-load failures, useful/effective-promoted of at least 0.9, and completion before target demand for most eligible blocks. Then evaluate paired target TTFT and aggregate pipeline throughput.