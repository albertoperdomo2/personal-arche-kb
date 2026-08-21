---
title: "ABC Version 3.1 — Continuous-Batching Remediation and v7 Implementation"
date: "2026-08-21"
type: "implementation-checkpoint"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
phase: "Version 3.1"
status: "implemented-locally-awaiting-image-and-agentx-validation"
codebase: "/Users/aperdomo/workspace/redhat/vllm"
branch: "experimental/v2-admission-prefetch"
baseline_commit: "c379bfdd5d717a3b3084097cefc77bb3a587bcb8"
planned_image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v7"
---

# ABC Version 3.1 — Continuous-Batching Remediation and v7 Implementation

## Executive conclusion

ABC should continue along the Version 3 demand-safe JIT direction, but the `max_num_seqs=1` implementation was not safe to extrapolate directly to continuous batching. Version 3.1 remediates the concrete problems that matter for `max_num_seqs > 1` without changing the research proposition or introducing a learned model.

The implementation is complete in the local vLLM worktree and configured for a first AgentX test at `max_num_seqs=8`. It has not yet been built into or validated as the v7 image, so there is no runtime-performance claim in this checkpoint.

The implemented mechanism is:

- batch-round-aware rather than serialized-position-aware;
- fail-closed until continuous-batching timing is measured;
- limited to one speculative request owner;
- limited to one eight-chunk filesystem read at a time;
- demand-priority with one read-worker lane reserved when possible;
- immediately aborting and releasing ownership on capacity failure;
- free of the obsolete `shadow_mode`, `jit_activation`, and `demand_idle_only` controls.

The direction was **not** changed to hole-tolerant downstream KV reuse. vLLM offload keys are prefix chained: changing block $N$ changes the valid prefix state for later blocks, so blocks $N+1\ldots$ cannot be treated as reusable KV for the changed request merely because their old files exist.

## 1. Scope and provenance

Repository:

- `/Users/aperdomo/workspace/redhat/vllm`
- branch `experimental/v2-admission-prefetch`
- baseline HEAD `c379bfdd5d717a3b3084097cefc77bb3a587bcb8`
- implementation currently exists as uncommitted working-tree changes

Experiment configuration:

- `/Users/aperdomo/workspace/redhat/benchflow/experiments/rhoai/abc.yaml`
- control deployment: `multi-tier-offloading-nvme`
- treatment deployment: `prefetch-cpu-kv-offload-nvme`
- planned image for both cells: `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v7`
- `max_num_seqs=8` in both cells
- AgentX concurrency remains 64

Request opt-in remains:

```yaml
extra_inputs: '{"ignore_eos": true, "kv_transfer_params": {"abc_admission_prefetch": true}}'
```

No additional request field is required. The deployment-level `prefetch.enabled` switch controls whether the policy exists; the request field marks requests whose ordered KV keys may be offered to it.

## 2. Continuous-batching lead-time model

### 2.1 Why the old model was invalid

The single-sequence model treated queue position as a count of serialized service intervals:

$$
H_{old} = q \cdot \bar{s}
$$

That overestimates the available time when several requests can enter the running batch together. A request at queue position 7 with eight free sequence slots does not have seven service intervals of lead time; it is eligible for the same next scheduling round.

### 2.2 Scheduler signals added

`ScheduleEndContext` now carries:

- `num_scheduled_reqs`: number of requests that received tokens in the scheduler step;
- `max_num_seqs`: configured scheduler sequence capacity.

`OffloadingConnectorScheduler` reads `scheduler_config.max_num_seqs` during initialization, configures the prefetch manager before requests are admitted, and forwards the current scheduled-request count at every schedule end.

Relevant code:

- `vllm/v1/kv_offload/base.py`
- `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`
- `vllm/v1/kv_offload/tiering/manager.py`

### 2.3 Admission-round estimator

Let:

- $M$ be `max_num_seqs`;
- $A$ be the most recently observed number of scheduled sequences;
- $F = \max(0, M-A)$ be the estimated free sequence slots;
- $q$ be the zero-based queue position at enqueue;
- $\bar{b}$ be the EWMA number of new requests admitted per admission round;
- $\bar{t}$ be the EWMA interval between admission rounds.

Requests that fit the estimated free slots receive no speculative horizon:

$$
q < F \Rightarrow H=0
$$

For a request behind those free slots:

$$
r = \left\lceil \frac{q-F+1}{\bar{b}} \right\rceil
$$

$$
H = r \cdot \bar{t}
$$

The model tracks batch admission size and admission-round spacing, not per-request completion time. Admission-to-first-schedule lead divided by predicted rounds is used only to bootstrap the first interval sample; once two real admission rounds exist, direct wall-clock spacing between those rounds is authoritative.

### 2.4 Fail-closed cold start

For `max_num_seqs > 1`, the previous 1,450 ms serialized seed is not accepted as continuous-batching evidence. Until a real admission-round interval is observed, predicted lead time is zero even when the estimator can calculate a nonzero round count.

This deliberately sacrifices early opportunities rather than authorizing speculative I/O from an invalid horizon.

### 2.5 Conservative occupancy limitation

`num_scheduled_reqs` is a scheduler-output proxy for occupied sequence slots. If token-budget constraints leave a running request resident but unscheduled in a particular step, the proxy can undercount occupancy and appear to expose free slots. In this implementation, apparent free slots produce a zero horizon for the requests that fit them, so this error is conservative: it suppresses prefetch rather than inflating lead time.

## 3. Bounded speculative filesystem I/O

### 3.1 Micro-batch bound

`max_promotions_per_step` now means the maximum number of chunks in one speculative filesystem read. Its default and v7 experiment value are both 8.

A candidate bundle may contain up to 64 chunks, but only one slice may be outstanding. After a slice completes, the scheduler rechecks:

- whether demand-safe filesystem capacity exists;
- whether the request deadline still has enough margin;
- whether the previous slice failed or was demanded before completion;
- whether primary speculative capacity is still available.

Only then may it submit the next slice.

This bounds non-preemptible storage work. With the AgentX configuration, an individual speculative call is at most eight chunks rather than the previous 64-chunk call.

### 3.2 Demand-priority queueing

Filesystem loads retain separate demand and prefetch queues. Worker selection always chooses queued demand reads before queued speculative reads.

An I/O call already executing cannot be preempted; the eight-chunk bound is therefore the physical preemption granularity.

### 3.3 Spare-worker gate

The filesystem pool permits one prefetch slice only when:

- no demand load is queued; and
- active demand tasks leave capacity beyond a demand reserve.

When there is more than one read worker, one read lane is kept available for newly arriving demand. A single-reader deployment may prefetch only while demand is idle; the bounded slice limits how long subsequent demand can wait.

The AgentX treatment config has 64 read threads, so one lane is reserved and at most one speculative slice occupies the remaining pool.

### 3.4 Metadata lookup no longer recreates idle-only starvation

Demand and speculative residency probes already use a strict-priority asynchronous lookup queue. Version 3.1 therefore does not make the existence of demand metadata lookup work a global submission veto.

The old binary gate could starve speculation forever under continuous admissions even while data-read workers were available. Demand lookup still bypasses queued speculative lookup batches; it simply does not disable all prefetch data movement.

Relevant code:

- `vllm/v1/kv_offload/tiering/async_lookup.py`
- `vllm/v1/kv_offload/tiering/fs/thread_pool.py`
- `vllm/v1/kv_offload/tiering/fs/manager.py`
- `vllm/v1/kv_offload/tiering/manager.py`

## 4. Ownership and failure semantics

### 4.1 One speculative owner

At most one request owns speculative CPU capacity. When there is no owner, the policy selects the queued bundle with the earliest absolute demand deadline.

The owner remains exclusive while lookup or one bounded I/O slice is active. A second bundle cannot begin while the manager still has a speculative job or pending speculative submission.

### 4.2 Immediate capacity-failure release

