---
title: "RFC (issue-ready): Eager KV prefetching in vLLM"
date: "2026-09-22"
type: "research-rfc"
experiment: "ABC — Activity-Based KV Cache Tier Placement"
status: "draft-for-review"
decision: "Proposed design and validation gates for an upstream-ready RFC; no implementation authorized by this document"
authors:
  - "Alberto Perdomo — project owner"
  - "AI-assisted synthesis and code investigation"
code_reference:
  repository: "vllm-project/vllm"
  branch: "main"
  source_rfc_commit: "1ea7c63f4af7bb4fd6f025c8db44434ab274cb51"
  verification: "All code facts below re-verified against vLLM main on 2026-09-22 via GitHub MCP; line numbers follow the source RFC's pinned commit"
evidence:
  kind: "Synthesis of source RFC 09, verified vLLM main code, and prior experiment reports"
  new_benchmarks: false
related:
  - "[[Research/ABC/Methodology/09 - RFC - Eager KV prefetching in vLLM]]"
  - "[[Research/ABC/Reports/2026-09-07 - Lookahead demand staging v1 to v3]]"
  - "[[Research/ABC/Reports/2026-08-23 - Working-set oracle AgentX first comparison]]"
  - "[[Engineering/Learnings/vLLM KV offload retrieval path - lookup, promotion, and load]]"
---

# RFC (issue-ready): Eager KV prefetching in vLLM

## 0. Purpose of this document

This is the issue-ready companion to [[Research/ABC/Methodology/09 - RFC - Eager KV prefetching in vLLM]] (the research RFC). It converts that synthesis into a technically rigorous design that can be posted as a vLLM GitHub issue or upstream RFC. It adds what the research RFC deliberately left out: a concrete mechanism design (bounded CPU→GPU staging), precise integration points verified against current `main`, a state machine, failure/race handling, and an implementation plan.

It is still a design document. No prototype, benchmark launch, deployment, or upstream submission is performed by creating it. No numerical speedup is promised.

## 1. Summary and requested decision

The project aims to make reusable KV cache available before it becomes an exposed dependency of inference, lowering TTFT or improving useful throughput **without degrading ongoing generation or correctness**. A small repeatable benefit is sufficient; a known regression is not acceptable.

Eager prefetching includes:

- **Non-speculative preparation:** moving exact KV for an already queued request, or loading the next layer whose execution is guaranteed.
- **Speculative preparation:** moving existing KV for a likely future continuation or workflow branch before its inference request arrives.

The project moves existing, identity-verified KV. It does not predict unknown token values or synthesize approximate KV.

**Requested decision:** review the design below. Prioritize **bounded CPU→GPU staging for known waiting requests** (mechanism B) as the first engine-only prototype, with **early workflow/session advisories** (mechanism A) as the broader direction. Evaluate layer-wise loading (C) and tier pipelining (D) only when stage measurements establish their opportunity. Keep all new behavior opt-in until correctness and non-regression gates pass.

## 2. Problem statement and motivation

### 2.1 The exposed stall

In the native reactive path, a request that hits a secondary tier (filesystem/NVMe) must complete a full storage read → CPU promotion → CPU→GPU copy before its first token. Measured on the AgentX Weka workload (Nemotron 253B FP8, TP=8, concurrency 64, TieringOffloadingSpec, fs on node-local NVMe):

- Reactive baseline external lookup: P50 ≈ 2.2 s, P90 ≈ 5 s, P99 reached the 10 s histogram ceiling (2026-08-23 brief).
- The working-set oracle treatment promoted 676,388 chunks of which 673,320 (99.5%) were eventually useful — yet only **20 of 2,638 request intents (0.76%) were fully ready at first connector lookup**; 2,618 still deferred ([[Research/ABC/Reports/2026-08-23 - Working-set oracle AgentX first comparison]]).

