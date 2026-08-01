---
title: vLLM KV offload retrieval path - lookup, promotion, and load
date: 2026-08-01
type: learning
topic: KV cache offloading
repo: vllm-project/vllm
commit: 4ee9702bee668a447e9983a6aefc16ebbc3ad32e
commit_date: 2026-07-31
---

# vLLM KV offload retrieval path — lookup, promotion, and load

How an offloaded KV block gets back into GPU memory when a request needs it, across the CPU primary tier and the secondary tiers (filesystem/NVMe, object store, P2P).

All code links are pinned to `vllm-project/vllm@4ee9702` (`4ee9702bee668a447e9983a6aefc16ebbc3ad32e`, 2026-07-31, on `upstream/main`), so line anchors stay correct even as `main` moves. This covers the **native** offloading path only — see [[vLLM KV block prefetch architecture]] for how it compares to the LMCache connector and to model-weight prefetch.

Related: [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]] · [[vLLM offloading spec architecture and dev-shm confound]] · [[vLLM KV Events canonical form]] · [[CephFS performance tuning for KV cache offloading]] · [[vLLM]]

---

## 0. Headline: there is no speculative prefetch

`grep -i prefetch` over `vllm/v1/kv_offload/` returns nothing. Retrieval is **demand-driven at request admission**: when a request enters the scheduler, the connector looks up its block hashes across the tiers, and whatever hits is pulled back *before* the request is allowed to run. Nothing is fetched on a guess about future traffic.

Two mechanisms make it *behave* somewhat like prefetch, and they are the interesting engineering:

1. **Staged promotion.** Secondary tiers can never touch GPU memory. A hit on FS/NVMe/obj/P2P triggers a secondary→CPU promotion that runs across scheduler steps *while the request waits in the queue*. The request is not scheduled until its blocks land.
2. **Async batched existence checks.** Secondary-tier `stat()`-equivalents run on a background thread with one batch per scheduler step, overlapping the model-execution window, so the scheduler thread never blocks on storage.

The naming in the older KB note ("prefetch") is therefore slightly misleading; "retrieval" or "promotion" is the accurate framing.

---

## 1. Vocabulary and the layer model

### Key types

