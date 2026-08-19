---
title: "ABC KV lookup experiments"
date: "2026-08-19"
type: "research-index"
experiment: "ABC CPU offload KV lookup comparison"
status: "active"
---

# ABC

## Navigation

- [[Reports/00 - Index|Experiment Reports]] — executed benchmark runs, validations, failures, plots, and conclusions.
- [[Methodology/00 - Index|Methodology and Implementation]] — experiment definitions, plans, implementation guides, and design discussions.
- [[Version2/00 - Index|Version 2 — Proactive Speculative Prefetching]] — deterministic, event- and queue-informed program. The first five-cell V2.1 run validates the control plane and safety behavior but found zero live submissions because no truly free CPU KV slots remained.

## Methodology and implementation

- [[Methodology/01 - Experiment Definition|01 — Experiment Definition]] — problem statement, proposed end-state framework, and the four-phase path from reactive fetching to speculative prefetching.
- [[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide|02 — Phase 1 Naive Prefetch Implementation Guide (historical)]] — rejected post-miss N-chunk read-ahead design, retained as a split historical guide.
- [[Methodology/03 - Phase 2 Heuristic Prefetch Implementation Guide|03 — Phase 2 Heuristic Prefetch Implementation Guide (tentative)]] — adaptive N controller, feature-based block selection, and sliding-window group support.
- [[Methodology/04 - Phase 1 Queued-Request Oracle Prefetch Implementation Guide|04 — Phase 1 Queued-Request Oracle Prefetch Implementation Guide]] — current admission-time, assume-resident implementation tutorial.
- [[Methodology/05 - Initial versus Admission-Time Proactive Prefetching|05 — Initial versus Admission-Time Proactive Prefetching]] — end-to-end explanation of both designs, why the first failed, and how the current mechanism works.
- [[Methodology/2026-08-14 - Phase 1 queued-request oracle prefetch plan|2026-08-14 — Phase 1 queued-request oracle prefetch plan]] — controlled experiment plan for blind first-N queued-request promotion.

## Experiment reports

- [[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report|2026-08-10 — Nemotron no-offload versus CPU-offload KV lookup report]]
- [[Reports/2026-08-14 - Phase 1 CPU prefetch validation|2026-08-14 — Phase 1 CPU prefetch validation]] — rejected: no secondary tier was configured.
- [[Reports/2026-08-14 - Phase 1 NVMe prefetch validation|2026-08-14 — Phase 1 NVMe prefetch validation]] — rejected original post-miss candidate policy.
- [[Reports/2026-08-17 - Phase 1 admission prefetch first execution report|2026-08-17 — Phase 1 admission prefetch first execution report]] — invalid/inconclusive due manager/scheduler wiring mismatch and stale-image repeats.
- [[Reports/2026-08-18 - Phase 1 admission prefetch repaired-image validation|2026-08-18 — Phase 1 admission prefetch repaired-image validation]] — mechanism accepted; performance remains provisional.
- [[Reports/2026-08-18 - AgentX Weka admission prefetch first exploratory run|2026-08-18 — AgentX Weka admission prefetch first exploratory run]] — valid negative policy result at concurrency 32: first-N prefetch ran, but N=100 was mostly redundant, failed, and late.
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64|2026-08-18 — AgentX Weka admission prefetch at concurrency 64]] — Phase 1 queue-sensitivity supported: more waiting sharply increased useful yield and reduced lateness; performance remains inconclusive.
- [[Version2/Reports/2026-08-19 - V2.1 first five-cell comparison|2026-08-19 — V2.1 first five-cell comparison]] — control plane and non-evicting safety validated; live data plane blocked by zero truly free CPU KV slots.

## Current conclusion

The original Phase 1 post-miss read-ahead policy remains closed as rejected. vLLM hashes are prefix-chained and the filesystem tier is append-like, so a stored later chunk normally implies its predecessor was also stored. Selecting later keys after a resolved terminal miss therefore produces candidates that are normally absent.

Phase 1 now means the queued-request oracle proof of concept in guide 04: build the first `N` keys at request admission, bypass secondary membership lookup, and directly submit assumed-resident NVMe→CPU promotions while the request waits. The benchmark controls residency by construction and uses `kv_transfer_params.abc_admission_prefetch` to disable prefetch during NVMe population and enable it only for measured requests.

The first live execution on 2026-08-17 did **not** exercise that mechanism. The manager stored the parsed value as `_admission_prefetch_chunks`, while the scheduler read `manager.admission_prefetch_chunks` with a zero fallback. All nine prefetch metric queries were empty in the N=100 cell. The performance result is therefore invalid/inconclusive for proactive prefetch; the slower treatment aggregate is run variability between effectively non-prefetching cells. The local vLLM working tree now exposes a read-only manager property matching the scheduler contract, has a real-manager scheduler regression test, and passes the focused tiering and admission/lookup suites. No corrected image or live mechanism result exists yet.