**Figure 1** contrasts these two populations. Provenance: exact cumulative counts in the August 23 report; percentages computed from the shown numerators/denominators. These bars are not a common partition.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 1 — Eventually useful chunks versus requests ready at first lookup",
  "width": 640,
  "height": 130,
  "data": {
    "values": [
      {
        "population": "Promoted chunks eventually useful",
        "numerator": 673320,
        "denominator": 676388,
        "percent": 99.54641418830612
      },
      {
        "population": "Request intents fully ready at first lookup",
        "numerator": 20,
        "denominator": 2638,
        "percent": 0.7581501137225171
      }
    ]
  },
  "mark": {
    "type": "bar"
  },
  "encoding": {
    "y": {
      "field": "population",
      "type": "nominal",
      "sort": null,
      "title": "Separate populations"
    },
    "x": {
      "field": "percent",
      "type": "quantitative",
      "title": "Share of the stated population (%)",
      "scale": {
        "domain": [0, 100],
        "zero": true
      }
    },
    "color": {
      "field": "population",
      "type": "nominal",
      "scale": {
        "scheme": "category10"
      },
      "legend": null
    },
    "tooltip": [
      { "field": "population" },
      { "field": "numerator", "title": "Observed count" },
      { "field": "denominator", "title": "Population count" },
      { "field": "percent", "format": ".2f", "title": "Share (%)" }
    ]
  }
}
~~~

Figure 1 explains why an accurate selection can still miss its deadline: the mechanism moved the right data, but not early enough relative to demand. The working-set experiment was therefore **not a true perfect-residency oracle** — it rarely achieved full readiness — and its small outcome deltas must not be read as proof that eager prefetching is ineffective.

### 2.2 What prior experiments established

| Campaign | Verdict | Source |
|---|---|---|
| Lookahead demand staging v1–v3 | Mechanism accepted as neutral; no end-to-end benefit on this workload. Retrieval parallelism identified as the larger lever, built but not yet measured end to end | [[Research/ABC/Reports/2026-09-07 - Lookahead demand staging v1 to v3]] |
| Working-set oracle (admission-time, one-owner) | 0.76% full readiness; mean TTFT +1.87%, p95/p99 TTFT −1.11%/−6.51%, throughput +0.16%; failed the 5%/3% research gate. Rejects that implementation as a readiness oracle, not the concept | [[Research/ABC/Reports/2026-08-23 - Working-set oracle AgentX first comparison]] |
| Blind first-N admission (AgentX C32) | 90.99% redundant, 87.08% load-failed, 98.50% late | [[Research/ABC/2026-08-23 - ABC prefetch research brief for feedback]] |
| FS job split (v0.29.0, handoff 2026-09-22) | Standalone 512×2 MiB read: 372 ms one task vs 160 ms split (2.33×). Serving comparison: prefill p50/p90/p99 −2.5/−2.7/−2.3%, decode +20.1/+6.7/+3.3%, preemption +181%, output-token rate −6.8%, request throughput −2.8% — a regression signal; GIL contention is a plausible explanation, not an isolated observation | KV-OFFLOAD-JOB-SPLIT-HANDOFF.md (local, 2026-09-22) |

**Evidence validity verdict (from source RFC 09):** conditionally valid synthesis; no new performance proof. The September 22 handoff adds a reported serving regression to the September 7 KB, which still described that experiment as unmeasured. This document preserves that chronology.

### 2.3 Metric semantics that constrain interpretation

Verified in `vllm/v1/metrics/stats.py` (main):

- `FinishedRequestStats`: `queued_time` = first QUEUED event → first SCHEDULED; `prefill_time` = first SCHEDULED → first NEW_TOKEN; `decode_time` = first NEW_TOKEN → last NEW_TOKEN; `inference_time` = first SCHEDULED → last NEW_TOKEN.
- Native asynchronous KV loading happens **before** the SCHEDULED event. Therefore a smaller prefill interval is not direct proof that faster filesystem reads caused a request-level gain.
- `PrefillStats` and `PromptTokenStats` already separate `num_local_cached_tokens` from `num_external_cached_tokens` (`ALL_SOURCES = ("local_compute", "local_cache_hit", "external_kv_transfer")`), so external-token accounting exists and can be extended for staging.

## 3. Current behavior (verified against main, 2026-09-22)

The native retrieval sequence:

