---
title: "ABC KV lookup experiments"
date: "2026-08-19"
type: "research-index"
experiment: "ABC CPU offload KV lookup comparison"
status: "active"
---

# ABC

## Lookahead demand staging result — 2026-09-07

- [[Reports/2026-09-07 - Lookahead demand staging v1 to v3|Lookahead demand staging — v1 regression, diagnosis, fix, and the pivot to retrieval parallelism]] — **the design checkpoint below was implemented, measured, and is now a clean negative.** v1 cost −10.0% throughput and +40.4% mean TTFT; three probe-path defects were quantified (predicted +2.92 ms/step against an observed +3.35 ms) and fixed; the accepted paired run measured −1.7% throughput with p95 TTFT flat — indistinguishable from neutral.

**Latest working conclusion.** Lookahead demand staging works mechanically — it removes 32.4% of external retrieval stall — but produces no end-to-end benefit on AgentX C64 with a local NVMe tier, because **admission is gated by batch capacity, not by metadata readiness**. The decisive evidence is that a 32.4% stall reduction moved `num_requests_running` by 1.9% and waiting depth by −1.8%: retrieval was not what requests were waiting on. vLLM's reactive path already overlaps retrieval with queue wait (the admission loop `continue`s on a deferred lookup and only `break`s on allocation failure), so lookahead's marginal population is just the requests behind a head-of-line allocation break.

**Two generalizable findings**, both new:

1. **Cost in any probing policy scales with per-probe burst size, not probe rate.** Cutting probes 7x left the synchronous lookup p99 unchanged; cutting the scan 2.6x halved it.
2. **The filesystem tier's data path is serialized per job.** `submit_load`/`submit_store` enqueued one task per job, so `n_read_threads` only parallelized *across* requests. Benchmarked on the target NVMe: 512 × 2 MiB blocks take 372 ms (2.89 GB/s) serially versus 160 ms (6.72 GB/s) split — 2.33x, on a device the runs left 74% idle. Implemented as `v3`; **not yet measured end to end.**

**Disposition.** Keep lookahead, defaulted off; do not sweep its knobs further. The next lever is the retrieval parallelization, which shortens the **demand** path for every request with an external hit and so is not subject to the capacity-bound argument.

## Design checkpoint — 2026-09-03

- [[Methodology/08 - Lookahead demand staging design investigation|Lookahead demand staging — design investigation]] — exploratory design for eager KV block loading against `vllm@2db1c4dc31`: no predictor, no new policy object. Recommends M0 (memoized probe + stage telemetry), L1 (CPU-side lookahead past the allocation barrier under a Cao-et-al. do-no-harm gate), L2 (budget-independent GPU staging, flag, off by default), and R1 (parallelize the single-threaded NVMe promotion job). Expected gain is regime-dependent and small on C64, zero on C32, never negative by construction; validation gates and falsification criteria included. **Superseded by the 2026-09-07 result above:** M0/L1 were implemented and measured neutral, R1 was implemented and remains unmeasured, and L2 was never built. The design's own gain model `min(W, P)` was falsified — it assumed a request runs once its data is ready, when in fact it runs when capacity frees.

## Research reset — 2026-08-21