If demand borrows enough of the speculative reserve that a later slice receives `capacity_skipped`, Version 3.1:

1. classifies the refused tail as `alloc_refused`;
2. stops the contiguous run;
3. releases speculative ownership immediately;
4. parks only already-issued I/O until it drains; and
5. terminates the bundle as `CAPACITY_SKIPPED`.

The mechanism no longer waits behind a bundle that cannot complete its contiguous prefix.

### 4.3 Failed slice aborts the remainder

If a speculative filesystem load fails, later chunks in the candidate are not submitted. The already-submitted keys remain classified as submitted and the unsubmitted resolved tail is cancelled. The bundle outcome is `failed`, ownership is released, and primary failed-write cleanup remains the manager's responsibility.

### 4.4 Demand reaching an in-flight slice aborts the remainder

If a primary lookup observes `HIT_PENDING` for a key owned by an in-flight prefetch, the bundle is marked as demanded while pending. Once that slice drains, Version 3.1 stops rather than submitting more chunks. The remaining unsubmitted tail is gate-rejected and the bundle outcome is `late`.

This prevents continuous batching from amplifying speculative work after the overlap window has already been missed.

## 5. Exact-prefix rule remains mandatory

The proposed remediation text suggested tolerating holes and reusing later blocks through a session identifier. That is not implemented because it would violate current vLLM KV correctness.

The offload key for a block belongs to a prefix-chained hash sequence. A changed token block changes the valid attention state for all later positions. Even if files from the old session path remain on disk, their KV tensors are not valid for the altered prefix.

The current policy therefore preserves these rules:

- primary scanning stops at the first primary miss and starts the candidate there;
- secondary interpretation stops at the first absent key;
- allocation refusal closes the later tail;
- later disconnected blocks are never promoted as reusable KV.

Agent-aware reuse across dynamic metadata would require an upstream semantic transformation that proves an unchanged token/KV prefix or isolates mutable metadata outside the cached prefix. A session ID alone is not sufficient.

## 6. Simplified runtime configuration

The public prefetch configuration no longer accepts:

- `shadow_mode`;
- `jit_activation`;
- `demand_idle_only`.

Enabling ABC now always means the one supported Version 3.1 behavior: JIT, single-owner, deadline-gated, demand-priority, bounded-slice speculative promotion.

The planned treatment configuration is:

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

Unknown legacy keys fail startup validation rather than silently selecting a stale mode.

## 7. Hot-path work reduction

The implementation includes the following low-risk performance changes:

- slotted `PrefetchConfig`, `Bundle`, `QueuedRequest`, `PendingPromotion`, and transfer-cost model objects;
- one drivable owner instead of allocating and sorting a list of active bundles each scheduler step;
- no redundant active-owner gauge write on every scheduler step; the gauge changes only on ownership transitions;
- no per-completion request-ID set allocation for the normal one-job/one-request path;
- bounded candidate, bundle, residency-probe, and I/O sizes;
- no synchronous secondary filesystem lookup on the scheduler thread.

The earliest-deadline activation scan remains $O(B)$ over at most `max_pending_bundles` live bundles. With the configured bound of 256 and only one owner, this is intentionally simpler than maintaining a mutable heap with stale-entry cleanup.

## 8. Files changed for Version 3.1

Production paths:

- `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`
- `vllm/v1/kv_offload/base.py`
- `vllm/v1/kv_offload/tiering/base.py`
- `vllm/v1/kv_offload/tiering/fs/manager.py`
- `vllm/v1/kv_offload/tiering/fs/thread_pool.py`
- `vllm/v1/kv_offload/tiering/manager.py`
- `vllm/v1/kv_offload/tiering/prefetch/admission.py`
- `vllm/v1/kv_offload/tiering/prefetch/base.py`
- `vllm/v1/kv_offload/tiering/prefetch/config.py`
- `vllm/v1/kv_offload/tiering/prefetch/estimators.py`
- `vllm/v1/kv_offload/tiering/spec.py`

Test paths:

- `tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py`
- `tests/v1/kv_offload/tiering/test_admission_prefetch_manager.py`
- `tests/v1/kv_offload/tiering/test_admission_prefetch_policy.py`
- `tests/v1/kv_offload/tiering/test_fs_tier.py`
- `tests/v1/kv_offload/tiering/test_tiering_offloading.py`

Build and experiment paths:

- `Containerfile.vllm-prefetch`
- `/Users/aperdomo/workspace/redhat/benchflow/experiments/rhoai/abc.yaml`
- `/Users/aperdomo/workspace/redhat/benchflow/profiles/deployment/rhoai/prefetch-cpu-kv-offload-nvme.experimental.yaml`

## 9. Verification evidence

Executed on August 21, 2026:

- 70 policy tests passed;
- 117 combined policy, manager, and async-lookup tests passed;
- 127 tiering/filesystem tests passed and 7 were skipped;
- 52 CPU offload-manager tests passed;
- 23 focused connector admission/occupancy tests passed, with 117 unrelated tests deselected;
- Ruff checks passed;
- Ruff formatting checks passed;
- Python `compileall` passed;
- `git diff --check` passed in vLLM and BenchFlow;
- `bflow experiment validate experiments/rhoai/abc.yaml` returned `valid`;
- full RunPlan resolution produced exactly two cells with the same v7 image and `--max-num-seqs=8`; only the treatment contains the `prefetch` block.

The full connector scheduler file was not used as a release gate because many unrelated tests require external Hugging Face model metadata. An earlier attempt produced predominantly `httpcore.ConnectError` failures in that dependency path. The focused offline connector tests covering the modified wiring pass.

## 10. Planned first continuous-batching experiment

Both cells:

- same AgentX profile and pinned Weka corpus;
- same v7 image;
- same TP8 Nemotron deployment;
- same 256 GiB CPU offload tier;
- same local NVMe tier with 64 read and 64 write threads;
- `max_num_seqs=8`;
- AgentX concurrency 64;
- same request opt-in.

Only the treatment adds the Version 3.1 prefetch block.

The first experiment should answer mechanism questions before parameter tuning:

1. Does prefetch continue to activate under sustained continuous batching, or do demand-capacity deferrals dominate?
2. Are speculative reads bounded to eight chunks with no overlapping speculative jobs?
3. Does `capacity_skipped` release ownership immediately?
4. What share of submitted chunks is useful, late, failed, or evicted before demand?
5. Does treatment preserve or improve TTFT without reducing request throughput or increasing errors?
6. Do NVMe read bandwidth, IOPS, latency, queue depth, and busy time show bounded incremental work rather than saturation?

## 11. Current limitations

- There is no v7 AgentX result yet; the implementation is not a performance result.
- One speculative owner is deliberately conservative and may underuse storage bandwidth.
- The admission deadline is fixed from the enqueue-time estimate rather than recomputed for every queue mutation.
- Occupancy is inferred from scheduler output rather than a direct running-queue size signal.
- Secondary residency is exact and prefix contiguous; dynamic metadata inside the cached prefix still breaks reuse.
- Filesystem-to-CPU movement does not use CUDA streams because this path is not a GPU transfer path.
- Eight chunks is a safe first micro-batch, not an established optimum.
- The 512-block reserve and one-bundle lease remain experimental bounds.

## 12. Decision rule after the v7 run

Continue this direction if the treatment demonstrates all of the following:

- the policy activates under continuous load;
- demand work retains priority and no correctness failure appears;
- useful-prefetch share remains materially above the failed v5 behavior;
- TTFT or queueing improves, or at minimum remains neutral while mechanism evidence is sound;
- storage and CPU telemetry show bounded overhead.

Reconsider the direction if prefetch remains starved despite spare workers, if demand TTFT regresses from storage interference, or if exact-prefix reuse opportunities are too rare to cover the mechanism cost. Those outcomes would justify changing the trigger or workload target; they would not justify unsafe hole-tolerant KV reuse.