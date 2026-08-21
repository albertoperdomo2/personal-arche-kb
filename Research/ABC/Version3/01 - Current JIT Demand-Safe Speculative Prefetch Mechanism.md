---
title: "ABC Version 3 — Current JIT Demand-Safe Speculative Prefetch Mechanism"
date: "2026-08-21"
type: "implementation-reference"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
phase: "Version 3"
status: "current-code-reference"
codebase: "/Users/aperdomo/workspace/redhat/vllm"
branch: "experimental/v2-admission-prefetch"
commit: "c379bfdd5d717a3b3084097cefc77bb3a587bcb8"
image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v6"
---

# ABC Version 3 — Current JIT Demand-Safe Speculative Prefetch Mechanism

## Purpose and scope

This article is the standalone description of the speculative-prefetch mechanism implemented in the current ABC Version3 vLLM branch. It explains both:

1. the theory and constraints that justify the design; and
2. the concrete implementation, state machine, allocation contracts, I/O scheduling, configuration, telemetry, and lifecycle behavior in the code.

The code ground truth is commit `c379bfdd5d717a3b3084097cefc77bb3a587bcb8` on branch `experimental/v2-admission-prefetch` in `/Users/aperdomo/workspace/redhat/vllm`. The tested container is `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-v6`, digest `sha256:2d746cfe91ea5c47ffc635f2995d5696c066e94c0dc33185db16ccee8ad19033`.

This is an experimental fork, not an upstream vLLM feature.

## One-paragraph description

The mechanism observes a request when it enters vLLM, constructs its ordered KV-cache keys, removes the leading keys already present or loading in the CPU tier, and records a bounded candidate bundle beginning at the first CPU miss. In JIT mode it does **not** immediately probe storage or move data. Instead, it waits until demand and earlier speculative work are idle, chooses the queued bundle with the earliest predicted demand deadline, gives that request exclusive ownership of speculative capacity, asynchronously verifies a contiguous prefix on the configured secondary tier, and promotes that verified prefix from filesystem storage into reserved CPU KV blocks only if the remaining queue lead time still exceeds the measured transfer cost. Demand lookup and reactive loading always have higher service priority. Promoted blocks receive a bounded retention lease until their owner reaches demand, expires, finishes, fails, or is preempted. Demand converts useful speculative blocks into ordinary cache data; unused blocks remain first-class reclamation victims.

## 1. The problem being solved

### 1.1 Reactive retrieval puts secondary-tier latency on TTFT

The ordinary tiered-offload path is reactive. When a running request reaches a KV block absent from GPU and CPU but resident on the filesystem tier, vLLM must:

1. discover the secondary-tier hit;
2. allocate a destination CPU block;
3. read the KV data from the filesystem tier into CPU memory;
4. observe transfer completion; and
5. make the block available to the running request.

That work is on the request's critical path. For long agentic prefixes, repeated reactive promotion can dominate time to first token.

### 1.2 Agentic workloads can expose an overlap window

A queued request may be known to vLLM before it is first scheduled. If a reusable KV prefix exists on the secondary tier and the request will wait long enough, secondary-to-CPU movement can overlap queueing instead of extending TTFT.

The useful overlap condition is:

$$
H_{remaining} > L_{promotion}(n)
$$

where:

- $H_{remaining}$ is the predicted time until the request first needs the KV prefix; and
- $L_{promotion}(n)$ is the predicted end-to-end promotion time for the remaining $n$ chunks.

Prefetch is useful only if the data is resident, the prefix is actually reusable, the transfer completes before demand, the destination survives until demand, and speculative work does not delay the active workload.

### 1.3 Why the mechanism is deterministic and does not need fine-tuning

The current policy does not use a learned predictor. It uses signals already available in the serving stack:

- queue position;
- observed first-schedule service interval;
- exact primary-tier lookup state;
- asynchronous secondary-tier residency results;
- ordered prefix structure;
- measured promotion duration;
- demand activity; and
- physical CPU-cache capacity.

