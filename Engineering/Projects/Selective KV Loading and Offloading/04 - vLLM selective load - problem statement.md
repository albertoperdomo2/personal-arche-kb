---
title: vLLM selective load - problem statement
date: 2026-09-03
type: problem-statement
status: proposed
topic: KV cache routing and offloading
repos:
  - vllm-project/vllm
  - llm-d/llm-d-router
inspected_vllm_commit: 0e3ac4907d21e77cb4781338c49fef17bfea8f2b
inspected_router_commit: b1bf63da5e9a52dc8815264809d00f45f5b5e966
---

# vLLM selective load — problem statement

## Summary

A KV-cache hit answers whether reusable state exists, but it does not answer whether retrieving that state is cheaper than recomputing it on the GPU. Today, vLLM’s tiered-offloading path automatically attempts to retrieve matching offloaded KV during request admission. llm-d-router has the endpoint, prefix, and tier visibility needed to make a higher-level economic decision, but it cannot yet reliably tell vLLM to forgo every offload source and recompute instead.

The required first capability is deliberately narrow: `kv_transfer_params.kv_load_tiers=[]` must be a reliable binary opt-out from offloaded KV loading, including the primary CPU tier. It must not disable or bypass vLLM’s local GPU prefix cache. This creates a stable enforcement primitive that the router can drive with a statically configured, experimentally calibrated threshold before any adaptive policy is attempted.

This document defines the problem and the required behavioral boundary. The cross-repository delivery plan is in [[01 - Initial implementation plan]], the vLLM engine design is in [[02 - vLLM primary-tier selective loading]], and the router design is in [[03 - llm-d-router selective load and offload policy and wiring]].

## Existence is not economic value

An offloaded block can be present and addressable while still being a poor reuse candidate. Reusing it incurs more than the raw media read: the request can wait for tier lookup, asynchronous completion, promotion, CPU allocation, CPU-to-GPU transfer, connector coordination, and scheduler retries. Recomputing the corresponding prompt tokens consumes GPU work, but for a sufficiently small prefix or a sufficiently slow or pressured load path, recomputation can complete sooner.

The decision is therefore a comparison, not a cache-membership test:

$$
C_{\text{reuse}} = C_{\text{lookup}} + C_{\text{queue}} + C_{\text{transfer}} + C_{\text{promotion}} + C_{\text{coordination}}
$$

$$
\text{reuse only when } C_{\text{reuse}} < C_{\text{GPU recompute}}
$$

These costs depend on prefix length, active queueing, tier medium and locality, CPU memory pressure, storage or network latency, transfer bandwidth, and metadata freshness. A hit is necessary for reuse, but it is not sufficient evidence that reuse will improve time to first token or overall throughput.

## Current ownership and behavior

vLLM owns the KV blocks, scheduler state, physical tier managers, promotion state, and GPU transfer execution. Its local GPU prefix cache is the first and cheapest reuse domain. The tiered-offloading connector then examines offloaded state during admission: the primary path is CPU to GPU, while a secondary storage hit is promoted from storage to CPU and then loaded from CPU to GPU.

llm-d-router is the control plane. It receives cache-location information, knows which endpoint was selected, and can observe how much of the request prefix is cached by tier. That global endpoint and tier view is the appropriate place to decide whether expected offloaded reuse is valuable enough to request. The router should express the decision; vLLM should enforce it and remain solely responsible for moving bytes.

The current contract is incomplete for this purpose. vLLM’s `kv_load_tiers` filtering applies to secondary tiers, but the primary CPU lookup remains unconditional. Consequently, an empty list can suppress secondary lookup while still allowing a CPU-resident match to load. The apparent meaning “load from no offload tier” is therefore not yet reliable.

The division of responsibility and the existing vLLM-to-router and router-to-vLLM control channels are documented in [[vLLM and llm-d-router KV cache responsibility split]]. The admission-time lookup and promotion state machine are documented in [[vLLM KV offload retrieval path - lookup, promotion, and load]].

## Data paths and the GPU-cache boundary

Selective loading concerns only KV that is outside the local GPU prefix cache.

The relevant paths are:

```text
CPU-resident offloaded KV:
CPU -> GPU

Storage-resident offloaded KV:
storage -> CPU -> GPU

Locally resident GPU prefix KV:
GPU prefix-cache hit; no offload-tier load
```

