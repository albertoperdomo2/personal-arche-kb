---
title: "ABC KV lookup experiments"
date: "2026-08-19"
type: "research-index"
experiment: "ABC CPU offload KV lookup comparison"
status: "active"
---

# ABC

## Research reset — 2026-08-21

- [[Future-Value Placement/00 - Index|COSTAR future-value placement experiments]] — Experiment 1 decomposes the finite-retention result: matched forced-admit and bypass-capable next-use policies both avoid 12/12 reads and 36.44 seconds; bypass is not required for the movement result but cuts matched-policy churn by 58.8%.
- [[Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration|COSTAR Experiment 0 — AgentX oracle-corpus calibration]] — C32 oracle trace accepted (2.24M events, zero validator errors); C64 object is present but awaits source-side trace certification because MLflow cannot stream the 6.96 GB artifact.
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

The C64 benchmark is operationally clean and the manually recovered 6.96 GB object is visible in MLflow, but the proxy returns 504 before streaming it. Treat C64 as a valid pressure-run and only a conditionally accepted corpus until the original file passes source-side validation and a checksum/report is uploaded.

Next: normalize C32, reproduce native ready/deferred and timing outcomes, certify C64, add chunked/compressed trace artifacts, then run the physically constrained clairvoyant oracle. This batch does not establish prefetch benefit.

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

## COSTAR finite-retention checkpoint — 2026-08-25

[[Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle|The finite-CPU retention oracle]] passed its movement ground-truth gate with 0/898 mismatches. The corrected target is the external matched-token segment, not total cached group counts: 157,283 references, 116,409 unique keys, and 42/901 nonzero requests. Recorded residency caused 12 native reads totaling 36.44 seconds of device service; an equal-capacity clairvoyant next-use admission policy avoided all 12 by rejecting low-future-value ordinary arrivals. This establishes retention/admission headroom, not end-to-end TTFT benefit and not a need for proactive reads.

Next: measure how much of this oracle can be recovered by online signals available before eviction—reuse frequency/recency, session/category identity, and predicted next-use ranking—while preserving native fallback.

## Next experiment

The current pair did not realize a perfect-residency oracle. Before another performance sweep:

1. preserve the immutable complete candidate target and distinguish selected-subset completion from connector-authoritative full readiness;
2. ensure or gate on source residency, or re-probe transient admission misses while mirroring completes;
3. repair eviction-outcome coverage, since 64.8% of victim outcomes exceeded the current history;
4. require the treatment to make at least 50% fewer requests defer before interpreting latency;
5. then run replicated same-node crossovers and require at least 5% lower mean/p95 TTFT or 3% higher throughput.

Do not increase the 8,192-chunk ceiling: the observed working sets were not clipped. If genuine request readiness still cannot meet the end-to-end gate, stop admission-time CPU staging and move prediction earlier than HTTP admission or integrate staging with source-data readiness.