The two numerical models—the queue lead-time estimate and transfer-cost estimate—adapt online from serving observations. Configuration exposes bounds and experimental controls, not trainable weights. This keeps the mechanism usable out of the box while preserving knobs for research.

The broader research motivation and the corrections that led to this proposition are recorded in [[../Version2/04 - Theoretical Validation|Version2 theoretical validation]]. The current implementation is specifically a residency/deadline admission mechanism; it is not yet the true tool-start or handoff-event mechanism proposed for a later phase.

## 2. Theory and design principles

### 2.1 Ordered contiguous prefixes are the policy unit

KV offload keys are derived from prefix-chained request block hashes. A demand lookup consumes the reusable prefix in order and stops at the first missing block. Therefore, speculative value is not additive over arbitrary independent keys.

The policy selects a contiguous run:

$$
B = [k_m, k_{m+1}, \ldots, k_{m+r-1}]
$$

where $k_m$ is the first key not already present or loading in CPU, and every selected key is confirmed resident on the secondary tier. If the secondary lookup finds an absent key, the run ends there. Keys after the first absent key are not useful to the current prefix scan and are dropped rather than promoted as disconnected fragments.

This property is why allocation refusal also closes the remaining tail: allowing a hole and promoting later blocks would spend capacity on data the demand scan cannot reach.

### 2.2 Primary-residency frontier

At admission, the policy scans at most `max_candidate_chunks` ordered keys. It advances a frontier while CPU lookup returns either:

- `HIT`: ready in CPU; or
- `HIT_PENDING`: already being written into CPU.

Those keys are terminally classified as `primary_redundant`. The candidate bundle begins at the first CPU `MISS`.

This avoids the dominant V1 error: blindly prefetching the earliest chunks even though shared conversational prefixes were commonly already in CPU.

### 2.3 Queue-derived demand horizon

For a newly queued request, the lead-time estimator predicts:

$$
H = q \cdot \bar{s}
$$

where:

- $q$ is the request's queue position when observed; and
- $\bar{s}$ is an exponentially weighted moving average of elapsed time per first-scheduled request.

The implementation initializes $\bar{s}$ from `initial_admission_interval_ms` and updates it from scheduler batches. Requests first scheduled in one batch contribute one batch observation rather than artificial zero-time intervals. When the queue becomes idle, the sample chain is broken so idle time does not contaminate the next service observation.

Admission fixes an absolute deadline:

$$
d = t_{admit} + H
$$

Every later activation and submission decision uses:

$$
H_{remaining} = d - t_{now}
$$

The deadline does not slide forward when work is delayed. This prevents a stalled speculative task from continually justifying itself with a renewed horizon.

### 2.4 Measured transfer-cost model

Promotion cost is modeled as:

$$
L_{promotion}(n) = b + p n
$$

where:

- $b$ is the measured fixed cost per promotion job;
- $p$ is measured marginal cost per chunk; and
- $n$ is the remaining contiguous run.

`TransferCostModel` fits $b$ and $p$ using decayed weighted least squares over real completed secondary-to-primary transfers. Reactive demand promotions feed the same model, so the estimate can calibrate even while the prefetch policy is in shadow mode.

The configured `transfer_base_ms` and `transfer_per_chunk_ms` are seeds. The fit is used after at least 20 decayed samples with enough batch-size variance to identify a positive slope. The fitted intercept includes scheduler completion-detection delay, not just device latency. That is intentional for the gate because a promotion is not useful until the scheduler observes completion.

A busy or degraded tier naturally reports longer transfers. The estimate rises and the deadline gate becomes more conservative without a separate learned contention model.

### 2.5 JIT activation minimizes speculative lifetime

The first V2 implementation began residency lookup at admission. Later retention experiments showed that broad early promotion creates two conflicting failure modes:

- without protection, promoted data is reclaimed before demand; and
- with broad protection, speculative data occupies capacity for too long and can interfere with demand.