- [[Continuation Readiness/00 - Index|COSTAR continuation-readiness research program]] — A0 certifies the existing C32/C64 traces for the continuation-retention oracle, TTL frontier, and request-readiness allocation. All 1,838 observed continuation edges are exact and ordered under `x_correlation_id`; explicit tool/lifecycle/workflow events remain absent.
- [[Future-Value Placement/00 - Index|COSTAR future-value placement experiments]] — C32 and C64 now validate substantial equal-capacity placement headroom. Matched next-use avoids 12/12 reads on C32 and 144/212 on C64; future-aware victim ranking produces the movement result while bypass reduces churn. Practical exact-key and hard contextual policies remain negative.
- [[Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration|COSTAR Experiment 0 — AgentX oracle-corpus calibration]] — both oracle traces are accepted: C32 has 2.24M events and C64 has 13.63M, with closed lifecycles/transfers, exact capacity conservation, checksums, and zero native-movement reconstruction mismatches.
- [[Reports/COSTAR Offline Oracle/00 - Index|COSTAR offline oracle experiment series]] — corrected external-target replay shows only 42/901 requests reuse external KV; the finite 131,072-slot clairvoyant retention policy avoids all 12 native reads and 36.44 seconds of measured device service. Next: value-of-information baselines for practical retention admission.
- [[2026-08-23 - ABC prefetch research brief for feedback|ABC KV-cache prefetching — short research brief for feedback]] — shareable one-page-style summary of the tested strategies, decisive metrics, negative results, and feedback questions.
- [[2026-08-21 - Independent research audit and redirection for speculative KV prefetching|Independent research audit and redirection for speculative KV prefetching]] — **broad opportunity remains, but the current V7 primary path is killed.** The next gate is a perfect-residency oracle, followed by deadline-aware working-set/data-readiness research; do not continue V7 heuristic tuning.

## Navigation

- [[Reports/00 - Index|Experiment Reports]] — executed benchmark runs, validations, failures, plots, and conclusions.
- [[Methodology/00 - Index|Methodology and Implementation]] — experiment definitions, plans, implementation guides, and design discussions.
- [[Version2/00 - Index|Version 2 — Proactive Speculative Prefetching]] — deterministic, event- and queue-informed program. The first five-cell V2.1 run validates the control plane and safety behavior but found zero live submissions because no truly free CPU KV slots remained.
- [[Version3/00 - Index|Version 3 — JIT Demand-Safe Speculative Prefetch]] — v6 single-owner, demand-priority implementation. Mechanism accepted in the first AgentX run; causal performance benefit remains inconclusive because the nodes diverged before speculative promotion.

## Methodology and implementation

- [[Methodology/01 - Experiment Definition|01 — Experiment Definition]] — problem statement, proposed end-state framework, and the four-phase path from reactive fetching to speculative prefetching.
- [[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide|02 — Phase 1 Naive Prefetch Implementation Guide (historical)]] — rejected post-miss N-chunk read-ahead design, retained as a split historical guide.
- [[Methodology/03 - Phase 2 Heuristic Prefetch Implementation Guide|03 — Phase 2 Heuristic Prefetch Implementation Guide (tentative)]] — adaptive N controller, feature-based block selection, and sliding-window group support.
- [[Methodology/04 - Phase 1 Queued-Request Oracle Prefetch Implementation Guide|04 — Phase 1 Queued-Request Oracle Prefetch Implementation Guide]] — current admission-time, assume-resident implementation tutorial.
- [[Methodology/05 - Initial versus Admission-Time Proactive Prefetching|05 — Initial versus Admission-Time Proactive Prefetching]] — end-to-end explanation of both designs, why the first failed, and how the current mechanism works.
- [[Methodology/07 - Dynamic admission and cross-scope prefetch roadmap|07 — Dynamic admission and cross-scope prefetch roadmap]] — proposed model-neutral byte/deadline policy and roadmap from local cold data to cross-vLLM and cross-session advisories.
- [[Methodology/08 - Lookahead demand staging design investigation|08 — Lookahead demand staging design investigation]] — 2026-09-03 design for eager loading built from the reactive path (M0/L1/L2/R1) with a do-no-harm promotion gate. Implemented and measured; see [[Reports/2026-09-07 - Lookahead demand staging v1 to v3|the 2026-09-07 result]].
- [[Methodology/2026-08-14 - Phase 1 queued-request oracle prefetch plan|2026-08-14 — Phase 1 queued-request oracle prefetch plan]] — controlled experiment plan for blind first-N queued-request promotion.

## Experiment reports

