---
title: "ABC Version 3 — JIT demand-safe speculative prefetch"
date: "2026-08-21"
type: "research-index"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "v7-conditionally-valid-severe-regression-next-diagnostic"
---

# ABC Version 3

Version 3 changed the failed V2.1 broad admission-time promotion path into a bounded **just-in-time, demand-safe speculative prefetcher**. Version 3.1 now removes the `max_num_seqs=1` assumption and implements a continuous-batching-safe first experiment.

**Current verdict:** reject the v7 treatment unchanged. In the first continuous-batching AgentX pair, prefetch reduced request throughput by 55.0%, reduced output-token throughput by 59.1%, increased mean TTFT by 125.8%, and increased mean ITL by 180.3%. This is a conditionally valid negative result: the mechanism was active and 48.1% of submitted chunks became useful, but one cross-node pair with missing GPU telemetry cannot determine whether the cause is policy wiring, active speculative work, or node variance. Continue only with the bounded causal diagnostic below.

## Current documents

- [[2026-08-21 - V3.1 continuous-batching AgentX v7 comparison|2026-08-21 — V3.1 continuous-batching AgentX v7 comparison]] — **current runtime verdict:** conditionally valid severe regression, mechanism accounting, resource evidence, validity limits, and three-arm v8 diagnostic plan.
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
- Evaluated runtime image: `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v7`
- Evaluated image digest: `sha256:890cd1760333ccbad3da63adb9df96f705230b6bc3be2ba635a09855f914e297`
- Image status: v7 built and evaluated; use a new immutable v8 tag for post-review corrections

### Accepted Version 3.1 mechanism

1. **Batch-round lead time.** Queue depth is converted to admission rounds using `max_num_seqs`, observed scheduler occupancy, and EWMA admission batch size. The time horizon is admission rounds multiplied by EWMA time between admission rounds.
2. **Fail-closed calibration.** For `max_num_seqs > 1`, the old serialized timing seed cannot authorize speculative work. A real continuous-batching interval must be observed first.
3. **JIT single ownership.** At most one request owns speculative capacity, selected earliest-deadline-first.
4. **Bounded physical I/O.** One speculative filesystem call carries at most eight chunks, and only one slice may be outstanding.
5. **Demand-capacity sharing.** Demand remains queue-priority. Metadata lookup no longer creates a global idle-only veto, while the read pool reserves one worker for newly arriving demand when possible.
6. **Immediate failure release.** Capacity refusal releases ownership immediately. Failed or demanded-while-pending slices abort the unsubmitted remainder.
7. **Exact contiguous prefix.** Hole-tolerant downstream reuse is rejected because vLLM's prefix-chained hashes make later KV invalid after a changed block.
8. **One public mode.** `shadow_mode`, `jit_activation`, and `demand_idle_only` are removed. `prefetch.enabled=true` selects the one supported Version 3.1 behavior.

## Evaluated v7 treatment configuration

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

## v7 run registry

| Role | Run | Disposition |
|---|---|---|
| Reactive control | [fc55e9b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/fc55e9b024a345c096cb29a63853559a?workspace=benchflow) | Accepted baseline |
| ABC prefetch | [68ecf349](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/68ecf349407f4b68a6f232e24e2ad2b6?workspace=benchflow) | Rejected treatment as configured; retain as diagnostic evidence |

## Next experiment: v8 causal diagnostic

Build the corrected implementation as immutable v8 and add policy-step, scheduler-hook, demand/prefetch I/O queue, owner/deferral, GPU SM/PCIe, image-digest, and cache-state telemetry. At `max_num_seqs=8`, run three same-node, counterbalanced arms with at least three repetitions:

1. Reactive control.
2. Prefetch configured but request opt-in removed.
3. Active prefetch with `max_bundle_chunks=8` and `max_promotions_per_step=8`.

Interpretation:

- If the idle arm regresses, isolate policy/manager wiring overhead.
- If idle matches but active regresses, isolate active lookup, I/O, or cache interference.
- If eight chunks is neutral but 64 is harmful, sweep 8/16/32.
- If the regression disappears on the same node, require a node crossover before making a mechanism claim.

## Decision boundary

Do not resume benefit tuning until active treatment keeps request/output throughput, mean and P95 TTFT, and mean and P95 ITL within 5% of control; shows no demand-read or scheduler-step regression; remains active with useful share above the failed V2.1 level; and reproduces across at least three repetitions plus a node crossover.

Abandon or redesign this implementation path if the instrumented same-node diagnostic reproduces the severe regression and attributes it to unavoidable active-path overhead. Do not adopt session-ID or hole-tolerant KV reuse without a correctness proof that the downstream token prefix and attention state are unchanged.