1. Request registration creates connector state; it does not itself retrieve KV.
2. The scheduler examines a waiting request and checks the local GPU prefix (`_get_local_prefix_cache_hit`).
3. The offloading connector scans exact chunk keys. CPU hits are immediately known; secondary existence checks run asynchronously.
4. A secondary hit reserves a CPU destination; end-of-step batching submits secondary→CPU promotion.
5. On completed promotion, the CPU chunks become readable.
6. GPU allocation and CPU pinning create a worker load job; the request waits for asynchronous reception (`WAITING_FOR_REMOTE_KVS`).
7. After reception completes, the request can enter computation and valid GPU cache content can be published.

### 3.1 Verified code facts and their consequences for eager loading

| # | Code fact (verified on main) | Consequence for eager loading |
|---|---|---|
| F1 | Offloading scheduler `_chunks_being_loaded` dedup set, gated on `enable_prefix_caching` | Staging must register chunks in the same dedup set to prevent duplicate loads |
| F2 | `_lookup_complete_chunks` convergence loop; prefix scan continues across RETRY and stops on a true MISS | Ordinary demand lookup already scans ahead over known keys; post-miss same-request read-ahead is not the missing mechanism |
| F3 | `supports_partial_tail` (Mamba align mode); EAGLE volatile trailing chunk handling | Partial-tail and speculative-group semantics must be verified before staging hybrid models |
| F4 | Per-request `max_load_tokens` / `max_offload_tokens` caps; `OffloadPolicy.CHUNK_LEVEL` / `REQUEST_LEVEL` | Staging must respect the same per-request byte caps and policy granularity |
| F5 | Tiering manager `bp_detector`, `_should_store_to_tier`, `_update_backpressure` | Optional secondary-store backpressure exists on main; staging must respect it |
| F6 | `_flush_pending_cascades` (request-level tiers); `_maybe_finalize_request` delayed finalization | Request finalization must wait for in-flight staging; cascade semantics differ by tier level |
| F7 | `lookup()` returns `MISS` when promotion cannot be initiated; `_initiate_promotion` allocates the primary CPU slot immediately with `ref_cnt=-1` | CPU slot reservation pattern exists; GPU staging needs the analogous reservation discipline |
| F8 | FS manager enqueues one task per job (`self._pool.enqueue_load(job_id, 1, [load_task])`); `blocks_per_task` is NOT in main | More pool threads parallelize different jobs, not blocks within a job; the job-split knob is local/uncommitted |
| F9 | C batch loop (`csrc/fs_io.cpp` `batch_load_block`) releases the GIL for the whole batch, loops serially, and attaches `num_succeeded` on partial failure | The C layer already supports multi-block batches and preserves partial success; the Python layer under-uses it |
| F10 | `SingleDirectionOffloadingHandler` (gpu_worker.py) strictly serializes transfers within a direction: each transfer gets a unique CUDA stream that waits on the previous transfer's end event | Staging transfers compete with demand transfers in FIFO order within a direction; batching or priority is required for staging to win lead time |
| F11 | `is_src_access_order_any = not gpu_to_cpu` in `transfer_async` | CPU→GPU reads from host memory can use `CU_MEMCPY_SRC_ACCESS_ORDER_ANY` (driver pipelines source reads); GPU→CPU must keep stream ordering |
| F12 | `wait_for_layer_load` / `save_kv_layer` are abstract no-ops; `requires_piecewise_for_cudagraph` exists | Layer-wise prefetch needs real per-layer completion semantics; graph mode requires piecewise CUDA graphs |
| F13 | `KvHintsEnvelope` / `KvHintAction` exist; KVCR manager `on_new_request` submits hints via `self._kvcr.submit_hint(...)` | Typed hint transport exists; native filesystem/HBM prefetch execution is not supplied by the envelope itself |
| F14 | Scheduler blocked statuses (`WAITING_FOR_STRUCTURED_OUTPUT_GRAMMAR`, `WAITING_FOR_REMOTE_KVS`, `WAITING_FOR_STREAMING_REQ`) route to `skipped_waiting`; the admission loop `continue`s on deferred lookup and `break`s on allocation failure | Staging must not block the admission loop; deferred requests already do not cause head-of-line blocking |
| F15 | `WAITING_FOR_REMOTE_KVS` promotion happens only when the req_id appears in `finished_recving_kv_req_ids`, populated from worker-side `KVConnectorOutput.finished_recving` in `update_from_output` | The async-load completion path exists and can be reused for staging completion |
| F16 | Async-load admission sets `num_computed_tokens` optimistically, adds the request to `_inflight_prefills`, and skips zeroing of destination blocks via `_skip_zero_block_ids` | Staging destinations must also skip zeroing; the reservation must be visible to the allocator |
| F17 | CPU memory is shared between scheduler and workers through an mmap region | Moving planning/I/O coordination off the scheduler's Python path does not eliminate memory, PCIe, or GPU contention |
| F18 | The native filesystem tier has no ordinary capacity-based eviction; KVCR and backend-specific policies exist | "No eviction" must not be generalized to every secondary backend |

