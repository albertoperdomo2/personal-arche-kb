# vLLM KV block prefetch architecture

vLLM has three distinct KV block prefetch mechanisms. They differ in what they move, which tiers they cover, and how they coordinate with the scheduler.

> **Deep dive:** for the native tiering system, [[vLLM KV offload retrieval path - lookup, promotion, and load]] documents the full retrieval path with code permalinks pinned to a specific commit. Read that one when working on the code; this note is the three-way comparison.

## 1. TieringOffloadingManager (primary native system)

**Location:** `vllm/v1/kv_offload/`

**Tiers:** GPU ↔ CPU (primary) ↔ Secondary (filesystem, P2P, object store)

### Offload path (GPU → CPU → secondary)

As chunks become complete (all their GPU blocks allocated and computed), `prepare_store()` allocates a CPU slot and the worker transfers GPU→CPU asynchronously via CUDA stream. On `complete_store()` the blocks are cascaded to **all** secondary tiers via `submit_store()`. This is not eviction-triggered — it runs per scheduler step as chunks complete.

### Promotion path (secondary → CPU → GPU)

Note this is **demand-driven retrieval, not speculative prefetch** — nothing is fetched on a guess about future traffic. `grep -i prefetch` over `vllm/v1/kv_offload/` returns nothing.

1. `lookup()` checks the CPU primary tier first (fast, pinned memory)
2. On primary miss, queries secondary tiers via `AsyncLookupManager` — a background thread that batches existence checks **without blocking the scheduler**
3. On secondary hit, `_initiate_promotion()` reserves a CPU slot (`ref_cnt = -1`) and defers `submit_load()`
4. `_flush_pending_promotions()` submits one batched load job per (tier, request) **at end of scheduler step**
5. The **scheduler process** copies secondary → CPU (tier threads writing into the shared `/dev/shm` mmap); the **worker process** then copies CPU → GPU. Secondary tiers never touch GPU memory.

### Lookup return states

| State | Meaning |
|-------|---------|
| `HIT` | Block present in the tier and ready to read |
| `HIT_PENDING` | Slot allocated but its **GPU→CPU store is still in flight** (`ref_cnt = -1`) |
| `RETRY` | **Not an error** — a promotion was just initiated, or an async lookup has not resolved yet. Retry next step |
| `MISS` | Not found in any tier — *or* found in a secondary tier but the CPU primary tier was full, so promotion could not start |

### Data transfer

- GPU ↔ CPU: CUDA DMA copy engine
- Small (`< THRESHOLD_BYTES`), 8-byte-aligned payloads, CPU→GPU only: Triton kernels
- CPU memory: `mmap`-backed in `/dev/shm`, shared by scheduler and workers, registered with `cudaHostRegister` for pinned DMA

### Key source files

| File | Role |
|------|------|
| `vllm/v1/kv_offload/tiering/manager.py` | `TieringOffloadingManager` orchestrator |
| `vllm/v1/kv_offload/tiering/async_lookup.py` | Non-blocking secondary tier lookups |
| `vllm/v1/kv_offload/cpu/manager.py` | CPU primary tier manager |
| `vllm/v1/kv_offload/cpu/gpu_worker.py` | GPU ↔ CPU transfers (Triton kernel + DMA) |
| `vllm/v1/kv_offload/cpu/shared_offload_region.py` | Shared `/dev/shm` mmap region |
| `vllm/v1/kv_offload/tiering/fs/manager.py` | Filesystem secondary tier |
| `vllm/v1/kv_offload/tiering/p2p/manager.py` | P2P network secondary tier |
| `vllm/v1/kv_offload/tiering/obj/manager.py` | Object store secondary tier |

### Configuration

```bash
--kv-offloading-size <float>         # GiB buffer (activates offloading)
--kv-offloading-backend native|lmcache
```

Registered secondary tier types: `example`, `fs`, `p2p`, `obj`. See [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]].

## 2. LMCache MP Connector (distributed prefetch)

**Location:** `vllm/distributed/kv_transfer/kv_connector/v1/lmcache_mp_connector.py`

Uses an external LMCache server over ZMQ for distributed KV block prefetch.

### State machine

```
PREFETCHING → WAITING_FOR_LOAD → READY
```

1. Scheduler calls `get_num_new_matched_tokens()` → submits lookup to LMCache server
2. If LMCache has more cached blocks than vLLM's APC, the delta becomes `num_external_computed_tokens`
3. Blocks allocated, transitions to `WAITING_FOR_LOAD`
4. Worker submits batched `RETRIEVE` requests via ZMQ to LMCache server
5. LMCache server performs async GPU↔GPU or GPU↔CPU KV block transfers

## 3. Model weight prefetch (not KV blocks)

**Location:** `vllm/model_executor/offloader/prefetch.py`

Prefetches **model weights** (not KV blocks) from CPU to GPU with pipelining. Triggered by layer forward hooks. Configured via `--offload-group-size` and `--offload-prefetch-step`. This is a separate system from the KV cache tiering, and the only one of the three that is genuinely speculative.

## Summary

| Mechanism | What it moves | Tiers | Trigger | Async? |
|-----------|--------------|-------|---------|--------|
| TieringOffloadingManager | KV cache blocks | GPU↔CPU↔FS/P2P/obj | Scheduler step (admission-time lookup hit) | Yes, CUDA streams + tier threads |
| LMCache MP Connector | KV cache blocks | GPU↔LMCache server | Scheduler step (APC delta) | Yes, ZMQ |
| Weight Prefetch | Model parameters | CPU↔GPU | Layer forward hook | Yes, copy_stream |

**Design principle:** The native `TieringOffloadingManager` ensures the scheduler never blocks on remote tier lookups — `AsyncLookupManager` batches existence checks in a background thread, and promotion loads are deferred to end of step via `_flush_pending_promotions()`. The cost of that non-blocking guarantee is that a cold secondary-tier hit is **step-quantized**: it takes several engine steps before the request can run.

## Source

- Session ses_06baf2392ffe7BXoPnFTG2MiZV (2026-07-24): Code inspection of vllm-project/vllm KV block prefetch mechanisms via GitHub connector.
- 2026-08-01: Section 1 corrected against a local checkout at `4ee9702` — `HIT_PENDING`/`RETRY` semantics, the process that performs secondary→CPU transfers, and the store trigger were all inaccurate as originally written. Details in [[vLLM KV offload retrieval path - lookup, promotion, and load]].