A repeat on 2026-08-18 also did **not** contain the repair: both nominal N=0 and N=100 pods resolved the old `v0.27.0-prefetch-p1` digest `32a580...`. The N=100 cell again had no prefetch series. Its apparent p95 TTFT improvement is non-evidence because no proactive work ran, only one repetition exists, the cells used different nodes, and the nominal control completed only 255/256 requests. No corrected image or live mechanism result exists yet.

The repaired-image execution later on 2026-08-18 **passed the Phase 1 mechanism proof**. Both cells used digest `097cffbd...`; N=100 attempted 25,600 blocks, promoted 25,344, classified 256 redundant, and eventually used all 25,344 promotions. Exactly 1,782 promotions (7.03%) were late at first demand, so 92.97% were ready in time. There were no skips, waste, tracking overflow, or load failures. The observed TTFT tail was better, but performance remains inconclusive because the N=0 control again completed only 255/256 requests, nodes differed, JIT occurred during measurement, and only one pair exists.

The deployment and workload scaffolding otherwise worked: all three cells completed 256/256 requests without errors, both custom cells used the same immutable image digest, warm-up sent the request gate as false and measurement as true, the server rendered N=0 versus N=100 correctly, one sequence ran with roughly 6–7 waiting, and normal reactive NVMe offload remained active.

The first AgentX Weka exploration at concurrency 32 reached a low-queue regime. Both cells completed 863 profiling requests, but N=100 attempted 85,010 blocks: 90.99% were redundant in CPU, 87.08% of the 7,654 submitted NVMe promotions load-failed, only 990 became useful, and 98.50% were late at first demand. Mean waiting depth was below 0.25, so the trace provided little admission lead time. The N=100 latency aggregate was not better.

The concurrency-64 follow-up created the missing pressure: N=100 averaged 6.22 waiting requests and recorded 19,508 useful blocks. Relative to concurrency 32, useful/attempted rose from 1.16% to 15.81%, late/promoted fell from 98.50% to 42.39%, and load_failed/promoted fell from 87.08% to 37.78%. This supports the Phase 1 wiring and lead-time intuition. The performance comparison remains inconclusive and not positive overall: mean/p95 TTFT were 3.19%/10.44% higher, median TTFT and ITL improved, the nodes differed, request counts drifted by four, and both cells were heavily pressured.

## Version 2 theoretical validation checkpoint — 2026-08-19

[[Version2/04 - Theoretical Validation|Version2 theoretical validation]] reviewed all ABC records, the local V1 implementation path, and the primary research basis. The broad proposition is **conditionally valid**: workflow events and queue lead time can justify proactive movement of reusable KV, and deterministic policy should precede ML. The current V2.1 documents are **not implementation-ready**.

The no-go issues are the uniform-Hot admission scorer, missing ordered contiguous-prefix semantics, missing async-residency state machine, speculative CPU allocation that can evict, per-key misuse of AET, an incomplete cost gate, insufficient tool-window signaling, and unstable policy accounting. The corrected proposition is to promote residency-verified contiguous session prefixes only when predicted lead time can hide calibrated transfer latency and expected critical-path benefit exceeds contention and eviction cost.

The next Version2 action is V2.0 characterization/calibration, followed by revision of [[Version2/01 - Strategy and Re-sequencing|01]], [[Version2/02 - Phased Plan|02]], and [[Version2/03 - Event-Driven Temperature Heuristic Implementation Guide|03]]. Implementation of V2.1 is gated on that revision.

## Version 2 first execution checkpoint — 2026-08-19

The first five-cell V2.1 batch produced a partial mechanism success. Shadow mode found 171 gate-approved bundles / 10,832 keys. Live mode submitted zero keys: 33 resolved bundles became primary-redundant before manager submission and 132 were refused on their first missing key by the non-evicting allocator. The per-key terminal partition balanced exactly in shadow and live, disabled cells exposed no V2 metrics, and the deadline/transfer model calibrated correctly.

This is not a live-prefetch performance proof. It is evidence that V2.1's control plane and safety behavior work and that the next blocker is explicit speculative headroom. V1 N=100 remains only a negative control: it promoted 18,144 keys, failed 5,739, wasted 10,751, used 1,641, logged 67 missing-file jobs during profiling, and collapsed throughput.

The next implementation must reserve a bounded speculative block budget, split capacity reasons, and expose true allocated/free/evictable/speculative CPU block gauges. Do not restore unrestricted eviction.

## MLflow run registry