- [[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report|2026-08-10 — Nemotron no-offload versus CPU-offload KV lookup report]]
- [[Reports/2026-08-14 - Phase 1 CPU prefetch validation|2026-08-14 — Phase 1 CPU prefetch validation]] — rejected: no secondary tier was configured.
- [[Reports/2026-08-14 - Phase 1 NVMe prefetch validation|2026-08-14 — Phase 1 NVMe prefetch validation]] — rejected original post-miss candidate policy.
- [[Reports/2026-08-17 - Phase 1 admission prefetch first execution report|2026-08-17 — Phase 1 admission prefetch first execution report]] — invalid/inconclusive due manager/scheduler wiring mismatch and stale-image repeats.
- [[Reports/2026-08-18 - Phase 1 admission prefetch repaired-image validation|2026-08-18 — Phase 1 admission prefetch repaired-image validation]] — mechanism accepted; performance remains provisional.
- [[Reports/2026-08-18 - AgentX Weka admission prefetch first exploratory run|2026-08-18 — AgentX Weka admission prefetch first exploratory run]] — valid negative policy result at concurrency 32: first-N prefetch ran, but N=100 was mostly redundant, failed, and late.
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64|2026-08-18 — AgentX Weka admission prefetch at concurrency 64]] — Phase 1 queue-sensitivity supported: more waiting sharply increased useful yield and reduced lateness; performance remains inconclusive.
- [[Reports/2026-08-21 - Clean-prefetch v1 AgentX first comparison|2026-08-21 — Clean-prefetch v1 AgentX concurrency-32 comparison]] — full-cache admission worked, but 98.44% of useful promotions were late and performance was neutral.
- [[Reports/2026-08-22 - Clean-prefetch v1 AgentX concurrency 64 comparison|2026-08-22 — Clean-prefetch v1 AgentX concurrency-64 comparison]] — real queueing reduced lateness, but FIFO plan saturation and eviction regret make performance inconclusive and motivate demand cutoff plus deadline ordering.
- [[Reports/2026-08-22 - Clean-prefetch v1 repeat and attempted v2 invalidation|2026-08-22 — Clean-prefetch v1 repeat / attempted v2 invalidation]] — invalid for the surgical fix because both pods reused the exact v1 digest; the repeat reinforces the stale-FIFO and eviction-regret diagnosis.
- [[Reports/2026-08-22 - Clean-prefetch v2 AgentX concurrency 64 comparison|2026-08-22 — Clean-prefetch v2 AgentX concurrency-64 comparison]] — v2 mechanically passed but fixed-N=64 failed as a performance policy: timely chunk hits did not make complete requests ready, and eviction regret remained high.
- [[Reports/2026-08-23 - Working-set oracle AgentX first comparison|2026-08-23 — Working-set oracle AgentX first comparison]] — valid negative result for admission-time single-owner staging: 99.24% of intents still deferred at first lookup and performance remained near-neutral.
- [[Reports/2026-09-07 - Lookahead demand staging v1 to v3|2026-09-07 — Lookahead demand staging v1 to v3]] — **valid**. The only properly paired comparison in the campaign. Mechanism accepted as neutral (−1.7% throughput, p95 TTFT flat); the capacity-bound finding explains why every staging attempt in this series has stalled at the same wall. Identifies the serialized filesystem data path as the next lever.
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

## Version 3 JIT demand-safe checkpoint — 2026-08-21

[[Version3/00 - Index|Version3]] implements JIT activation, one earliest-deadline owner, demand-idle submission, demand-priority filesystem service, explicit demand/speculative allocation modes, physical reserve preservation, owner-bound cleanup, and a one-bundle retention lease. The first v6 treatment promoted 1,024 chunks: 512 useful, 448 wasted, and 64 pending at the measurement boundary. No reserve borrowing or lease reclamation occurred. Useful yield rose from 4.71% in the v5 failure to 50.0%, so the mechanism is accepted for continued research.

