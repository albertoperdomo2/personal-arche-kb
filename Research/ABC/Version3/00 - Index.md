---
title: "ABC Version 3 — JIT demand-safe speculative prefetch"
date: "2026-08-21"
type: "research-index"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "mechanism-accepted-performance-unproven"
---

# ABC Version 3

Version 3 changes the V2.1 live path from broad admission-time promotion into a bounded **just-in-time, demand-safe speculative prefetcher**. It keeps the evidence-backed V2 proposition—residency-verified contiguous session-prefix promotion when queue lead time can hide transfer latency—but narrows ownership and lifetime so speculative KV cannot dominate CPU capacity or filesystem service.

**Current verdict:** the v6 implementation mechanism is accepted. In its first AgentX comparison, 50.0% of promoted chunks became useful versus 4.71% in the failed v5 retention-lease run, and no lease reclamation or reserve borrowing occurred. Performance benefit is not yet established because the two nodes diverged during warmup before treatment submitted any speculative promotion.

## Documents

- [[2026-08-21 - V3 JIT demand-safe AgentX comparison|2026-08-21 — V3 JIT demand-safe AgentX comparison]] — first v6 control/treatment comparison, mechanism accounting, validity analysis, and next experiment.
- [[../Version2/Reports/2026-08-20 - V2.1 retention-lease Weka failure|V2.1 retention-lease Weka failure]] — the v5 failure that motivated this direction.
- [[../00 - Index|ABC project index]] — project history and run registry.

## Implementation record

### Provenance

- Repository: `/Users/aperdomo/workspace/redhat/vllm`
- Branch: `experimental/v2-admission-prefetch`
- Commit: `c379bfdd5d` (`fix: Change of direction`)
- Change size: 1,099 insertions and 114 deletions across 15 implementation/test files.
- Verification before image build: 294 tests passed, 7 skipped; Ruff and diff checks passed.
- Runtime image: `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v6`
- Immutable tested digest: `sha256:2d746cfe91ea5c47ffc635f2995d5696c066e94c0dc33185db16ccee8ad19033`

### What changed

1. **JIT activation and one active owner.** Admission still creates residency/deadline-aware bundles, but live promotion is activated only near demand. At most one request owns speculative residency. Eligible bundles are ordered earliest-deadline-first.
2. **Demand-idle submission.** With `demand_idle_only=true`, activation and submission yield whenever reactive demand work is present.
3. **Demand-priority filesystem service.** Async lookup, filesystem manager, and thread-pool scheduling distinguish demand from speculative work so demand lookup and loading are not queued behind prefetch.
4. **Explicit allocation contracts.** CPU allocation now distinguishes `DEMAND_CACHE`, `DEMAND_CRITICAL`, `SPECULATIVE_ONLY`, and `NONE`. Ordinary GPU→CPU persistence preserves the configured reserve; reactive demand may borrow it as a last resort; speculative allocation can reclaim only speculative victims.
5. **Owner-bound lifetime.** Speculative blocks are tied to the active owner. Ownership transition cleans up stale speculative state rather than allowing many admitted requests to accumulate promoted prefixes.
6. **Bounded retention lease.** `retention_lease_bundles=1` temporarily protects one completed promoted bundle from speculative replacement. Demand-critical allocation may break the lease, with separate accounting.
7. **Terminal accounting and telemetry.** The policy records considered, trimmed, redundant, gate-rejected, submitted, useful, wasted, evicted-before-demand, bundle states, active ownership, reserve/free/speculative/leased blocks, reserve borrowing, and lease reclamation.

### Files changed

Production paths:

- `vllm/v1/kv_offload/cpu/manager.py`
- `vllm/v1/kv_offload/tiering/async_lookup.py`
- `vllm/v1/kv_offload/tiering/base.py`
- `vllm/v1/kv_offload/tiering/fs/manager.py`
- `vllm/v1/kv_offload/tiering/fs/thread_pool.py`
- `vllm/v1/kv_offload/tiering/manager.py`
- `vllm/v1/kv_offload/tiering/prefetch/admission.py`
- `vllm/v1/kv_offload/tiering/prefetch/base.py`
- `vllm/v1/kv_offload/tiering/prefetch/config.py`

Focused tests cover CPU allocation/reserve/lease behavior, policy ownership and deadline ordering, manager integration, demand-priority async lookup, filesystem scheduling, and end-to-end configuration wiring.

## Tested configuration

The treatment used:

```json
{
  "enabled": true,
  "shadow_mode": false,
  "jit_activation": true,
  "demand_idle_only": true,
  "tier_idx": 0,
  "max_pending_bundles": 256,
  "max_promotions_per_step": 64,
  "max_bundle_chunks": 64,
  "max_candidate_chunks": 1024,
  "speculative_reserve_blocks": 512,
  "retention_lease_bundles": 1
}
```

Both cells used `max_num_seqs=1`, AgentX concurrency 64, one TP8 H100 replica, a 256 GiB CPU tier, the filesystem tier, and the same image digest. This is a mechanism-stress configuration, not yet a production concurrency recommendation.

## Design status

Accepted for continued research:

- JIT single-owner activation.
- Earliest-deadline-first bundle choice.
- Demand-idle-only speculative submission.
- Demand-priority I/O scheduling.
- Physical reserve preservation and explicit demand borrowing.
- Owner-bound cleanup, bounded lease, and exact accounting.

Not yet established:

- A causal TTFT or throughput benefit.
- The best reserve size; 512 blocks did not bind in this run.
- Robustness across nodes, repetitions, production `max_num_seqs`, and less saturated workloads.

The next step is not another redesign. Keep this mechanism, run a randomized cross-over with at least three repetitions, eliminate profiling cancellations, and capture GPU and device-specific storage telemetry. Only then tune the reserve or relax the single-sequence mechanism-test constraint.