### 3.2 Source map (pinned to the source RFC commit)

- Prefix lookup and request registration: `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L671`
- Waiting-request admission: `vllm/v1/core/sched/scheduler.py#L859`
- Tiering promotion and completion: `vllm/v1/kv_offload/tiering/manager.py`
- Filesystem job submission: `vllm/v1/kv_offload/tiering/fs/manager.py#L223`; C batch loop: `csrc/fs_io.cpp#L272`
- CPU→GPU worker: `vllm/v1/kv_offload/cpu/gpu_worker.py`
- Layer hooks and graph-mode contract: `vllm/distributed/kv_transfer/kv_connector/v1/base.py#L647`
- KV hint envelope: `vllm/v1/kv_hints/protocol.py`; KVCR hint forwarding: `vllm/v1/kv_offload/tiering/kvcr/manager.py#L577`

## 4. Goals and non-goals

### Goals

1. Hide exposed storage, network, and CPU→GPU loading latency using the earliest reliable signal.
2. Preserve useful KV until consumption without causing a larger miss or preemption elsewhere.
3. Make readiness explicit at the CPU, GPU, and, where supported, layer levels.
4. Bound scheduler work, outstanding bytes, destination reservations, retention time, and interference.
5. Retain the native reactive load/recompute path when preparation is late, unavailable, rejected, cancelled, or unsuccessful.
6. Produce a reproducible decision about where prefetch helps, including regimes where it should remain disabled.

### Non-goals (initial scope)

- A learned per-block temperature predictor.
- Model-weight prefetching or sparse/approximate attention.
- Automatic CPU-pool growth or a new GPU allocator.
- A blanket cache replacement-policy rewrite.
- Making faster filesystem I/O the project's sole goal.
- Opening an upstream issue or PR before the relevant contribution and human-review requirements are satisfied.

### Scope

Implementation target is the native vLLM offloading and scheduler path. CPU DRAM is the primary offload tier; filesystem/NVMe/CephFS and remote tiers supply colder copies. llm-d or an application runtime may provide an early signal and select a destination, but vLLM owns allocator state and the authority to admit a transfer.

Initial correctness scope: one explicitly supported full-attention configuration. Hybrid attention, Mamba, sliding windows, partial tails, speculative decoding, TP/PP, and cross-topology formats require their own compatibility gates before support is advertised.

## 5. Constraints and invariants

Required invariants (from source RFC 09, retained verbatim as design constraints):

- Exact hash/model/configuration identity and group-specific prefix rules remain authoritative.
- The original target never silently shrinks to make a completion metric look successful.
- Speculative preparation cannot evict active/pinned state.
- Capacity reserved for prefetch is visible to the normal allocator.
- Initial speculative policy uses a bounded explicit allowance; "evictable" is not synonymous with "free of opportunity cost."
- No global cache publication before the relevant data is valid.
- Failures preserve the largest safe usable result and use reactive fallback.
- Source and destination lifetime tracking survives aborts, preemption, reset, shutdown, and multi-rank completion.
- Planning cannot synchronously scan unbounded keys in the scheduler.
- An idle engine must still receive control work and finish transfers; avoid indefinite busy polling as the default control mechanism.

Admission heuristic (decision model, not a proven guarantee):

$$
\text{expected exposed stall saved} > \text{expected interference} + \text{eviction regret} + \text{planning cost}.
$$

Estimate it in time units with measured error bounds. Use byte- and duration-based transfer budgets; a fixed chunk count has different costs across models.

