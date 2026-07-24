# vLLM KV block prefetch architecture

vLLM has three distinct KV block prefetch mechanisms. They differ in what they move, which tiers they cover, and how they coordinate with the scheduler.

## 1. TieringOffloadingManager (primary native system)

**Location:** `vllm/v1/kv_offload/`

**Tiers:** GPU ↔ CPU (primary) ↔ Secondary (filesystem, P2P, object store)

### Offload path (GPU → CPU → secondary)

When GPU KV blocks are evicted, `prepare_store()` allocates a CPU slot, transfers asynchronously via CUDA stream, then cascades to secondary tiers via `submit_store()`.

### Promotion/prefetch path (secondary → CPU → GPU)

1. `lookup()` checks the CPU primary tier first (fast, pinned memory)
2. On primary miss, queries secondary tiers via `AsyncLookupManager` — a background thread that batches existence checks **without blocking the scheduler**
3. On secondary hit, `_initiate_promotion()` allocates a CPU slot and defers `submit_load()`
4. `_flush_pending_promotions()` submits batched load jobs **at end of scheduler step**
5. Worker copies from secondary tier → CPU → GPU

### Lookup return states

| State | Meaning |
|-------|---------|
| `HIT` | Block found in tier, ready to use |
| `HIT_PENDING` | Block found, promotion in progress |
| `RETRY` | Transient failure, try again next step |
| `MISS` | Block not found in any tier |

### Data transfer

- GPU ↔ CPU: CUDA DMA copy engine
- Small aligned payloads: Triton kernels
- CPU memory: `mmap`-backed, registered with `cudaHostRegister` for pinned DMA

### Key source files

| File | Role |
|------|------|
| `vllm/v1/kv_offload/tiering/manager.py` | `TieringOffloadingManager` orchestrator |
| `vllm/v1/kv_offload/tiering/async_lookup.py` | Non-blocking secondary tier lookups |
| `vllm/v1/kv_offload/cpu/manager.py` | CPU primary tier manager |
| `vllm/v1/kv_offload/cpu/gpu_worker.py` | GPU ↔ CPU transfers (Triton kernel + DMA) |
| `vllm/v1/kv_offload/tiering/fs/manager.py` | Filesystem secondary tier |
| `vllm/v1/kv_offload/tiering/p2p/manager.py` | P2P network secondary tier |

### Configuration

```bash
--kv-offloading-size <float>         # GiB buffer (activates offloading)
--kv-offloading-backend native|lmcache
```

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

Prefetches **model weights** (not KV blocks) from CPU to GPU with pipelining. Triggered by layer forward hooks. Configured via `--offload-group-size` and `--offload-prefetch-step`. This is a separate system from the KV cache tiering.

## Summary

| Mechanism | What it moves | Tiers | Trigger | Async? |
|-----------|--------------|-------|---------|--------|
| TieringOffloadingManager | KV cache blocks | GPU↔CPU↔FS/P2P | Scheduler step (lookup miss) | Yes, CUDA streams |
| LMCache MP Connector | KV cache blocks | GPU↔LMCache server | Scheduler step (APC delta) | Yes, ZMQ |
| Weight Prefetch | Model parameters | CPU↔GPU | Layer forward hook | Yes, copy_stream |

**Design principle:** The native `TieringOffloadingManager` ensures the scheduler never blocks on remote tier lookups — `AsyncLookupManager` batches existence checks in a background thread, and promotion loads are deferred to end of step via `_flush_pending_promotions()`.

## Source

- Session ses_06baf2392ffe7BXoPnFTG2MiZV (2026-07-24): Code inspection of vllm-project/vllm KV block prefetch mechanisms via GitHub connector.
