---
title: Selective KV loading and offloading
date: 2026-09-05
type: project-index
status: prototype-implemented
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

## Documents

| Document | Scope |
|---|---|
| [[01 - Initial implementation plan]] | Cross-repository sequence, contracts, static policy, calibration, rollout, and acceptance criteria |
| [[02 - vLLM primary-tier selective loading]] | Engine enforcement, primary CPU behavior, promotion correctness, and tests |
| [[03 - llm-d-router selective load and offload policy and wiring]] | Router policy, request mutation, sidecar/coordinator propagation, and tests |
| [[04 - vLLM selective load - problem statement]] | Focused motivation, required binary load opt-out, GPU-cache boundary, independent load/store controls, and success criteria |
| [[06 - 2026-09-05 naive llm-d-router static gating prototype]] | Implemented local static policy plugin, optimized-baseline configuration, validation, limitations, and transition to calibrated policy |

## Current status

The vLLM binary opt-out was implemented on a v0.27.0-based image and functionally validated in the `benchflow` cluster: an empty load-tier list skipped offload-manager lookup, a zero offload-token cap prevented storage, and an HBM-resident prefix still produced a local cache hit.

The naive router version is committed locally as `llm-d/llm-d-router@ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4` on branch `prototype/selective-kv-policy`. It adds an alpha `selective-kv-policy` `PreRequest` plugin with an explicit `binary-opt-out-v1` capability assertion and independent `preserve`/`disable` load and offload policies. Focused tests and `make presubmit` pass.

The router commit is a local prototype, not an upstream contribution. The next checkpoint is to build a router image and validate the complete optimized-baseline EPP-to-vLLM path.

## Current dependencies

- vLLM PR #48123 introduced secondary-tier `kv_load_tiers` filtering.
- vLLM PR #39983 introduced `max_offload_tokens`.
- vLLM PR #52397 addresses partial-tail and `max_offload_tokens=0` handling; production activation must use a build with equivalent behavior.
- llm-d-router issue #1952 is adjacent work exposing cache location to plugins.
- No open overlapping llm-d-router selective-loading or request-wiring issue/PR was found in the duplicate search performed before the prototype.
- A tracking issue is required before proposing the router work upstream.

## Decision log

- 2026-09-03: Treat load and offload as independent policy outputs.
- 2026-09-03: Choose static load-threshold calibration for the first production-oriented version; defer adaptive policy.
- 2026-09-03: Reuse vLLM's existing `max_offload_tokens` contract rather than adding another offload API.
- 2026-09-03: Stage primary-tier loading as binary opt-out first, then consider source-selective CPU/STORAGE semantics.
- 2026-09-04: Accept the v0.27.0-based vLLM binary opt-out as functionally validated for router integration while retaining a CPU-resident-only replay as follow-up evidence.
- 2026-09-05: Implement the router's first vertical slice as a static, configuration-driven `PreRequest` plugin rather than embedding a calibration policy immediately.
- 2026-09-05: Require an explicit operator capability assertion and register the plugin as alpha.
- 2026-09-05: Keep native-generate, sidecar/coordinator propagation, positive offload caps, per-tier selection, and threshold decisions outside the naive version.

## Next checkpoint

1. Build and push an EPP image containing router commit `ac5446eb`.
2. Add the alpha plugin and capability gate to the optimized-baseline values.
3. Validate backend-observed JSON and vLLM lookup/transfer metrics through the router.
4. Isolate load-only behavior with `loadPolicy=disable` and `offloadPolicy=preserve`.
5. Replace static load policy with a deterministic threshold using selected-endpoint, precise per-tier prefix evidence.

## Related

- [[vLLM and llm-d-router KV cache responsibility split]]
- [[vLLM KV offload retrieval path - lookup, promotion, and load]]
- [[vLLM KV Events canonical form]]
- [[Activity-Based KV Cache Offloading]]
- [[01 - Calibration Protocol]]

## Provenance

Direct code inspection of `vllm-project/vllm`, cluster validation of the v0.27.0-based selective-load image, stakeholder clarification from Maroon, the earlier router design inspection at `b1bf63da5e9a52dc8815264809d00f45f5b5e966`, and the implemented local router prototype at `ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4`. Router focused tests and full presubmit were observed passing before this update.