## 6. Proposed design: bounded CPU→GPU staging for known waiting requests (mechanism B)

### 6.1 Concept

Separate **transfer admission** from **compute admission**. When a waiting request has exact CPU-ready KV and compute admission is closed (token budget or `max_num_active_reqs` exhausted) but sufficient HBM remains, initiate the CPU→GPU copy immediately — without adding the request to RUNNING. The request becomes GPU-ready before compute admission opens, so the exposed copy latency disappears from the critical path.

This is the minimal engine-only change: it reuses the existing bulk CPU→GPU path (F10/F11), the existing async-load completion path (F15), and the existing reservation discipline (F7/F16). It does not require new hint transport (F13) or layer semantics (F12).

### 6.2 Trigger and decision

A staging candidate is a waiting request that satisfies **all** of:

1. **Exact CPU-ready KV exists** for a non-empty prefix (CPU residency verified; no secondary read needed).
2. **Compute admission is closed** for this request (token budget exhausted, or `num_running >= max_num_active_reqs`), so the copy would otherwise sit on the critical path.
3. **HBM headroom exists**: reserving the request's GPU blocks keeps `kv_cache_usage` below a staging-aware watermark.
4. **Staging budget is available**: outstanding staged bytes and reservation age are within the bounded allowance.
5. **No duplicate in flight**: the chunks are not already in `_chunks_being_loaded` (F1) and the request is not already staging.

The decision runs in the scheduler step, bounded to the already-resolved chunk set (no unbounded key scans — invariant 9). The staging allowance is a config knob (bytes + duration), default off.

### 6.3 State machine

~~~text
WAITING ──(staging decision: CPU-ready, compute closed, HBM headroom)──▶ STAGING
STAGING ──(GPU copy complete)──▶ WAITING (GPU-ready) ──(compute admission)──▶ RUNNING
STAGING ──(transfer failed)──▶ WAITING (reactive fallback: demand load or recompute)
STAGING ──(expired / cancelled / rerouted)──▶ WAITING (reservations released)
~~~

`STAGING` is a new blocked waiting status added to `_is_blocked_waiting_status` (F14), so the admission loop `continue`s past staging requests without blocking. Promotion out of `STAGING` mirrors `_try_promote_blocked_waiting_request` for `WAITING_FOR_REMOTE_KVS` (F15): the worker-side connector reports completion in `KVConnectorOutput.finished_recving`, the scheduler records the req_id in `finished_recving_kv_req_ids`, and the request returns to `WAITING` with `num_computed_tokens` reflecting only successfully loaded tokens.

### 6.4 Integration points

| Component | Change | Reuses |
|---|---|---|
| `vllm/v1/core/sched/scheduler.py` | New `STAGING` status in `_is_blocked_waiting_status`; staging decision in the waiting loop; promotion in `_try_promote_blocked_waiting_request`; add staged requests to `_inflight_prefills`; extend `_skip_zero_block_ids` to staging destinations | F14, F15, F16 |
| `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py` | New connector API, e.g. `stage_kv_for_request(request_id, chunk_ids)`; register chunks in `_chunks_being_loaded`; respect `max_load_tokens`/`max_offload_tokens`; report staging completion through the existing `finished_recving` channel | F1, F4, F15 |
| `vllm/v1/kv_offload/tiering/manager.py` | GPU-side reservation analogous to `_initiate_promotion`'s CPU slot (`ref_cnt=-1`); staging must not trigger cascades or premature finalization; respect `bp_detector` gates | F5, F6, F7 |
| `vllm/v1/kv_offload/tiering/fs/manager.py` | No change required for B (CPU-ready staging skips the secondary tier). Optional: `blocks_per_task` batching for the secondary→CPU leg, gated behind the job-split experiment | F8, F9 |
| `vllm/v1/kv_offload/cpu/gpu_worker.py` | Staging transfers flow through the existing `submit_load`; the FIFO serialization within a direction (F10) means staging must be **batched into the same transfer as demand loads** or given a bounded priority window, otherwise it cannot win lead time | F10, F11 |
| `vllm/v1/metrics/stats.py` | New staging counters (initiated, completed, consumed, expired, cancelled, failed, late; staged bytes; reservation age); extend `PrefillStats`/`PromptTokenStats` with a `staged` source label | existing external-token accounting |