Version3 separates **candidate discovery** from **activation**:

- admission records a bounded bundle and deadline;
- JIT activation starts lookup only when the policy can give one request exclusive ownership and the tier has no demand or older speculative work;
- the earliest-deadline queued request is selected first.

This reduces the interval between allocation and expected use and limits speculative occupancy to one request's active bundle.

### 2.6 Earliest-deadline-first ownership

When no request owns speculative work, the policy selects the `QUEUED` bundle with the smallest absolute deadline. This is earliest-deadline-first selection, not FIFO admission order.

The intent is narrow: keep residency close to likely demand and avoid retaining a farther-future request while an earlier deadline waits. It is not a claim that EDF is globally optimal for every serving workload.

### 2.7 Demand-safe resource hierarchy

The design enforces a strict priority order:

1. demand lookup and reactive load;
2. demand-critical CPU allocation;
3. ordinary GPU-to-CPU cache persistence;
4. speculative lookup, load, and retention.

Speculation is an optimization. A running request must be able to override it.

## 3. Request eligibility and trigger

Prefetch is opt-in per request. The scheduler hook runs only when:

```json
{
  "kv_transfer_params": {
    "abc_admission_prefetch": true
  }
}
```

The value must be the Boolean `true`, not a truthy string or number.

The scheduler also requires:

- the tiering manager to expose an enabled prefetch policy;
- exactly one KV-cache group;
- a stable full-attention group;
- no sliding-window group; and
- no Eagle group.

If these conditions fail, the request is skipped. The policy and the legacy V1 `admission_prefetch_chunks` mechanism are mutually exclusive in configuration.

The hook builds the request's full ordered offload-key list and passes it to `TieringOffloadingManager.prefetch_on_admission()`. Candidate-window and bundle limits are applied inside the policy.

## 4. End-to-end lifecycle

### Stage 0 — request enters the manager

`TieringOffloadingManager.on_new_request()` records the request in the policy's unscheduled queue. The queue record stores admission time and queue position.

### Stage 1 — scheduler constructs keys and applies the request gate

`OffloadingConnectorScheduler._maybe_prefetch_on_admission()` checks `abc_admission_prefetch`, validates the KV group, constructs ordered offload keys, and calls the policy.

### Stage 2 — candidate bundle is created

`AdmissionPrefetchPolicy.on_request_admitted()`:

1. predicts lead time from queue position;
2. scans the CPU-resident frontier;
3. rejects a disallowed secondary tier;
4. enforces `max_pending_bundles`;
5. bounds the bundle to `max_bundle_chunks`;
6. counts the remaining candidate tail as `bundle_trim`; and
7. creates one `Bundle` for the request.

With `jit_activation=true`, the bundle enters `QUEUED`. No secondary lookup is issued at this point.

### Stage 3 — schedule-end processing gives demand the first turn

On each scheduler step, `TieringOffloadingManager.on_schedule_end()` performs this order:

1. process completed transfer jobs;
2. serve external requests;
3. drive the prefetch policy;
4. flush pending promotion submissions; and
5. flush each secondary tier's asynchronous lookup batch.

Because policy execution happens after demand processing but before the step flush, speculative work can join the current step only after demand has already had its opportunity.

### Stage 4 — one bundle is activated

If there is no active owner, `_activate_next_bundle()` checks:

- no previous speculative transfer/submission is still active;
- no demand lookup or reactive load is queued or active when `demand_idle_only=true`;
- the earliest bundle's deadline has not expired; and
- remaining horizon exceeds predicted transfer cost for the bundle.

If accepted, the request becomes `_owner_req_id`, its deadline becomes `_owner_deadline`, and the bundle transitions from `QUEUED` to `PENDING_LOOKUP`.

### Stage 5 — asynchronous secondary residency is resolved

Every bundle key is offered to `lookup_prefetch()`, which uses the secondary tier's low-priority lookup queue. Results can be:

- `HIT`: extend the contiguous resident run;
- `MISS`: terminate the run at the first absent key; or
- `RETRY`: keep the bundle pending for a later scheduler step.

