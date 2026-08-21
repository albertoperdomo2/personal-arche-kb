---
title: "ABC Version 3 — JIT demand-safe speculative prefetch"
date: "2026-08-21"
type: "research-index"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "v3.1-implemented-awaiting-continuous-batching-run"
---

# ABC Version 3

Version 3 changed the failed V2.1 broad admission-time promotion path into a bounded **just-in-time, demand-safe speculative prefetcher**. Version 3.1 now removes the `max_num_seqs=1` assumption and implements a continuous-batching-safe first experiment.

**Current verdict:** continue this direction. The v6 mechanism established that single-owner JIT promotion could make 50.0% of promoted chunks useful in its first AgentX comparison, versus 4.71% in the failed v5 retention-lease run, but that comparison did not establish a causal performance benefit. The v7 implementation is complete locally and ready to build; no v7 runtime result exists yet.

## Current documents

- [[02 - Continuous-Batching Remediation and v7 Implementation|02 — Continuous-Batching Remediation and v7 Implementation]] — **current Version 3.1 code and test reference:** batch-round lead-time model, fail-closed calibration, bounded eight-chunk I/O, demand-capacity sharing, immediate capacity-failure release, simplified configuration, verification evidence, and v7 experiment plan.
- [[01 - Current JIT Demand-Safe Speculative Prefetch Mechanism|01 — Current JIT Demand-Safe Speculative Prefetch Mechanism]] — detailed v6/single-sequence design reference. Its `demand_idle_only` configuration and serialized queue formula are historical and are superseded by Version 3.1 document 02.
- [[2026-08-21 - V3 JIT demand-safe AgentX comparison|2026-08-21 — V3 JIT demand-safe AgentX comparison]] — first v6 control/treatment comparison, mechanism accounting, validity analysis, and motivation for the continuous-batching remediation.
- [[../Version2/Reports/2026-08-20 - V2.1 retention-lease Weka failure|V2.1 retention-lease Weka failure]] — the v5 failure that motivated the Version 3 direction.
- [[../00 - Index|ABC project index]] — project history and run registry.

## Version 3.1 implementation status

### Provenance

- Repository: `/Users/aperdomo/workspace/redhat/vllm`
- Branch: `experimental/v2-admission-prefetch`
- Baseline HEAD: `c379bfdd5d717a3b3084097cefc77bb3a587bcb8`
- State: implementation exists as uncommitted working-tree changes
- Planned runtime image: `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v7`
- Image status: not built or pushed in this checkpoint

### Accepted Version 3.1 mechanism

1. **Batch-round lead time.** Queue depth is converted to admission rounds using `max_num_seqs`, observed scheduler occupancy, and EWMA admission batch size. The time horizon is admission rounds multiplied by EWMA time between admission rounds.
2. **Fail-closed calibration.** For `max_num_seqs > 1`, the old serialized timing seed cannot authorize speculative work. A real continuous-batching interval must be observed first.
3. **JIT single ownership.** At most one request owns speculative capacity, selected earliest-deadline-first.
4. **Bounded physical I/O.** One speculative filesystem call carries at most eight chunks, and only one slice may be outstanding.
5. **Demand-capacity sharing.** Demand remains queue-priority. Metadata lookup no longer creates a global idle-only veto, while the read pool reserves one worker for newly arriving demand when possible.
6. **Immediate failure release.** Capacity refusal releases ownership immediately. Failed or demanded-while-pending slices abort the unsubmitted remainder.
7. **Exact contiguous prefix.** Hole-tolerant downstream reuse is rejected because vLLM's prefix-chained hashes make later KV invalid after a changed block.
8. **One public mode.** `shadow_mode`, `jit_activation`, and `demand_idle_only` are removed. `prefetch.enabled=true` selects the one supported Version 3.1 behavior.

## Planned v7 treatment configuration

```json
{
  "enabled": true,
  "tier_idx": 0,
  "max_pending_bundles": 256,
  "max_promotions_per_step": 8,
  "max_bundle_chunks": 64,
  "max_candidate_chunks": 1024,
  "speculative_reserve_blocks": 512,
  "retention_lease_bundles": 1
}
```

The BenchFlow matrix has exactly two cells:

- reactive control: tiered CPU/NVMe offload with no `prefetch` block;
- treatment: identical deployment plus the configuration above.

Both use the same v7 image, `max_num_seqs=8`, AgentX concurrency 64, and request opt-in:

```json
{"ignore_eos": true, "kv_transfer_params": {"abc_admission_prefetch": true}}
```

BenchFlow requirement resolution raises the runtime model length to 131,072 tokens for the AgentX profile.

## Verification completed on August 21, 2026

- 70 policy tests passed.
- 117 combined policy, manager, and async-lookup tests passed.
- 127 tiering/filesystem tests passed; 7 skipped.
- 52 CPU offload-manager tests passed.
- 23 focused connector admission/occupancy tests passed; 117 unrelated tests deselected.
- Ruff check and format validation passed.
- Python compilation and `git diff --check` passed.
- BenchFlow experiment validation returned `valid`.
- Full RunPlan resolution confirmed the two cells share v7 and `max_num_seqs=8`; only treatment has prefetch enabled.

## Research questions for the next run

The v7 AgentX run must determine:

- whether the policy activates under sustained continuous batching;
- whether spare-worker gating avoids starvation without harming demand;
- whether eight-chunk I/O slices prevent observable head-of-line blocking;
- whether ownership always releases on capacity failure, demand, expiry, preemption, finish, and load failure;
- useful, late, failed, refused, and evicted-before-demand shares;
- TTFT, throughput, errors, running/waiting requests, GPU KV occupancy, CPU pressure, and CPU↔GPU transfer behavior;
- NVMe read/write bandwidth, IOPS, latency, queue depth, busy time, and capacity.

## Decision boundary

Continue Version 3.1 if it remains active under continuous load, preserves demand correctness and latency, and retains a materially useful speculative share with bounded storage overhead.

Change direction if the policy still starves despite spare workers, if bounded prefetch measurably degrades demand TTFT, or if exact-prefix opportunities are too sparse to cover the mechanism cost. Do not adopt session-ID or hole-tolerant KV reuse without a correctness proof that the downstream token prefix and attention state are unchanged.