### 6.5 Request/data flow

1. Scheduler step: waiting request fails compute admission (budget/active-req cap) but passes the staging decision (6.2).
2. Connector resolves the exact CPU-ready chunk set (already known from the demand lookup) and registers it in `_chunks_being_loaded`.
3. Tiering manager reserves GPU destinations (counted against normal capacity) and pins the CPU source chunks.
4. Scheduler sets status `STAGING`, adds the request to `_inflight_prefills`, and extends `_skip_zero_block_ids` to the destinations.
5. End of step: the worker-side connector submits the CPU→GPU copy through `submit_load` (batched with demand loads per 6.4).
6. On copy completion, the worker reports `finished_recving`; the scheduler promotes the request to `WAITING` (GPU-ready).
7. When compute admission opens, the request enters RUNNING with zero exposed copy latency; valid GPU content is published per the normal rules.

## 7. Alternatives and tradeoffs

| Mechanism | Trigger | Why it might work | What could defeat it | Verdict |
|---|---|---|---|---|
| **B. Bounded GPU staging (proposed first)** | CPU-ready KV + closed compute admission + HBM headroom | Moves beyond CPU-only readiness; reuses bulk copy path and graph mode | Active requests already consume HBM; forecast reservations prolong pressure; transfer slows generation | **Primary prototype** |
| **A. Early workflow/session preparation** | Predecessor starts, tool wait begins, known continuation scheduled | Preparation horizon starts before inference admission | Late/incorrect hints, unstable prefix identity, unavailable source data, wasted retention, destination load imbalance | Broader direction; requires llm-d coordination and reroute reconciliation |
| **C. Layer-wise deterministic prefetch** | Execution order guarantees the next layer | Overlap even with little request-level lead time | Short new suffixes, PCIe/memory contention, fragmented transfers, piecewise-graph overhead; native partial-tail/group semantics unverified | Conditional on stage measurements |
| **D. Pipelined secondary→CPU→GPU promotion** | Initial group of exact chunks completes its storage read | Overlaps two transfer stages; removes whole-job barrier; valuable on slower tiers (CephFS) | Bookkeeping, small-transfer inefficiency, scarce destinations, partial-failure handling | Conditional; depends on job-split outcome |

Faster filesystem service is an enabling optimization for A–D. It supplies neither an earlier trigger nor a readiness contract by itself.

## 8. Failure handling, cancellation, and races

- **Transfer failure:** the request returns to `WAITING` and uses the native reactive fallback (demand load or recompute). `batch_load_block` already attaches `num_succeeded` (F9) so partial results are preserved; `_update_waiting_for_remote_kv` caches only successfully loaded tokens.
- **Cancellation:** stops unsubmitted work. In-flight destinations remain protected until I/O completion makes release safe; cancellation is not permission to reuse memory immediately.
- **Expiry:** unused staging releases its reservations and retention protection; the request proceeds reactively.
- **Reroute (llm-d):** a destination change must invalidate or reconcile the old plan; without this, llm-d may consume bandwidth on a replica that never serves the continuation.
- **Races with zeroing:** staging destinations must be added to `_skip_zero_block_ids` (F16) exactly like async-load destinations; the scheduler's skip set only edits the step being scheduled and cannot retract zeroing shipped earlier for a since-reallocated block.
- **Races with eviction:** staged/pinned state is not evictable; the allocator sees reservations as normal capacity (invariant 4).
- **Shared requests:** requests referencing common work share it rather than cancelling each other's data.
- **Multi-rank completion:** source and destination lifetime tracking must survive aborts, preemption, reset, and shutdown (invariant 8).

## 9. Memory, admission, and backpressure

- Staging reservations count against normal GPU capacity; `kv_cache_usage` includes them.
- A staging-aware watermark gates new staging when HBM headroom disappears.
- Outstanding staged bytes and reservation age are bounded by the configurable allowance; expired work drains.
- The `bp_detector` gates (F5) apply to staging as to any store; staging must not trigger request-level cascades or premature finalization (F6).
- The admission loop's `continue`-on-deferred / `break`-on-allocation-failure semantics (F14) are preserved: staging requests never cause head-of-line blocking.