The policy re-drives unresolved lookup state on later steps. It never blocks the scheduler waiting for filesystem metadata.

### Stage 6 — deadline gate is re-evaluated

Once the bounded lookup resolves, `_drive_bundle()` calculates remaining horizon and predicted cost for the entire remaining run. It records:

$$
margin = H_{remaining} - L_{promotion}(n)
$$

If the margin is non-positive, the bundle is rejected or classified late. If positive, it may submit a slice bounded by the current `max_promotions_per_step` budget.

The cost gate is rechecked on later slices. A large bundle is carried across scheduler steps rather than silently trimmed by the global step budget.

### Stage 7 — speculative CPU allocation and batched filesystem load

For each submitted key, the manager rechecks CPU residency. A new CPU hit or pending write becomes `primary_redundant` rather than a duplicate promotion.

A CPU miss invokes `_initiate_promotion()` with:

- `is_prefetch=true`;
- `AllocationMode.SPECULATIVE_ONLY`; and
- the request ID as speculative owner.

If allocation succeeds, the manager batches keys by secondary tier and request. `on_schedule_end()` submits one filesystem load job for the batch instead of one job per chunk.

If allocation refuses one key, submission stops and the remaining tail is classified `alloc_refused` to preserve contiguity.

### Stage 8 — completion, speculative marking, and lease

When a promotion job completes successfully:

1. CPU blocks become ready;
2. blocks not already demanded in flight are marked speculative;
3. those blocks receive the bounded retention lease;
4. legacy useful/late/wasted tracking is updated; and
5. the policy is notified through `on_promotion_finished()`.

A bundle becomes:

- `READY` if all submitted keys finished before demand;
- `LATE` if demand reached a key while it was pending;
- `FAILED` if the load failed; or
- `SUBMITTING` if more of the verified run remains for a later slice.

`READY` is terminal for bundle processing, but JIT ownership intentionally remains active. This keeps its CPU ownership and lease until the request reaches demand or another explicit release condition occurs.

### Stage 9 — demand or cleanup releases ownership

When the owner is first scheduled, the policy calls `prefetch_release_owner(..., demanded=true)`. Its speculative blocks become ordinary demand-owned cache data and stop counting against speculative reserve.

Ownership is also released on:

- deadline expiry;
- request finish;
- preemption;
- promotion failure;
- cancellation;
- cache reset; or
- any non-ready terminal state.

If released without demand, blocks lose their lease but remain speculative and reclaimable. If a transfer completion arrives after owner release, `_released_prefetch_owners` prevents that completion from recreating ownership or a lease.

## 5. Bundle state machine

| State | Meaning | Typical next states |
|---|---|---|
| `QUEUED` | JIT candidate exists; no secondary lookup has started | `PENDING_LOOKUP`, `GATE_REJECTED`, `LATE`, `CANCELLED` |
| `PENDING_LOOKUP` | Low-priority secondary residency is unresolved | `RESIDENT`, `ABSENT`, `LATE`, `CANCELLED` |
| `RESIDENT` | A non-empty contiguous secondary-resident run is known | `SUBMITTING`, `SUBMITTED`, `REDUNDANT`, `CAPACITY_SKIPPED`, `GATE_REJECTED` |
| `SUBMITTING` | Some run keys have been dispositioned; more remain | `SUBMITTED`, `READY`, `LATE`, `FAILED`, `CAPACITY_SKIPPED` |
| `SUBMITTED` | Promotions are in flight | `READY`, `LATE`, `FAILED`, `CANCELLED`, `SUBMITTING` |
| `READY` | Promotion completed before demand | ownership retained until demand/cleanup |
| `LATE` | Demand deadline or demand arrival beat completion | terminal |
| `ABSENT` | No usable contiguous secondary-resident run | terminal |
| `FAILED` | Promotion job failed | terminal |
| `GATE_REJECTED` | Remaining horizon did not exceed predicted transfer cost | terminal |
| `CAPACITY_SKIPPED` | Speculative CPU allocation refused | terminal |
| `REDUNDANT` | Recheck found all relevant keys already in CPU/loading | terminal |
| `SHADOW_SUBMITTED` | Shadow policy would have submitted the run | terminal |
| `CANCELLED` | Finish, preemption, reset, or abandonment closed the bundle | terminal |