The CPU tier is both an independently reusable source and the mandatory staging gateway for a secondary-tier promotion. The implementation must distinguish those roles internally, but the externally required first behavior is simpler: when the router sends an empty load-tier list, vLLM must not initiate reuse from either CPU or any secondary offload tier.

A local GPU prefix-cache hit remains valid regardless of `kv_load_tiers`. If part of a request is already cached on the selected GPU and a later part exists only in CPU or storage, the empty list preserves the GPU-resident prefix and recomputes only the remainder that would otherwise have been loaded from an offload tier. Selective loading must never turn a local GPU hit into a miss.

## When offloaded reuse can be slower

Several common conditions can reverse the expected benefit of reuse:

- The matching offloaded prefix is small, so GPU recomputation is cheaper than the fixed lookup, scheduling, and transfer overhead.
- The CPU tier is under memory or bandwidth pressure, increasing allocation, queuing, or CPU-to-GPU transfer delay.
- A storage tier is slow or temporarily pressured, increasing lookup and read time before the mandatory CPU staging step.
- Multiple requests compete for the same tier workers, transfer engines, or scheduler attention.
- Remote or storage metadata is stale, so the router predicts reusable state that is gone or no longer cheaply reachable.
- The match is only partial, and the saved GPU work is too small to amortize retrieval coordination.
- Endpoint conditions have changed between routing and admission, so a previously reasonable load decision is no longer economical.

These cases explain why automatic retrieval from every known cache location is not always the best default for a router-directed deployment. They also explain why the first policy must be calibrated against end-to-end behavior rather than derived from storage bandwidth alone.

## Required binary opt-out contract

The minimum reliable contract is:

```json
{
  "kv_transfer_params": {
    "kv_load_tiers": []
  }
}
```

For tiered offloading, this request must mean:

- Do not reuse KV whose source is the primary CPU tier.
- Do not query, retrieve, or promote KV from storage, object, P2P, or any other secondary offload tier.
- Continue to use any matching prefix already present in the local GPU prefix cache.
- Recompute on the GPU the eligible prompt portion that would otherwise have come from an offload tier.
- Preserve the request’s model semantics and output correctness.
- Avoid changing store/offload behavior unless the request independently provides an offload control.

When `kv_load_tiers` is absent, existing vLLM behavior must remain unchanged. This backward-compatible default is essential for clients and routers that do not know about selective loading.

The initial requirement is a trustworthy all-offload-sources opt-out. Complete non-empty source selection—such as precise CPU-only versus storage-only behavior across promotion and staging—is useful future work but is not required to solve the immediate policy problem.

## Loading and offloading are independent

Loading decides whether existing offloaded KV should be reused for the current request. Offloading decides how much newly computed KV from that request should be stored for possible future reuse. One decision must not imply the other.

vLLM already exposes the offload/store control as `kv_transfer_params.max_offload_tokens`: `0` disables offload for the request, a positive integer caps the leading request tokens eligible for offload, and absence or `None` leaves the path uncapped. The router policy and propagation for this field are missing.

The two controls permit four valid combinations:

| Reuse existing offloaded KV | Store newly computed KV | Router request controls | Intended behavior |
|---|---|---|---|
| No | No | `kv_load_tiers=[]`, `max_offload_tokens=0` | Use local GPU hits, recompute the rest, and do not offload the new result |
| No | Yes | `kv_load_tiers=[]`, positive router-selected cap or deliberate uncapped behavior | Recompute instead of loading, but store the selected newly computed prefix for future requests |
| Yes | No | Compatible load tiers or the default load behavior, `max_offload_tokens=0` | Reuse eligible existing KV, but do not add newly computed KV to offload tiers |
| Yes | Yes | Compatible load tiers or the default load behavior, positive router-selected cap or deliberate uncapped behavior | Reuse eligible existing KV and offload the independently selected new prefix |

This independence is required because current-system economics and future reuse value are different questions. A CPU tier can be too pressured to load from now while the newly computed shared prefix is still valuable to store for later; conversely, an existing large prefix can be worth loading even when the new request-specific tail is not worth offloading.

## Static threshold first