- No-offload reference: [c2c2e87883324898995c3ca1639db3b1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/c2c2e87883324898995c3ca1639db3b1?workspace=benchflow)
- 256 GiB CPU-offload control: [5f57165d7d464cee8514645215c526c7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5f57165d7d464cee8514645215c526c7?workspace=benchflow)
- Nominal `prefetch_chunks=100` without secondary tier (rejected): [d5bace21821648ec96bcb7f6efdb3077](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d5bace21821648ec96bcb7f6efdb3077?workspace=benchflow)
- NVMe control, `prefetch_chunks=0`: [988f03995bb745659749110472019c6b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/988f03995bb745659749110472019c6b?workspace=benchflow)
- NVMe nominal prefetch, `prefetch_chunks=100` (rejected for effect/tuning; all candidates skipped): [96d01b33a71f4f1bbb2d55a53a8aaacd](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/96d01b33a71f4f1bbb2d55a53a8aaacd?workspace=benchflow)
- Admission-prefetch official-image control: [23b7f315a6a54c08b484b113037abccc](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/23b7f315a6a54c08b484b113037abccc?workspace=benchflow)
- Admission-prefetch custom-image N=0 control: [3ee22e3ae07144039b83d9e6b8dfcbf0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/3ee22e3ae07144039b83d9e6b8dfcbf0?workspace=benchflow)
- Admission-prefetch configured N=100 treatment (invalid; scheduler observed zero): [b6bce02143a0431baa9935731cbe8b23](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/b6bce02143a0431baa9935731cbe8b23?workspace=benchflow)
- Repeat custom-image N=0 control (invalid; stale p1 image and only 255/256 completed): [048fa4300c4c4b878941f72395c1258e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/048fa4300c4c4b878941f72395c1258e?workspace=benchflow)
- Repeat configured N=100 treatment (invalid; stale p1 image and no prefetch series): [eddf9874c8304cf79fe3231b722be21c](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/eddf9874c8304cf79fe3231b722be21c?workspace=benchflow)
- Repaired-image N=0 control (performance-invalid; 255/256 completed): [3581db3f82d7427c883ff72113390121](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/3581db3f82d7427c883ff72113390121?workspace=benchflow)
- Repaired-image N=100 treatment (mechanism accepted; performance provisional): [b28bd1db0836406a94c31c2e3faa7c35](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/b28bd1db0836406a94c31c2e3faa7c35?workspace=benchflow)
- AgentX Weka N=0 control: [d82302a3769541cd9f98ad91bd8c3a69](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/d82302a3769541cd9f98ad91bd8c3a69?workspace=benchflow)
- AgentX Weka concurrency-32 N=100 treatment (policy ineffective; performance inconclusive): [915dac9e54d54b18b9b5a79ac8f69c2b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/915dac9e54d54b18b9b5a79ac8f69c2b?workspace=benchflow)
- AgentX Weka concurrency-64 N=0 control: [beaf48bcd79d46a1b155ba9af508ec2c](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/beaf48bcd79d46a1b155ba9af508ec2c?workspace=benchflow)
- AgentX Weka concurrency-64 N=100 treatment (mechanism accepted; performance inconclusive): [6febe03b9d1f4b4e95f628a34e59c038](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/6febe03b9d1f4b4e95f628a34e59c038?workspace=benchflow)
- V2.1 reactive overlay: [afce8c043cfe4e01b6e65ed8b26cf69d](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/afce8c043cfe4e01b6e65ed8b26cf69d?workspace=benchflow)
- V2.1 shadow: [7a5ba9c3e31c4a1c9dd311b5555d27fa](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/7a5ba9c3e31c4a1c9dd311b5555d27fa?workspace=benchflow)
- V1 N=100 negative control: [3deeb035004e46a291c4975560e5e0d5](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/3deeb035004e46a291c4975560e5e0d5?workspace=benchflow)
- V2.1 live, zero submitted: [6ccc8c955f6149f488bc6c488f95d927](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/6ccc8c955f6149f488bc6c488f95d927?workspace=benchflow)
- V2.1 reactive stock: [4111b847dba14ae0a8f6b6617aec939e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/4111b847dba14ae0a8f6b6617aec939e?workspace=benchflow)

## Next experiment

Version2 is now the active experiment. Before another performance comparison:

1. split capacity skip by hard-cap, pending-bundle, step-budget, speculative-budget, manager-free-slot, and recheck-redundant reasons;
2. expose total allocated, truly free, evictable demand, speculative-ready, read-pending, and write-pending CPU blocks;
3. reserve 64, 256, and 1,024 speculative blocks (about 128 MiB, 512 MiB, and 2 GiB in this run) without allowing speculative work to evict demand data;
4. rerun stock, overlay, shadow, and live with at least three randomized, node-paired repetitions;
5. reject any live mechanism cell with zero submitted promotions, missing-file loads, terminal-accounting drift, or speculative eviction of demand data;
6. analyze TTFT and throughput only after submitted promotions complete successfully and at least some become useful.

The V1 N sweep is no longer the primary next step. V1 should remain a short negative safety control, not a tuning candidate.