State-transition counters are separate from per-key terminal accounting, so asynchronous retries and multiple slices do not double-count considered keys.

## 6. CPU allocation contracts

The CPU tier now exposes four explicit allocation modes.

| Mode | Caller | May use free non-reserved capacity | May reclaim speculative blocks | May evict ordinary demand data | May borrow unused reserve |
|---|---|---:|---:|---:|---:|
| `DEMAND_CACHE` | ordinary GPU→CPU cache persistence | yes | yes, but respects active lease | yes | no |
| `DEMAND_CRITICAL` | reactive secondary→CPU load for a running request | yes | yes; may break lease as last resort | yes | yes, as last resort |
| `SPECULATIVE_ONLY` | proactive prefetch | only within unused reserve | yes, but only speculative victims and respecting lease | no | no |
| `NONE` | strictly free-only caller | yes | no | no | no |

### 6.1 Physical reserve

`speculative_reserve_blocks` is implemented as a physical allocation watermark, not merely a logical counter.

For demand allocations, allocatable free capacity is:

$$
free_{demand} = free_{raw} - reserve_{unused}
$$

The value is allowed to become negative. That negative shortfall forces ordinary demand-cache allocation to evict enough of its own cache data to restore the reserve watermark after allocation.

This is the correction for the earlier failure where metrics reported unused reserve while the physical free-block count was zero.

### 6.2 Reserve bound

The CPU manager clamps speculative reserve to at most 25% of the CPU KV pool. If the effective reserve is smaller than `max_bundle_chunks`, the manager caps the bundle size to the effective reserve so a bundle cannot recycle and evict its own leading prefix while it is still being constructed.

If no explicit reserve is configured, the manager derives:

$$
reserve = \max(max\_bundle\_chunks, \lfloor pool\_blocks / 64 \rfloor)
$$

subject to the 25% ceiling.

### 6.3 Speculative provenance

Speculative blocks are entered in an insertion-ordered `_speculative` ledger at allocation time, not completion time. An in-flight store already consumes reserved capacity, so delaying accounting until readiness would over-allocate the reserve.

A demand touch through CPU `prepare_load()`, `touch()`, or `mark_demanded()` removes the key from speculative and lease ledgers. From that moment it is ordinary useful cache data.

### 6.4 Reclamation order

Demand allocations reclaim eligible speculative blocks before asking the configured LRU/ARC policy to evict ordinary cache data. Speculative allocations may reclaim only speculative blocks.

This gives speculative data deliberately weaker eviction rights than proven-demand data.

## 7. Single-owner semantics

The policy and CPU allocator both track the active owner.

- Policy: `_owner_req_id` and `_owner_deadline`.
- CPU manager: `_active_speculative_owner` and `_speculative_owner_by_key`.
- Transfer manager: owner ID stored with each prefetch job.

A speculative allocation with a different owner is refused while an owner is active. This prevents concurrent requests from filling the reserve with unrelated bundles.

Ownership is claimed at allocation time because in-flight blocks already consume capacity. The policy releases it only at demand or an explicit terminal condition.

## 8. Retention lease

The lease solves a specific failure: speculative blocks are preferred reclamation victims, so a successful promotion can be evicted before the queued request arrives.

`retention_lease_bundles` is converted to a block budget:

$$
lease_{blocks} = retention\_lease\_bundles \cdot max\_bundle\_chunks
$$

and clamped to the speculative reserve.

Only ready blocks that are still speculative and belong to the active owner are eligible. Demand releases the lease immediately. A newer anonymous lease or owner transition cannot make the lease grow without bound.

