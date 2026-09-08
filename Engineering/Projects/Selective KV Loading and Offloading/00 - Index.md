---
title: Selective KV loading and offloading
date: 2026-09-08
type: project-index
status: router-threshold-live-validated
topic: KV cache routing and offloading
repos:
  - vllm-project/vllm
  - llm-d/llm-d-router
---

# Selective KV loading and offloading

## Objective

Let llm-d-router make independent per-request decisions about whether vLLM should load reusable KV and how much newly computed KV should be offloaded. vLLM remains responsible for enforcing those decisions and moving the KV bytes.

## Confirmed direction

- There is a load-value threshold below which recomputation is preferable. Start with a statically configured, experimentally calibrated threshold; preserve an extension point for dynamic decisions later.
- The router must be able to override both directions independently: loading existing KV and offloading newly computed KV.
- The offload-direction API exists in vLLM as `kv_transfer_params.max_offload_tokens`.
- The v0.27.0-based vLLM patch makes `kv_load_tiers=[]` skip all offload sources, including primary CPU, while preserving local HBM prefix-cache reuse.
- The first router version is deliberately static. It validates capability gating and request mutation before introducing prefix evidence or threshold calibration.
- The threshold calibration compares load-allowed and forced-recompute requests with equivalent cache state. Request-level TTFT is the primary outcome; transfer, scheduler, token-source, and resource metrics explain the mechanism and validate each sample.

## Documents

| Document | Scope |
|---|---|
| [[01 - Initial implementation plan]] | Cross-repository sequence, contracts, static policy, calibration, rollout, and acceptance criteria |
| [[02 - vLLM primary-tier selective loading]] | Engine enforcement, primary CPU behavior, promotion correctness, and tests |
| [[03 - llm-d-router selective load and offload policy and wiring]] | Router policy, request mutation, sidecar/coordinator propagation, and tests |
| [[04 - vLLM selective load - problem statement]] | Focused motivation, required binary load opt-out, GPU-cache boundary, independent load/store controls, and success criteria |
| [[05 - 2026-09-04 vLLM binary opt-out cluster validation]] | Functional cluster validation of the vLLM binary load opt-out |
| [[06 - 2026-09-05 naive llm-d-router static gating prototype]] | Implemented local static policy plugin, optimized-baseline configuration, validation, limitations, and transition to calibrated policy |
| [[07 - Selective loading calibration test plan]] | Paired load-versus-recompute experiment, metrics, validity rules, threshold selection, and rollout acceptance criteria |
| [[08 - 2026-09-07 selective loading calibration results]] | Functional validation, targeted break-even sweep, AgentX pressure evidence, provisional threshold, and validity limitations |
| [[09 - 2026-09-08 Diadochos router threshold live validation]] | Live validation of the 1,024-token EPP gate, evidence-pipeline fixes, deterministic reduced-cache procedure, results, and caveats |

## Current status

The vLLM binary opt-out was implemented on a v0.27.0-based image and functionally validated in the `benchflow` cluster: an empty load-tier list skipped offload-manager lookup, a zero offload-token cap prevented storage, and an HBM-resident prefix still produced a local cache hit.

The naive router version is committed locally as `llm-d/llm-d-router@ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4` on branch `prototype/selective-kv-policy`. It adds an alpha `selective-kv-policy` `PreRequest` plugin with an explicit `binary-opt-out-v1` capability assertion and independent `preserve`/`disable` load and offload policies. Focused tests and `make presubmit` pass.

The always-recompute benchmark demonstrated that disabling every external load can be severely harmful under a long-prefix, high-pressure workload. It validates the need for selective rather than unconditional opt-out, but it is not suitable for locating the break-even threshold because it produced no load samples and entered pathological queueing.

The first targeted calibration batch is complete. All 27 load-allowed probes consumed external KV and all 27 forced-recompute probes avoided it. In the quiescent three-repetition sweep, 752 observed external tokens was the first bucket where loading won 3/3 and every larger bucket also won 3/3. The provisional router recommendation is a conservative 1,024 external-token threshold, limited to this deployment fingerprint. This is not yet production calibration: the sweep is under-replicated, pressure was quiescent, and the concurrency-32 AgentX comparisons were confounded by node placement or telemetry-sampling drift.

Sampled resolution telemetry also showed request-visible load waits above one second when two to four transfer jobs were pending. This supports a later pressure-aware veto, but the sample is too small to set that veto now.

The 2026-09-08 Diadochos live validation deployed EPP image `dev-a808637f` with the 1,024-token threshold. After fixing the TLS service name and qualifying self-describing KV events with pod and model identity, a temporary 131,072-token HBM cache made eviction deterministic: the 512-token case recomputed with zero external tokens, while the 2,048-token case restored 2,032 external tokens and transferred 266,338,304 bytes. The normal 1,824,960-token GPU cache and production-like auth/debug settings were restored afterward. See [[09 - 2026-09-08 Diadochos router threshold live validation]].