The first router policy should use a statically configured threshold derived from controlled measurement. Conceptually, the router compares the amount of reusable offloaded prefix against a configured minimum and sends `kv_load_tiers=[]` below that boundary. The exact signal, units, scope, and calibration procedure belong in [[01 - Initial implementation plan]] and must be based on measured break-even behavior.

A static threshold is intentionally limited. It will not respond automatically to transient storage latency, CPU pressure, queue growth, topology changes, or drift in GPU recomputation cost. Its purpose is to establish a deterministic, observable baseline and validate the end-to-end control contract. Dynamic or adaptive policy can follow once the binary opt-out is reliable and calibration data demonstrates which live signals improve the decision.

No universal threshold is asserted here. It may differ by model, hardware, tier, topology, workload, concurrency, and deployment configuration.

## Concrete 128-token example

Suppose the selected endpoint already has a 128-token matching prefix in an offload tier, but that prefix is not present in its local GPU cache. Automatic retrieval treats this as reusable because the KV exists. Economically, the router must compare the expected end-to-end retrieval path with recomputing those 128 tokens on the selected GPU.

For a CPU-resident match, the reuse path includes admission coordination and CPU-to-GPU transfer. For a storage-resident match, it additionally includes storage lookup/read and storage-to-CPU promotion before CPU-to-GPU transfer. Recomputation avoids those steps but consumes GPU prefill work.

The example does not establish that 128 tokens should always be recomputed or always be loaded. If calibration shows that 128 tokens falls below the measured break-even threshold for the relevant model, tier, hardware, and load level, the router sends `kv_load_tiers=[]`; vLLM retains any local GPU prefix hit and recomputes the offloaded portion. If 128 tokens is above the calibrated threshold, the router allows the normal load path. The point is that existence alone cannot choose between those outcomes.

## Success criteria

The problem is solved at the contract level when all of the following are true:

- `kv_load_tiers=[]` prevents initial reuse from every offload source, including the primary CPU tier.
- The same request still benefits from any matching KV already in the selected engine’s local GPU prefix cache.
- Suppressed offload-tier reuse results in normal GPU recomputation rather than an error, indefinite retry, or false external-cache hit.
- Omitting `kv_load_tiers` preserves current loading behavior.
- `kv_load_tiers` and `max_offload_tokens` can be set and propagated independently in all four combinations.
- Router decisions survive request mutation, disaggregated-serving sidecars, retries, and serialization without overwriting unrelated connector parameters.
- The initial static threshold is configurable, experimentally calibrated, and observable; no hard-coded universal token count is presented as correct.
- Selective loading does not change generated-token correctness or model semantics.
- Tests distinguish local GPU hits from CPU and secondary-tier hits and prove that only the intended source class is suppressed.
- Operational telemetry can separate “no offloaded KV existed” from “offloaded KV existed but policy chose recomputation.”

## Non-goals

This problem statement does not propose:

- Lazy offload or changes to when GPU blocks are evicted or copied to CPU.
- A direct storage-to-GPU data path that bypasses the CPU staging tier.
- Full non-empty source-selective semantics for every CPU, storage, locality, and promotion combination.
- A dynamic, learned, or continuously adaptive threshold in the first version.
- Changes to model computation, numerical behavior, sampling, or generated-output correctness.
- Moving KV bytes in llm-d-router; physical data movement remains a vLLM responsibility.
- Replacing vLLM’s existing `max_offload_tokens` API with a new store-control field.

## Related

- [[00 - Index]]
- [[01 - Initial implementation plan]]
- [[02 - vLLM primary-tier selective loading]]
- [[03 - llm-d-router selective load and offload policy and wiring]]
- [[vLLM and llm-d-router KV cache responsibility split]]
- [[vLLM KV offload retrieval path - lookup, promotion, and load]]
- [[vLLM KV offload lookup - worked example]]

## Provenance

This statement is based on direct inspection of `vllm-project/vllm@0e3ac4907d21e77cb4781338c49fef17bfea8f2b` and `llm-d/llm-d-router@b1bf63da5e9a52dc8815264809d00f45f5b5e966`, the current selective-loading project guides, and stakeholder direction captured on 2026-09-03. It describes required behavior and economic motivation; it does not claim measured break-even timings or a calibrated threshold value.