The lease is not absolute:

- `DEMAND_CACHE` respects it and declines rather than consuming it;
- `SPECULATIVE_ONLY` respects it;
- `DEMAND_CRITICAL` may break it if a running request otherwise cannot progress.

Lease breaks are counted separately as `kv_offload_cpu_cache_speculative_lease_reclaimed_blocks` because they represent demand pressure overriding retention, not spontaneous prefetch waste.

## 9. Demand-priority lookup and filesystem I/O

Capacity safety alone is insufficient. Speculative work could still increase TTFT by occupying lookup and I/O queues.

### 9.1 Asynchronous lookup priority

`AsyncLookupManager` has two service classes:

- `DEMAND = 0`;
- `PREFETCH = 1`.

A priority queue orders every queued demand batch before speculative batches. An already running filesystem call cannot be preempted, but later demand bypasses all speculative batches that have not started.

If demand asks for a key whose speculative lookup is pending:

- an unflushed key is moved from the prefetch batch to the demand batch; or
- a demand-priority duplicate existence check is queued if the speculative check is already queued/in flight.

Duplicate existence checks are safe and avoid making the running request wait behind unrelated speculative lookup batches.

### 9.2 Filesystem load priority

`DualQueueThreadPool` separates:

- demand-load queue;
- prefetch-load queue; and
- store queue.

Read-priority workers choose demand load, then prefetch load, then store. Write-priority workers choose store, then demand load, then prefetch load. Thus prefetch is last within both worker classes.

`prefetch_demand_idle()` returns false while demand lookup or demand load work is queued or active. With `demand_idle_only=true`, policy activation and additional submission defer until that condition clears.

This is cooperative priority, not hard real-time isolation: an I/O operation already executing is not cancelled.

## 10. Submission budgets and batching

Three limits serve different purposes:

- `max_candidate_chunks`: how far into the request key list admission inspects;
- `max_bundle_chunks`: maximum per-request bundle and residency-probe size; and
- `max_promotions_per_step`: global number of chunks the policy may submit in one scheduler step.

The per-bundle ceiling trims at admission and records `bundle_trim`. The per-step budget does **not** trim; it carries the remaining verified prefix to later steps and rechecks the deadline.

Promotion keys are accumulated by secondary tier and request and submitted as one batched load job per request/tier at schedule end. This is necessary for the transfer-cost model and avoids per-chunk job overhead.

## 11. Shadow mode

`shadow_mode=true` runs candidate construction, residency lookup, deadline gating, state transitions, and per-step budgets but does not allocate CPU blocks or submit transfers.

Shadow submission drains in the same slices as live mode and rechecks the deadline between slices. It does not instantly classify the whole resident run, because doing so would make shadow optimistic relative to live I/O scheduling.

Shadow mode does not reserve CPU capacity.

## 12. Exact accounting and telemetry

### 12.1 Per-key terminal partition

Every considered key is finalized once into exactly one terminal class:

$$
\begin{aligned}
considered ={}& primary\_redundant + secondary\_absent + gate\_reject \\
&+ bundle\_overflow + bundle\_trim + alloc\_refused \\
&+ submitted + shadow\_submit + lookup\_unresolved + cancelled
\end{aligned}
$$

`_finalize_keys()` increments `considered` and one terminal counter together. A bundle cursor records which resolved keys have already been dispositioned, making multi-step submission idempotent.

Keys after the first secondary absence are outside the reachable prefix and are not repeatedly probed or classified as independent candidates.

### 12.2 Bundle and owner metrics

The implementation also exposes:

- terminal bundle outcomes;
- state transitions;
- activation deferral by `demand_busy` or `speculation_busy`;
- owner-release reasons;
- active-owner gauge;
- predicted and actual lead-time histograms;
- resident bundle-size histogram;
- deadline-margin histogram; and
- fitted transfer base/per-chunk gauges.

### 12.3 CPU safety metrics

The CPU manager reports:

- physical fill percentage;
- free blocks;
- evictable blocks;
- write-pending blocks;
- speculative blocks;
- configured speculative reserve;
- free speculative reserve;
- leased speculative blocks;
- reserve blocks borrowed by demand; and
- leased blocks reclaimed by demand.

### 12.4 Legacy effect metrics

The existing tiering prefetch metrics continue to track attempted, promoted, useful, late, wasted, evicted-before-demand, load-failed, skipped, and untracked outcomes. These answer whether submitted work helped; the admission partition answers why keys were or were not submitted.

## 13. Configuration reference

The v6 AgentX treatment used:

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

| Field | Meaning |
|---|---|
| `enabled` | Instantiate the policy. |
| `policy` | Policy factory name; defaults to `admission`. |
| `policy_module_path` | Optional out-of-tree policy module. |
| `shadow_mode` | Make and account decisions without moving data. Defaults true. |
| `jit_activation` | Record at admission but delay lookup/promotion until exclusive JIT activation. |
| `demand_idle_only` | Defer activation/submission while demand lookup/load work exists. |
| `tier_idx` | Secondary tier used for residency and promotion. |
| `max_pending_bundles` | Bound on live policy bundles. |
| `max_promotions_per_step` | Global live/shadow submission budget per scheduler step. |
| `max_bundle_chunks` | Per-request candidate/probe/promotion ceiling. |
| `max_candidate_chunks` | Admission frontier-scan window. |
| `speculative_reserve_blocks` | Physical CPU blocks reserved for speculative allocation. `None` auto-derives; `0` disables speculative allocation. |
| `retention_lease_bundles` | Ready bundles protected until demand. `None` defaults to one; `0` disables retention. |
| `initial_admission_interval_ms` | Seed for queue service-time EWMA. |
| `admission_interval_ewma_alpha` | EWMA adaptation rate. |
| `transfer_base_ms` | Seed fixed transfer cost. |
| `transfer_per_chunk_ms` | Seed marginal transfer cost. |

Unknown keys and invalid types are rejected. `chunk_bytes` is derived internally and cannot be supplied by the user.

## 14. Source-code map

| Path | Responsibility |
|---|---|
| `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py` | Per-request opt-in, eligibility checks, ordered key construction, admission hook. |
| `vllm/v1/kv_offload/tiering/prefetch/config.py` | Strict policy configuration and defaults. |
| `vllm/v1/kv_offload/tiering/prefetch/base.py` | Policy/host interfaces and admission metrics. |
| `vllm/v1/kv_offload/tiering/prefetch/estimators.py` | Queue lead-time EWMA and measured transfer-cost regression. |
| `vllm/v1/kv_offload/tiering/prefetch/admission.py` | Bundle state machine, JIT owner, residency resolution, deadline gate, slicing, cancellation, exact accounting. |
| `vllm/v1/kv_offload/tiering/manager.py` | Policy lifecycle hooks, promotion batching/completion, owner release, speculative mark and lease wiring. |
| `vllm/v1/kv_offload/cpu/manager.py` | Allocation modes, physical reserve, speculative provenance, owner enforcement, reclamation, retention lease, CPU metrics. |
| `vllm/v1/kv_offload/tiering/async_lookup.py` | Demand/prefetch lookup priorities, pending-state upgrade, background batch execution. |
| `vllm/v1/kv_offload/tiering/fs/thread_pool.py` | Demand-load, prefetch-load, and store queues. |
| `vllm/v1/kv_offload/tiering/fs/manager.py` | Low-priority prefetch lookup, demand-idle signal, and prefetch load classification. |
| `vllm/v1/kv_offload/tiering/spec.py` | Configuration parsing, mutual exclusion, and metric registration. |

## 15. Tests and implementation provenance

Commit `c379bfdd5d` changed 15 files with 1,099 insertions and 114 deletions. Focused coverage includes:

- CPU reserve restoration and borrowing;
- each allocation mode;
- speculative provenance and demand conversion;
- owner conflicts and release;
- lease retention and demand breakage;
- earliest-deadline selection;
- JIT activation deferral;
- multi-step submission accounting;
- async lookup demand upgrade;
- filesystem demand/prefetch priority;
- manager completion and late-owner races; and
- configuration wiring.

The implementation was verified before the v6 build with 294 tests passed, 7 skipped, plus Ruff and diff checks.

## 16. Guarantees, best-effort properties, and non-guarantees

### Enforced by the current implementation

- Per-request Boolean opt-in.
- One stable full-attention KV group only.
- One bounded contiguous candidate bundle per request.
- Exact primary-residency frontier.
- Asynchronous exact secondary lookup up to the first absence.
- Fixed admission deadline and repeated deadline gating.
- One JIT speculative owner.
- Earliest-deadline activation.
- Physical speculative reserve with 25% pool ceiling.
- Speculative-only allocation cannot evict demand-owned data.
- Reactive demand may override reserve/lease to ensure progress.
- Demand lookup/load is queued ahead of prefetch.
- Owner-bound lease and late-completion suppression.
- Exact per-key terminal partition.

### Best effort, not hard isolation

- An already executing speculative filesystem call cannot be preempted.
- Lead-time and transfer models are estimates.
- The lease can be broken by demand-critical pressure.
- A request may be late even after a positive gate if workload or device conditions change.
- The reserve may be clamped below the requested size.

### Not implemented or not proven

- No learned model or offline training.
- No semantic scoring of tool type, agent role, or content.
- No true out-of-band tool-start/handoff event API.
- No multi-replica placement or llm-d routing decision.
- No promotion directly into GPU KV cache.
- No general support for multiple KV groups, sliding-window attention, or Eagle groups.
- No causal end-to-end performance proof yet.
- No production recommendation for `max_num_seqs=1`, reserve 512, or the current budgets.

## 17. Current empirical status

The first v6 AgentX treatment is a valid mechanism result:

- 1,024 chunks promoted;
- 512 useful;
- 448 wasted;
- 64 pending at the measurement boundary;
- 50.0% useful/promoted versus 4.71% in the v5 failure;
- no reserve borrowing;
- no lease reclamation; and
- no promotion load failures.

This supports the owner/JIT/lease correction. It does not establish a performance effect: treatment was already faster during matched warmup before speculative promotion occurred, the cells used different nodes, and both profiling phases timed out with cancellations. See [[2026-08-21 - V3 JIT demand-safe AgentX comparison|the v6 comparison report]].

## 18. Concise operational model

The current mechanism can be remembered as this sequence:

```text
request admitted
  -> require abc_admission_prefetch=true
  -> construct ordered KV keys
  -> skip CPU-resident/loading prefix
  -> retain bounded candidate bundle and fixed deadline
  -> wait for no owner and demand-idle tier
  -> choose earliest deadline
  -> claim one request as owner
  -> async low-priority filesystem residency lookup
  -> stop contiguous run at first absence
  -> require remaining horizon > measured remaining transfer cost
  -> allocate only from bounded speculative capacity
  -> batch filesystem->CPU promotion
  -> mark ready blocks speculative and lease one bundle
  -> demand converts useful blocks to ordinary cache data
  -> expiry/failure/finish releases lease and leaves unused blocks reclaimable
```

The design is deliberately conservative: speculate late, for one request, over an exact contiguous resident prefix, inside bounded capacity, and only while demand is idle.

## Related records

- [[00 - Index|Version3 index]]
- [[2026-08-21 - V3 JIT demand-safe AgentX comparison|First v6 AgentX comparison]]
- [[../Version2/04 - Theoretical Validation|Version2 theoretical validation]]
- [[../Version2/06 - 2026-08-19 - V2.1 Implementation Deep Dive|Original V2.1 implementation deep dive]]
- [[../Version2/Reports/2026-08-20 - V2.1 retention-lease Weka failure|v5 retention-lease failure that motivated JIT ownership]]