The observed AgentX deltas are not causal evidence. Treatment warmup was already 59% shorter and mean TTFT 89% lower while speculative submitted/promoted counters were still zero. The pair also used different nodes, both profiling phases cancelled 51 requests at timeout, and DCGM telemetry was absent. The next experiment is a replicated node cross-over with complete draining and GPU/device-specific telemetry.

## Clean-prefetch reset checkpoint — 2026-08-22

The clean v0.27.0 branch now provides the simplest valid full-cache admission baseline. At concurrency 32 it promoted correct chunks but 98.44% were late because almost no requests waited. At realistic concurrency 64, mean waiting rose to about five and late/useful fell to 44.67%, confirming that queue lead time matters.

The concurrency-64 pair still does not prove benefit. The global 64-chunk footprint remained saturated, admission-to-ready reached 71.96 seconds mean, 256/958 promotions were wasted, and 586/966 ordinary CPU victims were later demanded. Inspection identified a concrete policy-lifetime defect: remaining FIFO work is cancelled only at request finish, so it can be submitted after the request has entered demand lookup.

The next clean-branch change is request demand cutoff plus bounded deadline-aware ordering of existing exact intents. Do not sweep N until post-demand submission is impossible, stale ready-delay tails disappear, and on-time benefit exceeds eviction cost.

An attempted corrected-image validation on 2026-08-22 did not contain that change: runs `a6fe8407257c4c90b57771bce155a1f2` and `7f096342a54241ce99c6a98e53a87ca4` both resolved the original v1 digest `7c977def...`. As a v1 repeat, treatment had 1,025 submissions, 511 useful outcomes, 419 wasted outcomes, 576 eviction-regret events, and a 740-second ready-delay p90. It is invalid for accepting or rejecting the demand-cutoff/order patch.

## Clean-prefetch v2 result — 2026-08-22

The corrected v2 pair used one immutable digest and the same H100 node, with nearly identical warmup. The repair worked: 4,670/4,670 submissions promoted, no post-demand submission, no late or failed jobs, and admission-to-ready p90 fell from the v1 repeat's roughly 740 seconds to 1.70 seconds. Performance did not improve: request throughput was 0.56% lower, mean TTFT 2.14% higher, and p95 TTFT 4.05% higher.

The central failure is request-level coverage, not stale scheduling. A 64-chunk bundle covers only 1,024 tokens, versus about 41,153 external tokens per average request. The implementation calls each later CPU hit useful even when the connector still defers the request and reactively promotes thousands of remaining chunks. In addition, 2,684/4,670 proactive evictions were later regretted.

Do not sweep N yet. Add request-level readiness and admission-to-first-lookup telemetry, then run a one-request full-working-set oracle. Kill admission-time NVMe→CPU prefetch for AgentX if perfect CPU readiness cannot reduce deferred lookup and meet the replicated 5% TTFT or 3% throughput gate.

The working-set oracle code is now implemented but not yet built or benchmarked. It retains the fixed-N baseline and v2 cutoff/fallback, gives one scheduler-ordered request the complete bounded candidate set, adds a per-request eviction budget, and records request-level readiness/defer outcomes. The intended image tag is `v0.27.0-clean-prefetch-oracle-v1`; the first Nemotron ceiling is 8,192 chunks. Focused and tiering tests plus ruff/mypy pass. The full scheduler file remains gated by a mismatched shared virtualenv and must be rerun in the build container.

## Working-set oracle first result — 2026-08-23

The first working-set pair used the intended immutable image and concurrency-64 AgentX configuration. The treatment promoted 676,388 chunks, of which 99.55% were eventually useful, with no load failures or post-demand submissions. That high chunk usefulness did not translate into request readiness: only 20/2,638 intents (0.76%) were fully ready at the connector's first lookup, while 2,618 still deferred into reactive loading.