## 10. Observability

| Question | Measurements |
|---|---|
| Did the signal arrive early enough? | Intent timestamp, expected-use window, actual first demand, immutable target bytes |
| Where was time spent? | Lookup enqueue/start/end, storage submission/start/completion, CPU-ready, GPU allocation, GPU-copy start/end, first compute, first token |
| Did preparation remove exposed delay? | Per-request readiness at demand/admission, valid prefix coverage, per-layer waits where applicable |
| What did it cost? | Scheduler-step duration, Python/GIL or host profiles, transfer queue/service time, CPU memory bandwidth, PCIe traffic, GPU SM/HBM activity |
| Did capacity suffer? | Free/evictable/pinned/staging bytes, reservation age, running/waiting requests, preemption and recomputation |
| Was it useful overall? | TTFT, ITL, E2E/workflow completion, request/output-token throughput, completed sessions, failures and cancellations |
| Was staging wasteful? | Consumed, late, expired, failed, duplicate, restaged bytes and complete eviction outcomes |

New staging counters: initiated, completed, consumed, expired, cancelled, failed, late; staged bytes; reservation age. Collect native temporal samples; do not downsample merely to fit a chart. Separate request quantiles from averages of rolling histogram quantiles. The filesystem tier's own read/write counters describe a different hop than CPU↔GPU transfer metrics; job timing must have documented semantics (summing concurrent task durations is not job elapsed time).

## 11. Security and compatibility

- **Security:** no new network surface in mechanism B (engine-local). Mechanism A introduces external advisories; they must be authenticated/validated like existing hint transport and must not allow arbitrary memory movement without engine authority.
- **Compatibility:** new behavior is opt-in behind a flag, default off. Feature-off/no-op behavior must be validated independently. Initial scope is one full-attention configuration; hybrid/Mamba/sliding-window/partial-tail/speculative/TP-PP/cross-topology formats require their own gates (F3).
- **Upstream coordination:** reuse or extend the existing typed envelope (F13) rather than duplicating another incompatible hint format. Related upstream RFCs: session hints #52113, programmatic management #51428, programmable policy #57103.

## 12. Rollout

1. **Phase 0 — Observability and A/A:** staging counters and no-op path; verify measurement stability and that the inactive mechanism is negligible.
2. **Phase 1 — Mechanism B prototype (engine-only, opt-in flag):** bounded GPU staging for exact queued keys; correctness and lifetime/failure tests.
3. **Phase 2 — Gates E1/E2:** readiness opportunity (CPU-ready vs GPU-ready controls at matched effective capacity) and queued GPU staging (reactive vs B).
4. **Phase 3 — Mechanism A:** advisory-driven preparation with llm-d, exact future prefixes, controlled lead times; then realistic signals.
5. **Phase 4 — Conditional C/D:** layer-wise loading and tier pipelining only if stage measurements establish the opportunity.

Default enablement is a separate decision. A mechanism useful only in a known regime remains opt-in.

## 13. Testing and benchmarks

- Correctness: existing relevant suites plus new behavior tests for lifetime/failure cases (expiry, cancellation, reroute, partial failure, multi-rank completion) and representative model evaluation.
- Feature-off/no-op validated independently.
- Benchmark protocol: same-node, counterbalanced comparisons; identical immutable images except when a graph-mode ablation requires otherwise; at least three paired repetitions per accepted arm; same seed, cache preparation, request mix, routing policy, drain policy, storage backend, and contention conditions; record actual image digests and rendered arguments.
- Regimes: low-queue and pressured (C32/C64 are historical starting points, not universal operating points), CPU-ready and secondary-resident hits, short and long new suffixes, and a no-external-hit control. First fix one model/topology; use a second model and multi-replica llm-d only after the mechanism is understood. Explicitly verify all participating KV groups for hybrid models.
- CephFS is a proposed discriminating regime, not a guaranteed winner. Low average NVMe utilization alone cannot prove storage latency never affects a request; inspect event-aligned queue/service time and tails.

## 14. Open questions