| Concept | Definition | Code |
|---|---|---|
| `OffloadKey` | Block hash + KV-group index, packed as raw bytes (avoids tuple GC overhead) | [`base.py:26-41`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/base.py#L26-L41) |
| **chunk** | The offloading unit — `blocks_per_chunk` consecutive GPU blocks. Offloaded blocks may be *larger* than GPU blocks | [`scheduler.py:229-258`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L229-L258) |
| `LookupResult` | `HIT` / `HIT_PENDING` / `RETRY` / `MISS` | [`base.py:107-113`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/base.py#L107-L113) |
| `OffloadingManager` | Scheduler-side: tracks what is offloaded and where | [`base.py:220`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/base.py#L220) |
| `OffloadingWorker` | Worker-side: performs the actual GPU↔CPU DMA | [`base.py:545`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/base.py#L545) |
| `SecondaryTierManager` | One non-primary tier; runs **in the scheduler process** | [`tiering/base.py:100`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/base.py#L100) |

The chunk/token relations that govern every index computation in the scheduler:

$$
\text{tokens\_per\_chunk} = \text{tokens\_per\_block} \times \text{blocks\_per\_chunk}
$$

$$
\text{hashes\_per\_chunk} = \frac{\text{tokens\_per\_chunk}}{\text{tokens\_per\_hash}}
$$

A chunk is offloadable only once all its GPU blocks are allocated *and* computed, which is why `storable_chunks()` takes the min of computed-token chunks and allocated-block chunks ([`scheduler.py:338-364`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L338-L364)).

### The layer model

```
                GPU KV cache  (worker process)
                     ▲
                     │  CPU→GPU DMA, dedicated CUDA stream
                     │  CPUOffloadingWorker.submit_load
                     │
        CPU primary tier  ──  mmap: /dev/shm/vllm_offload_{engine_id}.mmap
                     ▲          (mapped by BOTH scheduler and workers)
                     │
                     │  secondary→CPU, tier's own threads / NIXL
                     │  SecondaryTierManager.submit_load
                     │
   ┌─────────────────┴─────────────────┐
   │        │            │             │
  "fs"    "obj"        "p2p"       "example"
 FS/NVMe  object      remote peer   test tier
          store       (NIXL RDMA)
```

Registered secondary tier types are `example`, `fs`, `p2p`, `obj` ([`tiering/factory.py:55-77`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/factory.py#L55-L77)).

**The CPU tier is a mandatory gateway.** This is stated as an invariant on the interface itself ([`tiering/base.py:103-107`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/base.py#L103-L107)): store is `GPU → CPU → secondary` (cascade), load is `secondary → CPU → GPU` (promotion).

### Why the mmap matters

`SharedOffloadRegion` ([`cpu/shared_offload_region.py:28`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/shared_offload_region.py#L28)) is a single `/dev/shm` file mmapped by every process. Workers race to create it with `O_EXCL`; the winner `ftruncate`s and the rest wait for the expected size.

This is what makes the split work. Secondary-tier I/O happens on **scheduler-process** threads, but writes land in memory the **worker process** will DMA from — an FS read thread doing `os.readv` into `primary_kv_view[block_id]` is filling the exact buffer the CUDA copy engine reads later. **No KV bytes are ever IPC'd.** The scheduler-side view is created at [`tiering/manager.py:114`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L114) and handed to each tier at construction ([`tiering/spec.py:192-198`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/spec.py#L192-L198)); the worker-side tensors are carved out of the same region at [`tiering/spec.py:252-264`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/spec.py#L252-L264). The whole region is registered with `cudaHostRegister` for pinned DMA ([`cpu/gpu_worker.py:123-150`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py#L123-L150)); failure degrades to unpinned DMA with a warning rather than erroring out.

---

## 2. Path A — block is resident in the CPU tier

The fast path. Six stages, all within one or two engine steps.

### Stage 1 — Lookup at admission

The core scheduler asks the connector how many tokens it can supply beyond the local prefix-cache hit:

- [`v1/core/sched/scheduler.py:777-781`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/core/sched/scheduler.py#L777-L781) — call site. Note it passes a **block-aligned** local hit count, so a strictly longer external hit can supersede a local sub-block tail without racing copy-on-write ([`:791-811`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/core/sched/scheduler.py#L791-L811)).
- [`offloading/scheduler.py:816`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L816) — `get_num_new_matched_tokens`.
- [`offloading/scheduler.py:631`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L631) — `_lookup`, the real work.

`_lookup` is more subtle than a prefix scan because of hybrid models. It iterates **full-attention groups first**, then **sliding-window groups**, and re-runs the loop when a later group tightens `max_hit_size_tokens` enough to invalidate an earlier group's answer — a convergence loop, not a single pass.

| Group kind | Scan | Code |
|---|---|---|
| Full attention | Maximal prefix: count consecutive hits from the start | [`:545`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L545) |
| Sliding window / Mamba | Scan **backwards** for the last run of `sliding_window_size` consecutive hits | [`:579`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L579) |

Per-chunk, `manager.lookup()` on the CPU tier ([`cpu/manager.py:113`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/manager.py#L113)) returns:

- `HIT` — present and `is_ready`.
- `HIT_PENDING` — the slot is allocated but its **GPU→CPU store is still in flight**. (Not "promotion in progress" — that distinction matters when debugging.)
- `MISS` — not in the pool.

### Stage 2 — Park the request

`get_num_new_matched_tokens` returns `(num_hit_tokens, True)`; the `True` means "loaded asynchronously between scheduler steps". The scheduler allocates GPU blocks and moves the request to `WAITING_FOR_REMOTE_KVS` ([`v1/core/sched/scheduler.py:1023-1027`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/core/sched/scheduler.py#L1023-L1027)). It does **not** run this step.

### Stage 3 — Pin and build the transfer job

`update_state_after_alloc` ([`offloading/scheduler.py:873`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L873)) computes which chunks are still absent from GPU, then:

- `manager.prepare_load(keys)` → [`cpu/manager.py:130`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/manager.py#L130) increments `ref_cnt`, marks the blocks non-evictable, and returns a `CPULoadStoreSpec` of CPU block IDs.
- A `GPULoadStoreSpec` carries destination GPU block IDs plus `group_sizes` and `block_indices` ([`base.py:409-441`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/base.py#L409-L441)). `block_indices` exists because an offloaded chunk can span several GPU blocks: the first matching chunk may be misaligned, and the worker must skip part of it.

The pair becomes a `TransferJob` in the connector metadata, tracked with `pending_count = num_workers` ([`:962-967`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L962-L967)) — every TP rank must ack.

### Stage 4 — Worker executes the DMA

At the start of the forward pass: `start_load_kv` ([`offloading_connector.py:103`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading_connector.py#L103)) → `start_kv_transfers` ([`offloading/worker.py:319`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/worker.py#L319)) → `CPUOffloadingWorker.submit_load` ([`cpu/gpu_worker.py:537`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py#L537)) → `transfer_async` ([`cpu/gpu_worker.py:240`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py#L240)).

`transfer_async` builds three pinned descriptor arrays (src pointers, dst pointers, sizes) with a vectorized numpy expansion over sub-blocks ([`compute_sub_block_ptrs`, `:73`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py#L73)), then issues **one batched `swap_blocks_batch`** on a pooled CUDA stream. Streams, events, and descriptor buffers are all pooled for reuse.

Two details worth remembering:

- **Ordering.** Each transfer's stream waits on the previous transfer's end event ([`:379-383`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py#L379-L383)), so submission order is preserved within a direction.
- **Access-order semantics differ by direction** ([`:384-390`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py#L384-L390)). CPU→GPU uses `CU_MEMCPY_SRC_ACCESS_ORDER_ANY` (host pinned source is never concurrently written, so the driver may pipeline source reads). GPU→CPU must keep STREAM ordering because it reads the live GPU KV cache that the compute stream is still writing.

Kernel selection is resolved once at init ([`_select_swap_blocks_fn`, `:35`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py#L35)): GPU→CPU always uses the C++ DMA path (bandwidth-bound, copy engine wins); CPU→GPU uses a Triton kernel **only** for small (`< THRESHOLD_BYTES`), 8-byte-aligned pages, and falls back to DMA on ROCm/XPU or without Triton.

### Stage 5 — Completion

`get_finished()` polls the CUDA end event non-blockingly ([`cpu/gpu_worker.py:419`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py#L419)) and reports `transfer_size`/`transfer_time` from the recorded start/end events — this is the source of the KV-transfer bandwidth metrics used in the offload experiment reports. The connector worker maps the job back to a request id and emits it in `finished_recving` ([`offloading/worker.py:346`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/worker.py#L346)), which un-parks the request.

### Stage 6 — Unpin

`update_connector_output` ([`offloading/scheduler.py:1280`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L1280)) decrements `pending_count`; when it reaches zero across all workers, `manager.complete_load()` drops `ref_cnt` and the blocks become evictable again ([`cpu/manager.py:153`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/manager.py#L153)).

---

## 3. Path B — block is only in a secondary tier (staged promotion)

This is the FS/NVMe/CephFS/object-store case, and the part most relevant to the offload experiments.

`TieringOffloadingManager.lookup` ([`tiering/manager.py:282`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L282)):

1. **Poll finished jobs first** ([`:314`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L314)), so a promotion that completed since the last call reads as `HIT` rather than stale `MISS`/`HIT_PENDING`, and blocks freed by completions are evictable in time for a promotion this same call may initiate. Gated to once per step by `_processed_jobs_this_step`.
2. **Ask the CPU primary tier.** `HIT`/`HIT_PENDING` short-circuits into Path A.
3. **On primary miss, walk the secondary tiers in order**, skipping any excluded by the request's `TierFilter`. A client can restrict this per-request via `kv_transfer_params`, e.g. `{"kv_load_tiers": [{"medium": "STORAGE", "locality": "LOCAL"}]}` — parsed at [`offloading/scheduler.py:387`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L387), matched at [`base.py:61-87`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/base.py#L61-L87).
4. **First `HIT` → `_initiate_promotion`** ([`tiering/manager.py:380`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L380)).

### What `_initiate_promotion` actually does

It **reserves the CPU slot immediately** but **defers the I/O**:

- `primary_tier.prepare_write([key])` allocates a CPU block with `ref_cnt = -1` (i.e. `is_ready == False`). This is load-bearing: any *subsequent* lookup of the same key in the same step sees `HIT_PENDING` from the primary tier and will not queue a duplicate promotion.
- If the CPU tier is full, `prepare_write` returns `None` → promotion fails → the block is reported **`MISS`**, not `RETRY`. Deliberate: better to recompute than to spin forever against a full primary tier.
- On success the `(keys, block_ids)` are accumulated into `_pending_load_submissions[tier][req_id]`; **no `submit_load` yet**.

`lookup` then returns `RETRY` → `_lookup` returns `None` → `get_num_new_matched_tokens` returns `None` → the core scheduler pops the request and *prepends it back onto the waiting queue untouched* ([`v1/core/sched/scheduler.py:783-789`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/core/sched/scheduler.py#L783-L789)). No GPU blocks are allocated for it this step.

### Flush at end of step

`on_schedule_end` ([`tiering/manager.py:728`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L728)) runs a fixed sequence per step:

1. catch-all `_maybe_process_finished_jobs()` (covers steps with no scheduled requests),
2. `serve_external_requests()` on each tier (lets P2P serve remote peers via the `ParentManager` facade, [`tiering/base.py:63`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/base.py#L63)),
3. reset the per-step polling gate,
4. **`_flush_pending_promotions()`** ([`tiering/manager.py:429`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L429)) — **one batched `submit_load` per (tier, request)**,
5. `on_schedule_end()` on each tier (which is where `AsyncLookupManager.flush()` fires).

Batching per (tier, request) is what turns "N chunk lookups" into one I/O job.

### Completion and hand-off

A later step's poll calls `tier.get_finished_jobs()`; `_process_finished_jobs` ([`tiering/manager.py:246`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L246)) dispatches on `JobMetadata.is_promotion`:

- **promotion** (secondary→primary): `primary_tier.complete_write()` flips `ref_cnt` from `-1` to `0` — the block is now a normal, evictable CPU resident.
- **cascade** (primary→secondary): `primary_tier.complete_read()` decrements the pin taken by `create_store_job`.

Next time the request is examined, the CPU lookup hits and **Path A runs normally**.

### Cost model

Let $s$ = engine steps. A cold secondary-tier hit costs at minimum:

$$
s_{\text{total}} \;\ge\; \underbrace{1}_{\text{async lookup returns RETRY}} \;+\; \underbrace{1}_{\text{promotion submitted at end of step}} \;+\; \underbrace{k}_{\text{tier I/O latency}} \;+\; \underbrace{1}_{\text{CPU}\to\text{GPU load}}
$$

where $k \ge 1$ depends on the tier's actual I/O latency relative to step duration. A CPU-tier hit skips the first three terms entirely. This step-quantized latency — not raw device bandwidth — is often what dominates TTFT for secondary tiers, and it is the reason a fast NVMe device does not automatically translate into a proportional TTFT win.

---

## 4. The async lookup subsystem

`FileSystemTierManager.lookup` never touches the filesystem on the scheduler thread. It delegates to `AsyncLookupManager` ([`tiering/async_lookup.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/async_lookup.py)), an abstract base whose only required method is `batch_lookup()`.

| Step | Thread | Code |
|---|---|---|
| `lookup(key)` — record key, return cached result or `None` | scheduler | [`:125`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/async_lookup.py#L125) |
| `flush()` — post the whole step's batch as one queue item | scheduler, from `on_schedule_end` | [`:147`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/async_lookup.py#L147) |
| `_worker()` — group by request, call `batch_lookup()` | background | [`:196`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/async_lookup.py#L196) |
| `drain_results()` — apply results lazily on next `lookup()` | scheduler | [`:160`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/async_lookup.py#L160) |

**There is no lock.** Thread safety comes from ownership: `_lookup_state`/`_lookup_batch` are scheduler-owned; `_lookup_queue` is scheduler-writes / worker-reads; `_pending_results` is worker-writes / scheduler-reads. Both are `queue.SimpleQueue`, safe for single-writer/single-reader. The design rationale is written out in the module docstring ([`:10-31`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/async_lookup.py#L10-L31)).

For FS, `batch_lookup` prefers the `vllm.fs_io_C` C extension, which **releases the GIL for the entire `faccessat()` batch** ([`fs/manager.py:76-83`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/manager.py#L76-L83)), falling back to `os.path.exists` per key. On a network filesystem this is the difference between a scheduler stall and a background batch.

**Consequence:** the first lookup of a cold key *always* returns `RETRY` and costs the request one step, even for a key that is present. That is the intended trade — bounded per-step scheduler latency in exchange for a one-step admission delay.

A related subtlety in `_maximal_prefix_lookup`: the scan deliberately **does not break on `RETRY`** ([`offloading/scheduler.py:568-576`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L568-L576)). It keeps scanning to seed as many async lookups as possible before the first true `MISS`, so the next step has a warm result set.

---

## 5. Per-tier load implementations

| Tier | `submit_load` does | Code |
|---|---|---|
| `fs` | `batch_load_block(paths, primary_kv_view, byte_offsets, block_size, use_o_direct)` queued on a dual-queue thread pool | [`fs/manager.py:221`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/manager.py#L221) |
| `obj` | NIXL `READ` transfer keyed by object name | [`obj/manager.py:283`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/obj/manager.py#L283) |
| `p2p` | RDMA read from a peer session; requires `P2PSourceInfo` in the request context, else the job fails immediately | [`p2p/manager.py:473`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/p2p/manager.py#L473) |

### FS specifics (the NVMe/CephFS path)

- **Thread pool** is `DualQueueThreadPool` ([`fs/thread_pool.py:50`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/thread_pool.py#L50)): read-priority threads (`n_read_threads`, default 16) and write-priority threads (`n_write_threads`, default 16). Both groups can drain either queue, so **neither loads nor stores starve** — but the default split is what determines read/write contention under mixed load.
- **O_DIRECT is probed once at init** ([`fs/manager.py:178`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/manager.py#L178), [`fs/io.py:41`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/io.py#L41)) and falls back to buffered I/O with a warning on filesystems that reject it (overlayfs, some NFS mounts). **Worth checking in any benchmark log** — a silent fallback to buffered I/O changes the page-cache story completely and can flatter or distort results.
- **Read path** ([`fs/io.py:192`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/io.py#L192)) slices the memoryview per block and hands the slices to the C extension, or falls back to a Python loop. It **raises on first error and removes the offending file** — corrupt/truncated blocks self-heal by deletion.
- **File layout** ([`file_mapper.py:107`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/file_mapper.py#L107)): `<base>_r<rank>/<hash[:3]>/<hash[3:5]>_g<group_idx>/<hash>.bin`, hash-fanned to limit directory width. Base path embeds model name + a config hash so incompatible runs cannot collide.
- **Cross-instance sharing requires a fixed `PYTHONHASHSEED`** on every instance ([`fs/manager.py:97-104`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/manager.py#L97-L104)). Without it, `NONE_HASH` is seeded randomly per process, so identical token content produces **different filenames** and a shared PVC yields a 0% cross-instance hit rate that looks exactly like a working-but-cold cache. This is a top-tier gotcha for multi-replica llm-d deployments.

---

## 6. Invariants and safety mechanisms

| Invariant | Why | Code |
|---|---|---|
| Every read path pins via `ref_cnt` | Eviction only considers `ref_cnt == 0` blocks; a block being DMA'd or written to storage cannot be reused underneath | [`cpu/manager.py:186-209`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/manager.py#L186-L209) |
| `ref_cnt = -1` means "write in flight" | Distinguishes "allocated but not yet valid" from "resident"; surfaces as `HIT_PENDING`. Set in `BlockStatus.__init__`; `is_ready` is defined as `ref_cnt >= 0` | [`cpu/policies/base.py:20-33`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/policies/base.py#L20-L33) |
| One load **or** one-or-more stores per request, never both | Prevents cross-direction races on the same request's blocks | [`:960`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L960), [`:1176`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L1176) |
| A request with in-flight transfers is deferred outright | Same reason | [`:842`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L842) |
| `_chunks_being_loaded` suppresses duplicate loads | Two requests sharing a prefix must not both pull it; the second is delayed instead | [`:490`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L490), [`:769-793`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L769-L793) |
| `jobs_to_flush` fences GPU block reuse | If the KV cache manager re-allocates a GPU block with a pending store, the worker is told to `wait()` on that job before the block is overwritten | [`:1237-1249`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L1237-L1249), [`worker.py:292`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/worker.py#L292) |
| `reset_cache` drains tiers before resetting primary | A tier mid-`readv` into primary memory would corrupt it; a stuck tier blocks *visibly* here rather than corrupting silently | [`tiering/manager.py:779-819`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L779-L819), [`tiering/base.py:286`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/base.py#L286) |
| Secondary tiers are **not** reset on `reset_cache` | Persistent stores (FS, object) keep data across sleep/wake and weight updates | [`tiering/manager.py:788-791`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L788-L791) |

### The store side, briefly (it explains what is available to retrieve)

Fan-out on store is **unconditional**: `complete_store` cascades newly-stored blocks to *all* secondary tiers ([`tiering/manager.py:620-622`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py#L620-L622)). So a block that reached CPU eventually exists on FS too, and the promotion path only ever pulls back what has since fallen out of the CPU tier's LRU/ARC. Stores are also **deferred to the next engine step** on purpose ([`worker.py:332-344`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/worker.py#L332-L344)) so offloading starts *after* the transfers related to token sampling, avoiding delays to token generation.

---

## 7. Observability

Metrics that report on this path, and where they come from:

| Metric | Source |
|---|---|
| `vllm:kv_offload_tiering_lookup_sync_delay_seconds` | Blocking time spent querying secondary tiers per request, accumulated until allocation/finish — [`tiering/spec.py:86`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/spec.py#L86) |
| `vllm:kv_offload_tiering_lookup_async_delay_seconds` | Wall-clock from first *deferred* secondary lookup to allocation/finish — i.e. **the promotion stall** — [`tiering/spec.py:108`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/spec.py#L108) |
| Load/store bytes, time, size histograms | From CUDA event timings, aggregated at [`offloading/scheduler.py:1292-1320`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L1292-L1320) |
| CPU cache usage / read-usage / write-usage gauges | [`cpu/manager.py:293-327`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/manager.py#L293-L327) |
| `BlockStored` / `BlockRemoved` KV events | Per-tier, aggregated by `take_events()` — see [[vLLM KV Events canonical form]] |

**The async-delay histogram is the one to watch when evaluating a secondary tier.** It isolates the step-quantized promotion stall from raw device throughput, which is exactly the gap that device-level metrics (NVMe IOPS, CephFS throughput) cannot explain on their own.

---

## 8. Gotchas

1. **`RETRY` is not an error.** It means "promotion started" or "async lookup not yet resolved". Treating it as a failure signal misreads the whole state machine.
2. **`HIT_PENDING` on the CPU tier means a store is in flight**, not a promotion.
3. **A full CPU tier degrades secondary tiers to useless**, silently. `_initiate_promotion` returns `MISS` when `prepare_write` fails, so an undersized `cpu_bytes_to_use` makes an FS/NVMe tier look like a cache miss rather than a capacity problem. Cross-check the CPU usage gauge before blaming the storage device.
4. **Secondary tier I/O consumes scheduler-process CPU.** The threads live in the scheduler process, so thread counts and I/O stalls compete with scheduling work, not with model execution.
5. **Without a fixed `PYTHONHASHSEED`, cross-instance FS sharing silently yields 0% hits.**
6. **O_DIRECT can silently fall back to buffered I/O.** Grep the startup log.
7. **The store path skips SWA chunks that can never serve a load hit** ([`is_store_reachable_swa_chunk`, `offloading/scheduler.py:122`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L122)). On hybrid models (e.g. MLA + SWA) the offloaded byte volume is therefore *not* simply proportional to token count.
8. **EAGLE/MTP draft groups exclude their trailing chunk** from both store and load scheduling — its draft-layer KV can be rewritten after spec-token rejection, so it has no stable hash.

---

## 9. Code map

| Concern | File |
|---|---|
| Interfaces, key packing, lookup states | [`vllm/v1/kv_offload/base.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/base.py) |
| Scheduler-side connector: lookup, job building | [`.../offloading/scheduler.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py) |
| Worker-side connector: submit/poll | [`.../offloading/worker.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/worker.py) |
| Connector glue (`start_load_kv`, `wait_for_save`) | [`.../v1/offloading_connector.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading_connector.py) |
| Multi-tier orchestration, promotion | [`vllm/v1/kv_offload/tiering/manager.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/manager.py) |
| Secondary tier interface | [`vllm/v1/kv_offload/tiering/base.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/base.py) |
| Async existence checks | [`vllm/v1/kv_offload/tiering/async_lookup.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/async_lookup.py) |
| CPU primary tier (pool, ref_cnt, eviction) | [`vllm/v1/kv_offload/cpu/manager.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/manager.py) |
| Eviction policies (`lru`, `arc`) and `BlockStatus` | [`vllm/v1/kv_offload/cpu/policies/`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/policies/base.py) |
| GPU↔CPU DMA | [`vllm/v1/kv_offload/cpu/gpu_worker.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py) |
| Shared `/dev/shm` region | [`vllm/v1/kv_offload/cpu/shared_offload_region.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/shared_offload_region.py) |
| FS tier + I/O + thread pool | [`fs/manager.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/manager.py) · [`fs/io.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/io.py) · [`fs/thread_pool.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/fs/thread_pool.py) |
| Tier registry (`example`/`fs`/`p2p`/`obj`) | [`vllm/v1/kv_offload/tiering/factory.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/factory.py) |
| Core scheduler integration | [`vllm/v1/core/sched/scheduler.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/core/sched/scheduler.py) |

### Tests that exercise these state machines

- [`tests/v1/kv_offload/tiering/test_tiering_offloading.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/tests/v1/kv_offload/tiering/test_tiering_offloading.py) — promotion/cascade sequencing
- [`tests/v1/kv_offload/tiering/test_async_lookup.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/tests/v1/kv_offload/tiering/test_async_lookup.py) — lookup queueing
- [`tests/v1/kv_offload/tiering/test_fs_tier.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/tests/v1/kv_offload/tiering/test_fs_tier.py) — FS tier
- [`tests/v1/kv_connector/unit/test_offloading_connector.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/tests/v1/kv_connector/unit/test_offloading_connector.py) — connector-level

---

## 10. Open threads

- The **step-quantization cost** of promotion (§3) is a structural claim read off the code, not a measurement. It would be worth confirming against the `lookup_async_delay` histogram in an existing NVMe/CephFS run to see how many steps a promotion actually takes under load.
- Whether the fixed 16/16 read/write thread split in the FS tier is a bottleneck at high concurrency is untested here.
- Nothing in this path does **admission-time prioritization** across tiers by expected latency — the first tier in config order that hits wins. Whether ordering tiers by latency measurably helps is an open question.

## Provenance

Direct source reading of `vllm-project/vllm` at `4ee9702bee668a447e9983a6aefc16ebbc3ad32e` (local checkout, verified present on `upstream/main`), 2026-08-01. Supersedes the promotion/lookup section of [[vLLM KV block prefetch architecture]], which described `HIT_PENDING` and `RETRY` inaccurately and attributed secondary→CPU transfers to the worker process.