The manager-local `prefetch_complete_at_first_lookup=754` is not the authoritative readiness result. The target currently shrinks at the first admission-time source miss, so it describes completion of a shortened, probeable subset. The connector later observes the actual external working set. Preserve the original denominator and separate subset completion from full request readiness.

Performance was mixed and below the gate: request throughput +0.16%, mean TTFT -1.87%, p95 TTFT +1.11%, and p99 TTFT +6.51%. Do not increase N: no working set hit the 8,192-chunk ceiling. The next blocker is source readiness, owner reach, and deadline-complete coverage, plus trustworthy eviction accounting.


## COSTAR Experiment 0 corpus checkpoint — 2026-08-25

[[Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration|The initial COSTAR oracle-corpus batch]] used stock-reactive vLLM with one immutable image across AgentX/Weka C32 and C64 cells. C32 passed the complete automated validator: 2,241,218 events, 901 closed request lifecycles, exact transfer joins, no sequence or capacity errors, and reconstructed CPU occupancy reaching exactly 131,072 blocks. Its median/p95 HTTP-admission-to-first-lookup horizon was only 7.48/25.15 ms while the mean complete working set was about 7.93 GiB. This strengthens the case for an earlier completion-oriented oracle and contradicts the idea that admission-time selection normally provides enough time to stage complete requests.

The complete C64 trace was recovered and accepted: 6,959,277,072 bytes, SHA-256 `1167b512741bb97d2b76744cb238ede58fa0c6c2ef35ad7b3e9892c05b4ece3d`, 13,629,779 normalized events, and zero lifecycle, transfer, capacity, or native-movement replay errors.

Next: build and validate a soft request/prefix expected-value ranking across C32 and C64. Add chunked/compressed trace artifacts for future collections. This corpus validation does not establish live prefetch benefit.

## MLflow run registry


