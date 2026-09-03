---
title: Selective KV loading and offloading
date: 2026-09-03
type: project-index
status: proposed
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
- The offload-direction API already exists in vLLM as `kv_transfer_params.max_offload_tokens`. Router-side policy and reliable request propagation are missing.
- The primary CPU tier is not currently controlled by `kv_load_tiers`, so vLLM needs a focused follow-up before the router can safely emit an empty load-tier list.

## Documents

| Document | Scope |
|---|---|
| [[01 - Initial implementation plan]] | Cross-repository sequence, contracts, static policy, calibration, rollout, and acceptance criteria |
| [[02 - vLLM primary-tier selective loading]] | Engine enforcement, primary CPU behavior, promotion correctness, and tests |
| [[03 - llm-d-router selective load and offload policy and wiring]] | Router policy, request mutation, sidecar/coordinator propagation, and tests |

## Current status

Planning complete; implementation not started. The recommended first code contribution is the vLLM binary load opt-out: an explicit `kv_load_tiers=[]` must skip every offload source, including the CPU primary tier.

## Current dependencies

- vLLM PR #48123 introduced secondary-tier `kv_load_tiers` filtering.
- vLLM PR #39983 introduced `max_offload_tokens`.
- Open vLLM PR #52397 fixes a partial-tail assertion failure, including incorrect handling of `max_offload_tokens=0`, and should be resolved or accounted for before enabling router offload overrides broadly.
- llm-d-router issue #1952 is adjacent work exposing cache location to plugins.
- No open overlapping selective-loading/router-wiring PR was found on 2026-09-03.

## Decision log

- 2026-09-03: Treat load and offload as independent policy outputs.
- 2026-09-03: Choose static load-threshold calibration for the first version; defer adaptive policy.
- 2026-09-03: Reuse vLLM’s existing `max_offload_tokens` contract rather than adding another offload API.
- 2026-09-03: Stage primary-tier loading as binary opt-out first, then consider source-selective CPU/STORAGE semantics.

## Related

- [[vLLM and llm-d-router KV cache responsibility split]]
- [[vLLM KV offload retrieval path - lookup, promotion, and load]]
- [[vLLM KV Events canonical form]]
- [[Activity-Based KV Cache Offloading]]
- [[01 - Calibration Protocol]]

## Provenance

Direct code inspection of `vllm-project/vllm@0e3ac4907d21e77cb4781338c49fef17bfea8f2b` and `llm-d/llm-d-router@b1bf63da5e9a52dc8815264809d00f45f5b5e966`, stakeholder clarification from Maroon, and duplicate-work checks on 2026-09-03.