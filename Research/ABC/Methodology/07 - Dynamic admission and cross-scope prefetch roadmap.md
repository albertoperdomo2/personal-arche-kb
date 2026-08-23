---
title: "Dynamic admission and cross-scope prefetch roadmap"
date: "2026-08-23"
type: "research-design"
experiment: "Activity-Based KV Cache Tier Placement"
status: "proposed"
codebase: "vLLM v0.27.0 clean-prefetch worktree"
---

# Dynamic admission and cross-scope prefetch roadmap

## Decision

Make admission prefetch **model-neutral and request-dynamic**, but do not use adaptive chunk count as the research objective. Use bytes, deadline, complete-source readiness, destination capacity, and request-level readiness.

Current working-set mode adapts only to request length:

`target_chunks = min(request_chunks, max_candidate_chunks)`.

The 8,192-chunk ceiling, per-step width, batch width, tracking capacity, eviction budget, and one-owner rule remain static. The code also accepts only one full-attention, non-EAGLE KV group.

## Proposed local-vLLM policy

At admission, build the exact complete prompt working set and compute:

- `bytes_per_chunk` from the model-derived CPU offload specification;
- immutable original target bytes/chunks;
- current CPU-resident and source-resident contiguous coverage;
- estimated admission-to-first-lookup horizon;
- fitted secondary→CPU completion time;
- global in-flight byte/job capacity;
- per-request eviction-byte budget and expected regret.

For request $r$:

$$
B_{feasible}(r)=\min\left(
B_{missing}(r),
B_{bandwidth}(H_r-\epsilon),
B_{destination},
B_{global}
\right).
$$

In **request-ready** mode, submit only when the complete external working set is source-resident and deadline-feasible. Do not shrink the success denominator at the first source miss. Re-probe changing source state or mark the request ineligible.

In **contiguous-progress** mode, stage only the largest deadline-feasible contiguous prefix. This is a secondary baseline; recent AgentX evidence says partial coverage often does not remove deferral.

Rank eligible requests by earliest deadline and positive expected utility rather than one fixed owner:

$$
U(r)=\text{expected exposed stall removed}
-\text{I/O contention}
-\text{eviction regret}.
$$

Bound the policy by bytes and jobs, not model-specific chunk counts.

## Code mapping

- `vllm/v1/kv_offload/tiering/prefetch.py`: add an `auto` mode and byte/deadline budgets.
- `.../offloading/scheduler.py::_maybe_prefetch_on_admission()`: publish exact model-derived working-set geometry and scheduling horizon.
- `vllm/v1/kv_offload/tiering/manager.py`: retain immutable targets, select deadline-feasible plans, enforce global/per-request byte budgets, and re-probe pending source state.
- `vllm/v1/kv_offload/cpu/spec.py`: expose the already-derived `kv_bytes_per_chunk`.
- Existing `/v1/kv/prefetch`: retain as the advisory seam for signals that occur before request admission.

## Model portability

The first portable version should support standard single-group full-attention models. Hybrid, sliding-window, Mamba, and EAGLE layouts require separate working-set semantics and must fail closed until implemented.

Cross-vLLM reuse additionally requires an exact compatibility fingerprint: model and revision, tokenizer/prompt tokens, KV dtype/layout, TP/parallel geometry, block/chunk geometry, attention groups, and adapter identity.

## Four target scopes

| Scope | Admission-time policy sufficient? | Required addition |
|---|---|---|
| Complete cold working set inside one vLLM | Mostly | Strict source residency, model-derived bytes, complete-request deadline gate |
| Prefetch across vLLMs | No | Distributed KV directory, compatible source/target fingerprints, P2P transfer and routing/advisory control |
| Forecast hot/warm/cold | Partly | Reuse/deadline ranking outside the critical path; labels feed utility rather than trigger unconditional transfer |
| Across sessions/users | No | Session identity and prefix registry; tenant/privacy boundaries; earlier workflow/router event |

Cross-user reuse must be restricted to explicitly shareable, exact compatible prefixes. Private KV must not be exposed through cache-presence or transfer side channels.

## Required cross-model metrics

Use normalized bytes, tokens, time, and request outcomes—not raw chunk counts:

- target/source-ready/promoted/useful/evicted/regretted bytes;
- complete working-set ready and deferred requests;
- admission-to-first-lookup and predicted/actual ready time;
- reactive bytes and exposed stall avoided;
- demand versus speculative queue delay;
- CPU→GPU residual transfer;
- throughput, TTFT, ITL, errors, and preemptions.

## Experimental sequence

1. Run a true complete-source-residency oracle on two different single-group models.
2. Compare static working-set and dynamic request-ready modes with identical source snapshots.
3. Add multiple request owners only after complete readiness is achieved for one request.
4. Use natural AgentX tool/session events with the detached prefetch API; do not add an artificial sleep.
5. Only then evaluate cross-vLLM P2P placement and session/user forecasting.

Proceed only with at least 50% fewer externally deferred requests and either 5% lower mean/p95 TTFT or 3% higher throughput, after eviction regret and demand-I/O interference.