The live result is functional evidence, not new threshold calibration. It used one probe per branch, no event replay or index warmup, and a redundant KServe tokenizer remained crashing outside the successful tokenization path.

## Current dependencies

- vLLM PR #48123 introduced secondary-tier `kv_load_tiers` filtering.
- vLLM PR #39983 introduced `max_offload_tokens`.
- vLLM PR #52397 addresses partial-tail and `max_offload_tokens=0` handling; production activation must use a build with equivalent behavior.
- llm-d-router issue #1952 is adjacent work exposing cache location to plugins.
- No open overlapping llm-d-router selective-loading or request-wiring issue/PR was found in the duplicate search performed before the prototype.
- A tracking issue is required before proposing the router work upstream.
- The router's exact per-tier prefix-count semantics must be confirmed before calculating external reusable tokens.
- Request decisions must be correlated with vLLM outcomes and prompt-token sources before threshold data is accepted.

## Decision log

- 2026-09-03: Treat load and offload as independent policy outputs.
- 2026-09-03: Choose static load-threshold calibration for the first production-oriented version; defer adaptive policy.
- 2026-09-03: Reuse vLLM's existing `max_offload_tokens` contract rather than adding another offload API.
- 2026-09-03: Stage primary-tier loading as binary opt-out first, then consider source-selective CPU/STORAGE semantics.
- 2026-09-04: Accept the v0.27.0-based vLLM binary opt-out as functionally validated for router integration while retaining a CPU-resident-only replay as follow-up evidence.
- 2026-09-05: Implement the router's first vertical slice as a static, configuration-driven `PreRequest` plugin rather than embedding a calibration policy immediately.
- 2026-09-05: Require an explicit operator capability assertion and register the plugin as alpha.
- 2026-09-05: Keep native-generate, sidecar/coordinator propagation, positive offload caps, per-tier selection, and threshold decisions outside the naive version.
- 2026-09-07: Define the static threshold as the smallest external reusable-token bucket where paired request-level TTFT reliably favors loading under representative pressure.
- 2026-09-07: Treat transfer time as mechanism evidence, not the threshold objective; active DMA time alone omits queueing, synchronization, and scheduler completion observation.
- 2026-09-07: Fail open to normal vLLM loading when backend capability or cache evidence is missing, stale, incompatible, or outside the calibrated envelope.
- 2026-09-07: Accept the binary router-to-vLLM opt-out as functionally validated across 54 targeted probes.
- 2026-09-07: Use 1,024 external reusable tokens as the conservative provisional threshold for the next router experiment; 752 tokens is the observed quiescent crossing candidate.
- 2026-09-07: Reject the telemetry-overhead comparison and quantitative cross-arm AgentX ranking because node placement and telemetry settings were not jointly controlled.

- 2026-09-08: Validate the 1,024-token router gate end to end on Diadochos after fixing the TLS FQDN and self-describing, model-qualified KV-event identity.
- 2026-09-08: Accept the reduced-cache 512-recompute and 2,048-restore pair as functional validation only; reject the earlier full-cache attempts for enforcement/performance conclusions.

## Next checkpoint

1. Pin the model pod to one H100 host and keep telemetry sampling at 1.0 for every arm.
2. Repeat 512, 768, 1,024, 1,536, and 2,048 external-token buckets with at least 20 valid repetitions at concurrency 1.
3. Repeat the narrowed sweep under representative transfer pressure.
4. Add a router decision identifier that joins router evidence, selected endpoint, vLLM resolution, and AIPerf request data.
5. Compare always load, the provisional 1,024-token policy, and always recompute on the same representative trace.
6. If the crossing moves with pressure, calibrate a hysteretic pressure veto from pending jobs/bytes and recent request-visible load wait.

## Related

- [[vLLM and llm-d-router KV cache responsibility split]]
- [[vLLM KV offload retrieval path - lookup, promotion, and load]]
- [[vLLM KV Events canonical form]]
- [[Activity-Based KV Cache Offloading]]
- [[01 - Calibration Protocol]]
- [[03 - KV Transfer Metrics and PromQL]]

## Provenance

Direct code inspection of `vllm-project/vllm`, cluster validation of the v0.27.0-based selective-load image, stakeholder clarification from Maroon, the earlier router design inspection at `b1bf63da5e9a52dc8815264809d00f45f5b5e966`, the implemented local router prototype at `ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4`, and the paired calibration design developed on 2026-09-07, and the Diadochos router-threshold live validation on 2026-09-08. Router focused tests and full presubmit were observed passing before this update.