1. Does this workload expose storage time, CPU→GPU time, GPU allocation time, or mostly compute capacity?
2. Can application events identify a stable existing prefix early enough, and is the eventual destination stable?
3. Should the initial delivery be B alone, or an advisory-driven A/B combination?
4. What explicit staging allowance is affordable without displacing more valuable data?
5. Can staging transfers be batched with demand loads without disturbing the FIFO serialization (F10), or is a bounded priority window required?
6. Which attention groups and topology combinations can be supported without weakening validity/publication rules?
7. Is a separate control process worthwhile after measuring actual scheduler/GIL cost?
8. How should out-of-band hints be accepted while no inference request is active, and which upstream interface owns this?
9. When is valid partial-prefix use preferable to waiting for a longer external hit?
10. What experiment precision is needed to accept a small improvement under the no-regression objective?

## 15. Acceptance criteria and implementation plan

### Acceptance criteria

- Correctness passes the existing relevant suites plus behavior tests for new lifetime/failure cases and representative model evaluation.
- Feature-off/no-op behavior is validated independently.
- At least one supported regime shows a repeatable request/workflow benefit attributable to earlier useful readiness.
- No supported evaluated regime has an established throughput, ITL, preemption, or error regression.
- Statistical precision is sufficient to assess the claimed benefit and non-regression. "Not statistically significant" is not evidence of safety.
- Resource use remains bounded; expired work drains; no starvation or persistent speculative pressure appears.
- Retention-only, transfer-only, and prediction contributions are isolated where combined.

"No regression" is the objective, not a proof obtainable for every unseen workload. Before running, state the practical detection precision and confidence procedure. Previous 5%/3% or 10%/5% research gates were campaign-specific and are not silently adopted as this RFC's minimum upside; a smaller demonstrable gain remains acceptable.

Stop or redirect a mechanism when actual readiness fails to improve outcomes, overhead exceeds savings, sufficient lead time cannot be obtained, or capacity cannot be preserved. A documented negative result is a valid research deliverable.

### Implementation plan (mechanism B, minimal)

1. Add `STAGING` to `_is_blocked_waiting_status`; extend `_try_promote_blocked_waiting_request` with the staging promotion path (reusing `finished_recving_kv_req_ids`).
2. Add the staging decision to the waiting loop (bounded to the already-resolved chunk set; staging-aware watermark; byte/duration allowance).
3. Add `stage_kv_for_request` to the offloading connector; register chunks in `_chunks_being_loaded`; respect per-request caps.
4. Add GPU-side reservation in the tiering manager (analogous to `_initiate_promotion`); extend `_skip_zero_block_ids` and `_inflight_prefills`.
5. Batch staging transfers with demand loads in `submit_load` (or bounded priority window).
6. Add staging counters and the `staged` source label to metrics.
7. Tests: feature-off/no-op, expiry, cancellation, reroute, partial failure, multi-rank completion, zeroing race.
8. Run gates E0 (A/A), E1 (readiness opportunity), E2 (queued GPU staging) per the protocol in section 13.

## 16. References

- [[Research/ABC/Methodology/09 - RFC - Eager KV prefetching in vLLM]] — evidence synthesis, run registry (L0/L1/V0/V1/W0/W1/F0/F1/N0–N2/NX), and related-systems survey (KVFlow, Symphony, CachedAttention, Strata, PBKV).
- [[Research/ABC/Reports/2026-09-07 - Lookahead demand staging v1 to v3]]
- [[Research/ABC/Reports/2026-08-23 - Working-set oracle AgentX first comparison]]
- [[Research/ABC/2026-08-21 - Independent research audit and redirection for speculative KV prefetching]]
- [[Research/ABC/2026-08-23 - ABC prefetch research brief for feedback]]
- [[Engineering/Learnings/vLLM KV offload retrieval path - lookup, promotion, and load]]
- [[Engineering/Learnings/vLLM KV block prefetch architecture]]
- [[Engineering/Learnings/vLLM KV Events canonical form]]
- [[Engineering/Learnings/vLLM and llm-d-router KV cache responsibility split]]
- [[Engineering/Learnings/vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]]
- [[Engineering/Learnings/vLLM offloading spec architecture and dev-shm confound]]
- [[Research/ABC/00 - Index]], [[Research/ABC/Methodology/00 - Index]]