- COSTAR Experiment 0 C32 oracle corpus (validator accepted): [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- COSTAR Experiment 0 C64 pressure corpus (benchmark valid; trace certification pending): [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow)

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

- Version3 v6 JIT control (performance baseline; pair confounded): [19c4d1be0d0b4bbeb6358da05c32721f](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/19c4d1be0d0b4bbeb6358da05c32721f?workspace=benchflow)
- Version3 v6 JIT treatment (mechanism accepted; performance inconclusive): [5be11650e5a34043a3940c2e57dded74](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/5be11650e5a34043a3940c2e57dded74?workspace=benchflow)
- Clean-prefetch v1 repeat control: [7f096342a54241ce99c6a98e53a87ca4](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/7f096342a54241ce99c6a98e53a87ca4?workspace=benchflow)
- Clean-prefetch v1 repeat treatment (invalid for v2; stale FIFO reproduced): [a6fe8407257c4c90b57771bce155a1f2](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/a6fe8407257c4c90b57771bce155a1f2?workspace=benchflow)
- Clean-prefetch v2 concurrency-64 control: [24df61e44ac34ede8b94d42b23a8cb58](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/24df61e44ac34ede8b94d42b23a8cb58?workspace=benchflow)
- Clean-prefetch v2 concurrency-64 treatment (valid negative policy result): [c03bf0c79d6844da8069162633bb3d94](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/c03bf0c79d6844da8069162633bb3d94?workspace=benchflow)

- Working-set oracle control: [a34cca262119453a9837a2531c79c3de](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/a34cca262119453a9837a2531c79c3de?workspace=benchflow)
- Working-set oracle treatment (mechanism active; negative readiness result): [39a70a1b52e241bcb48abe5338d56110](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/39a70a1b52e241bcb48abe5338d56110?workspace=benchflow)

### Lookahead demand staging (2026-09-04 → 09-07, experiment 328)

- NVMe, stock v0.27.0 image: [65ccbf10c4354ab6b35e6e486b8b23a1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/65ccbf10c4354ab6b35e6e486b8b23a1?workspace=benchflow)
- NVMe, lookahead image, feature OFF (A/A gate, clean at +0.6%): [1573078c65f743c9a0bb3ce72be08237](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/1573078c65f743c9a0bb3ce72be08237?workspace=benchflow)
- NVMe, v1 lookahead ON (valid regression: −10.0% throughput, +40.4% mean TTFT): [ffe1170ac7d54fc5b0b40e28ae21afb7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/ffe1170ac7d54fc5b0b40e28ae21afb7?workspace=benchflow)
- NVMe, v2 ON (**invalid**; no contemporaneous control, sibling failed at 17 min): [5786964561f441c183f0bc303c355c98](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5786964561f441c183f0bc303c355c98?workspace=benchflow)
- NVMe, v2 ON full scan (conditionally valid, cross-day): [ca3ecd3cbcae4ef8bbb1e4514cb9469c](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/ca3ecd3cbcae4ef8bbb1e4514cb9469c?workspace=benchflow)
- **NVMe, v2 control — accepted pair**: [0e982c4a8094475fb84dfd63b9b9da0b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/0e982c4a8094475fb84dfd63b9b9da0b?workspace=benchflow)
- **NVMe, v2 @1024 treatment — accepted pair** (−1.7% throughput, p95 TTFT flat): [6c4bdb0195524b0c9109b3075edf79cb](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/6c4bdb0195524b0c9109b3075edf79cb?workspace=benchflow)
- No-offload replicates (0.1304 / 0.1315 / 0.1283 req/s): [d7982fea41bc4fdba7eec32ca99e8604](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d7982fea41bc4fdba7eec32ca99e8604?workspace=benchflow), [f5bd52f08c17455897d740ebdf1f9ebf](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f5bd52f08c17455897d740ebdf1f9ebf?workspace=benchflow), [65a2930350254d459ff8cadce67f22f0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/65a2930350254d459ff8cadce67f22f0?workspace=benchflow)
- No-offload outlier (0.0217 req/s on identical config; establishes single-run variance): [5cffd9d9c0654cf5bdbf4640f24a30dd](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5cffd9d9c0654cf5bdbf4640f24a30dd?workspace=benchflow)

## COSTAR finite-retention checkpoint — 2026-08-25

[[Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle|The finite-CPU retention oracle]] passed its movement ground-truth gate with 0/898 mismatches. The corrected target is the external matched-token segment, not total cached group counts: 157,283 references, 116,409 unique keys, and 42/901 nonzero requests. Recorded residency caused 12 native reads totaling 36.44 seconds of device service; an equal-capacity clairvoyant next-use admission policy avoided all 12 by rejecting low-future-value ordinary arrivals. This establishes retention/admission headroom, not end-to-end TTFT benefit and not a need for proactive reads.

Next: measure how much of this oracle can be recovered by online signals available before eviction—reuse frequency/recency, session/category identity, and predicted next-use ranking—while preserving native fallback.

## Next experiment

**Retrieval parallelism A/B (2026-09-07 onward).** The lookahead line is closed as neutral; the open question is whether parallelizing the filesystem tier's data path converts to TTFT.

1. Same-batch pair, lookahead **off** in both arms, on `v0.27.0-lookahead-v3`.
2. Vary only `blocks_per_task` inside the `secondary_tiers` entry: `0` reproduces the old one-task-per-job behaviour, the default `32` is the split.
3. At least two repetitions per cell, given that one no-offload replicate came in 6x off.
4. Verify a clean NVMe start per run — and read `benchflow-kv-cache` occupancy directly, **not** `storage_nvme_filesystem_usage_percent_by_node_mount`, which also counts the node's 1.8 TB model cache.
5. Read `kv_offload_tiering_lookup_sync_delay_seconds` p99 and `num_requests_running` before throughput. A mechanism that reduces stall without moving the running count is not on the critical path.

Standing gates from earlier checkpoints remain: preserve the immutable candidate target, repair eviction-outcome coverage, and require replicated same-node crossovers before accepting a latency claim.