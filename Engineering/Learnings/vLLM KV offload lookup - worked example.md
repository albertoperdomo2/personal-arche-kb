---
title: vLLM KV offload lookup - worked example
date: 2026-08-08
type: learning
topic: KV cache offloading
repo: vllm-project/vllm
commit: 0601850791155003afbe5a0d5d086350cada8deb
commit_date: 2026-08-02
---

# vLLM KV offload lookup — a worked example

How the KV-offload lookup path works, explained as a single request traced end-to-end through the actual code. This is a **tutorial companion** to [[vLLM KV offload retrieval path - lookup, promotion, and load]]: that article is the reference map (every function, every invariant, the cost model), this one is the guided tour — it spends a full section on each concept that the reference map assumes you already know (block, chunk, hash, group, `ref_cnt`, engine step), and it narrates *why* the system is shaped the way it is, not just what it does.

All code links are pinned to `vllm-project/vllm@0601850` (`0601850791155003afbe5a0d5d086350cada8deb`, 2026-08-02, on `upstream/main`), so line anchors stay correct as `main` moves. This covers the **native** offloading path only (the `fs` tier standing in for NVMe/CephFS); the LMCache connector and model-weight prefetch are out of scope (see [[vLLM KV block prefetch architecture]]).

---

## 0. What "lookup" is — and is not

Before any code, get the framing right, because the word "lookup" suggests the wrong thing.

A lookup is **not** a search over a database, and it is **not** a prefetch. It is an **existence check performed at request admission time**, in the **scheduler process**, asking one question per chunk of the prompt:

> "Is this chunk's KV stored in one of the offload tiers, and if so, can I have it back?"

The answer is one of four verdicts — `HIT`, `HIT_PENDING`, `RETRY`, `MISS` — and the verdict determines how many prompt tokens the scheduler can skip recomputing.

Two design goals explain almost everything you are about to read:

1. **The scheduler thread must never block on storage.** A `stat()` on CephFS or an NVMe read can take milliseconds; blocking the scheduler thread stalls *every* request in the engine. So secondary-tier existence checks run on a **background thread, one batch per engine step**, and I/O is deferred to the end of the step. The price is that a cold lookup can never be answered in the same step it is asked — it costs at least one engine step of waiting. That is a deliberate trade, and the whole document is essentially the anatomy of that trade.
2. **KV bytes are never copied between processes.** The FS tier's read threads write directly into a shared `/dev/shm` mapping that the worker process's DMA engine reads from. Storage I/O happens in the scheduler process; the GPU copy happens in the worker process; the bytes rendezvous in one pinned memory region.

---

## 1. The mental model

### 1.1 Engine steps — the clock everything runs on

vLLM's V1 scheduler runs a loop: each iteration is one **engine step** — one scheduling pass followed by one forward pass (and, on the same step, the start of the next model execution). Requests are admitted from a waiting queue at the start of each step.

Two consequences that matter for everything below:

- A request that is not scheduled in a step simply **waits for the next step**. The waiting queue preserves its position — it is *prepended back* rather than pushed to the back, so it is not penalized behind newer arrivals (code in §3, Step 1e).
- **All latency in this system is quantized to steps.** A promotion that needs 3 ms of I/O and a promotion that needs 30 ms both cost "a few engine steps" if the step duration is ~10 ms; below that granularity, raw device bandwidth is nearly invisible. This is the single most important fact for performance reasoning about secondary tiers, and it is why §5 measures the cold path in steps, not milliseconds.

### 1.2 The tiers

The offloading stack has three conceptual levels:

```
GPU KV cache  (worker process — owns the tensors, does the math)
    ▲
    │  CPU→GPU and GPU→CPU DMA on a dedicated pooled CUDA stream
    │
CPU primary tier  —  a staging pool in a /dev/shm file,
    ▲              mapped by BOTH the scheduler process and every worker
    │
    │  secondary→CPU promotion (tier threads / NIXL)
    │  CPU→secondary cascade (tier threads / NIXL)
    │
   ┌──────────┬──────────┬──────────┐
 "fs"        "obj"      "p2p"     "example"
 FS/NVMe    object     remote     test
 (CephFS)    store      peer       tier
```

Registered secondary tier types are `example`, `fs`, `p2p`, `obj` (`vllm/v1/kv_offload/tiering/factory.py:55-77`).

**The CPU tier is a mandatory gateway.** This is stated as an invariant on the interface itself (`tiering/base.py:103-107`): store always flows `GPU → CPU → secondary` (a *cascade*), load always flows `secondary → CPU → GPU` (a *promotion*). A secondary tier can never touch GPU memory directly. Why? Two reasons:

- The secondary tiers are plain Python managers running in the scheduler process; they have no access to GPU tensors and no CUDA context.
- The `/dev/shm` region (§1.3) is the rendezvous: a promotion's read lands in the exact buffer the worker's DMA engine will later source from. Enforcing the gateway makes that handoff a *design invariant* rather than an accident.

Why offload at all? GPU memory is the scarce resource; KV cache for long prompts and large batches is gigabytes. Offloading moves KV that is not currently needed down the hierarchy (GPU → CPU RAM → NVMe/CephFS/object store), and pulls it back when a request needs it — the same memory-hierarchy idea as CPU page cache, applied to KV.

### 1.3 The `/dev/shm` region — how the split-brain works

The CPU tier lives in a single `SharedOffloadRegion` (`cpu/shared_offload_region.py:28`): one file, `/dev/shm/vllm_offload_{engine_id}.mmap`, **mmapped by every process**. Workers race to create it with `O_EXCL`; the winner `ftruncate`s it and the rest wait for the expected size.

This one trick makes the whole two-process design work:

- The **scheduler side** creates a `primary_kv_view` into the region and hands it to each secondary tier at construction (`tiering/spec.py`). An FS read thread doing `os.readv` into `primary_kv_view[block_id]` is filling memory that *the worker process* will DMA from. No KV bytes are ever sent over IPC.
- The **worker side** carves its KV tensors out of the same region, and the whole region is registered with `cudaHostRegister` for pinned DMA (`cpu/gpu_worker.py:124-146`). Pinned memory is what lets the copy engine DMA without bounce buffers. If registration fails (e.g., exceeding `RLIMIT_MEMLOCK`), it degrades to unpinned DMA with a warning rather than erroring out — worth grepping for in logs.

### 1.4 The units: block, hash, chunk, key, group

This is the vocabulary the whole walkthrough uses, and the note's tersest section, so it gets the full treatment here.

**GPU block.** The KV cache on GPU is a paged cache: memory is carved into fixed-size *pages*, each holding `tokens_per_block` tokens' worth of KV for one model group (typically 16). A block is the unit of GPU memory management — allocation, prefix caching, and DMA all operate on blocks. A block is also the unit of *content identity*: two requests that have computed the same tokens have identical block contents, which is what makes reuse possible.

**Chain hash / block hash.** vLLM's prefix cache is content-addressed. For every token position $t$, there is a hash of the token *content* of the prefix $[0, t]$ — a "chain hash", so that the hash at position $t$ is a function of the hash at $t-1$ and the token at $t$. Identical token sequences therefore produce identical hashes, in any request, on any engine (given the same seed — see the `PYTHONHASHSEED` gotcha in §7). The hash of the last token of a block *is* the block's identity: if request B's block hash matches request A's, B can reuse A's KV without recomputing.

**Chunk.** The offloading unit. A chunk is `blocks_per_chunk` (default 4) *consecutive GPU blocks*, so:

$$
\text{tokens\_per\_chunk} = \text{tokens\_per\_block} \times \text{blocks\_per\_chunk}
$$

Why group blocks into chunks at all? Four reasons, all visible in the code:

1. **Amortized I/O.** One chunk = one file on the FS tier and one read job; moving 4× the bytes per job amortizes syscall and job-bookkeeping overhead.
2. **Amortized DMA.** The worker issues one batched copy per chunk-group; fewer, larger copies beat many small ones.
3. **Reduced metadata.** The offload index is keyed by chunk, not by block — 4× fewer keys, lookups, and hash-map entries.
4. **Hash stability.** A chunk's key is the chain hash of its *last* token (§1.5), which is only meaningful for full, unchanging chunks — this matters for EAGLE draft groups (§5.4).

**KV group and group index.** A model does not have one KV cache; it has one *per KV group* (each group covers a set of layers sharing a block geometry). A plain dense model has one group. Hybrid models have several: DeepSeek-style architectures have a full-attention group (e.g., MLA) *and* sliding-window groups with different block sizes; Mamba models have state groups; EAGLE/MTP draft models have an extra draft-attention group. The same token position produces *different KV bytes in each group* — a latent vector in MLA vs. windowed KV in the SWA group — so the cache must keep them separate. The **group index** is the group's ordinal in the model's KV-cache spec; it is what makes the same token content distinguishable across groups.

**OffloadKey.** The identity of one stored chunk = (chunk hash, group index), packed as raw bytes:

```python
# vllm/v1/kv_offload/base.py:26-41
OffloadKey = NewType("OffloadKey", bytes)

def make_offload_key(block_hash: bytes, group_idx: int) -> OffloadKey:
    """Pack a block hash and group index into an `OffloadKey`."""
    return OffloadKey(block_hash + group_idx.to_bytes(4, "big", signed=False))
```

Why bytes instead of a `(hash, group_idx)` tuple? The lookup path calls `manager.lookup()` **once per chunk per step**, and tuples would allocate on every call — the comment at `base.py:23-25` says it outright: "encoded as raw bytes to avoid tuple GC overhead." Bytes are immutable, hashable, compact, and can be written straight to a filename or shared memory. `get_offload_block_hash` / `get_offload_group_idx` split it back apart.

**Key derivation — why the hash of the *last* token of the chunk.** `update_offload_keys` builds the keys lazily at every admission attempt (`offloading/scheduler.py:312-326`):

```python
for req_block_hash in islice(
    self.req.block_hashes,
    group_config.hashes_per_chunk * len(group_state.offload_keys)
    + group_config.hashes_per_chunk - 1,   # start at the last hash of the current chunk
    None,
    group_config.hashes_per_chunk,          # stride of one chunk
):
    group_state.offload_keys.append(
        make_offload_key(req_block_hash, group_config.group_idx)
    )
```

For chunk $i$ it takes the chain hash at token $(i+1)\cdot\text{tokens\_per\_chunk} - 1$ — the hash of the prefix *ending at the chunk's last token* — because that hash identifies the chunk's entire content. Note the consequence for a **trailing partial chunk**: a 400-token prompt (7 chunk windows of 64) only produces 6 keys, because token 399 is not at a stride-aligned position (the 7th key would need a hash at token 447, which doesn't exist). The last partial chunk is simply never looked up — it is always computed fresh. This is a detail most writeups miss, and it matters for the example in §3.

### 1.5 `ref_cnt` — the safety mechanism that makes cross-process sharing safe

Every CPU block has a `BlockStatus` (`cpu/policies/base.py:10-33`):

```python
class BlockStatus(ctypes.Structure):
    # ref_cnt - the current number of transfers using this block as a source.
    _fields_ = [("ref_cnt", ctypes.c_int32), ("block_id", ctypes.c_int64)]

    # initialize block as "not ready" (ref_cnt = -1)
    self.ref_cnt = -1

    @property
    def is_ready(self) -> bool:
        return self.ref_cnt >= 0
```

Three regimes, three meanings:

| `ref_cnt` | Meaning | Surfaces as |
|---|---|---|
| `-1` | Slot allocated, **write in flight** (a store or promotion hasn't landed yet). Not readable. | `HIT_PENDING` |
| `0` | Resident and valid. **Evictable** — the eviction policy only considers these. | `HIT` |
| `\ge 1` | **Pinned** by one or more active transfers (being DMA'd, being read to storage, being loaded to GPU). Eviction skips these. | `HIT` |

Why is pinning *necessary* rather than a nice-to-have? Because the transfer that reads a block and the code that might evict it run in *different processes*. The scheduler-process FS thread is `readv`-ing into a CPU block's buffer; the worker-process DMA engine is copying out of it; meanwhile the scheduler's own eviction policy could, without pinning, hand that block to a new store. `ref_cnt` is the cross-process "in use" flag. The two transitions that mutate it matter: `prepare_load` increments 0→1 (pin for a read), `complete_load` decrements back 1→0 (release); `prepare_write` sets -1 (write in flight), `complete_write` flips -1→0 (now resident). (Code: `cpu/manager.py:130-163`, plus the eviction-side guards in `cpu/policies/lru.py` and `cpu/policies/arc.py`.)

### 1.6 `LookupResult` — the four-state verdict

```python
# vllm/v1/kv_offload/base.py:107-113
class LookupResult(Enum):
    MISS = auto()
    HIT = auto()
    HIT_PENDING = auto()
    RETRY = auto()
```

| State | Meaning | When the caller sees it |
|---|---|---|
| `HIT` | Block present in the CPU primary tier and `is_ready`. | The fast path (§4, Path A). |
| `HIT_PENDING` | Block present, but a **GPU→CPU store is still in flight** (`ref_cnt == -1`). Not readable yet. | A block currently being offloaded that a new request needs. **Not** "promotion in progress" — that distinction matters when debugging. |
| `RETRY` | "I don't know yet" or "promotion started". | First lookup of a cold key (async existence check unresolved), or a secondary-tier hit whose promotion has been staged. The request will be re-examined next step. |
| `MISS` | Block nowhere in the tiers, **or** a promotion could not be staged because the CPU tier is full. | True absence, or CPU-capacity failure. |

The last line is subtle and worth stating loudly: `MISS` is also returned when a secondary-tier hit *cannot be promoted* because `prepare_write` found no free CPU slot. The designers deliberately chose `MISS` over `RETRY` here (comment at `tiering/manager.py:409-412`): better to recompute than to spin forever against a full primary tier. The practical consequence is a classic misdiagnosis: an undersized `cpu_bytes_to_use` makes a perfectly healthy NVMe/CephFS tier *look like a cache miss* in the hit-rate metrics. Check the CPU-usage gauge before blaming the storage device (§7).

---

## 2. The scenario — request R and the state of the world

### 2.1 The cast

The engine has been serving traffic. Among all the requests it has seen, three matter for our story:

- **R** — the request we trace. A 400-token prompt.
- **R′** — an earlier request with R's *exact* 400-token prompt. It was **preempted at token 256** (preemption happens when the KV cache needs space, or higher-priority work arrives; a preempted request's GPU blocks are released early).
- **a burst of follow-up requests** that shared R's first 96 tokens (think: a reused conversation head / system prompt).

### 2.2 The geometry — the numbers we will use

Engine config: `tokens_per_block = 16`, `blocks_per_chunk = 4`, prefix caching enabled, native offloading with an `fs` tier. That fixes everything else:

| Quantity | Value | Derivation |
|---|---|---|
| Prompt length of R | 400 tokens | |
| `tokens_per_block` | 16 | engine config |
| `blocks_per_chunk` | 4 | offload config (default) |
| `tokens_per_chunk` | 64 | $16 \times 4$ |
| `hashes_per_chunk` | 64 | $\text{tokens\_per\_chunk} / \text{tokens\_per\_hash}$, one hash per token |
| Offload keys of R | 6 (`k0`…`k5`) | 400 tokens → 6 stride-aligned chunk keys (the 16-token tail has no key, §1.4) |
| Local prefix-cache hit | 100 tokens | `get_computed_blocks` result (see below) |
| **Block-aligned** local hit | **96 tokens** | $100 - (100 \bmod 16)$ — see Step 1a |
| Stored chunks on FS | chunks 0..3 | tokens 0..255 (from R′) |
| GPU blocks after load | 16 | $\lceil 256/16 \rceil$ |
| GPU blocks to load | 10 | blocks 6..15 (tokens 96..255) |
| Tokens computed fresh | 144 | tokens 256..399 |

### 2.3 The state of the world — and *why* it looks like that

**Why does the GPU prefix cache know tokens 0..95?**

The chain of events: R′ shared the prompt with R. When R′ was preempted at token 256, its GPU blocks were released — but before that, the *first* six blocks of the prefix (tokens 0..95) had been reused by the burst of follow-up requests sharing the conversation head. Prefix caching keeps blocks alive by content hash after their owning requests finish, and the prefix-cache LRU had just touched those six blocks, so they are still **resident in GPU memory**. The scheduler's `get_computed_blocks` walks the hash→block map and reports **100 tokens** — six full blocks plus a partially filled seventh (tokens 96..99 live in block 6, which holds tokens 96..111).

**Why do the offload tiers hold chunks 0..3 (tokens 0..255)?**

When R′ was preempted, every released GPU block was **intercepted by the offload connector** — vLLM's offloading doesn't wait for an explicit "offload now" command; it hooks block release and stores released blocks. Each released block is copied GPU→CPU, and because the CPU tier *cascades* everything to secondaries (`tiering/manager.py:620-622` — fan-out is unconditional, so a block that reached CPU eventually exists on FS too), chunks 0..3 now sit on disk in the hash-fanned layout (§4, Step 2e).

**Why is the CPU tier empty of these blocks?**

The CPU tier is a **staging pool**, sized by `cpu_bytes_to_use` for in-flight transfers, not long-term residence. Its own LRU/ARC policy has since evicted chunks 0..3 down to the FS tier. This is the intended cascade: CPU holds data briefly; FS is the durable level. For our walkthrough it is what makes the story interesting — the CPU tier answers `MISS`, so the lookup must descend one level and stage a **promotion** from FS.

**What must the lookup conclude?**

- Local: tokens 0..95 (96, block-aligned).
- Loadable from FS: tokens 96..255 — that is chunks 1..3 (chunk 0, tokens 0..63, is already covered by the local hit).
- Computed fresh: tokens 256..399.

So the ideal answer is **160 externally-loadable tokens**, landing in GPU blocks 6..15, and the request starts decoding at token 256 without ever recomputing tokens 96..255.

A fairness note before we begin: the exact residency history above is a *prop* — chosen to exercise every mechanism at least once (prefix caching, preemption, release-and-store, cascade, CPU LRU eviction, GPU LRU residency). The mechanisms themselves are real and each is cited. What the code actually sees is just two numbers: 100 local tokens from the KV cache manager, and whatever the tiers answer per chunk.

**Figure 1** — the outcome the lookup is working toward:

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Where the 400 prompt tokens of request R come from",
  "data": {
    "values": [
      {"source": "GPU prefix cache (resident blocks)", "tokens": 96},
      {"source": "Promoted from FS tier", "tokens": 160},
      {"source": "Computed fresh during prefill", "tokens": 144}
    ]
  },
  "layer": [
    {
      "mark": "bar",
      "encoding": {
        "x": {"field": "source", "type": "nominal", "title": "Token source", "axis": {"labelAngle": -20}},
        "y": {"field": "tokens", "type": "quantitative", "title": "Prompt tokens"}
      }
    },
    {
      "mark": {"type": "text", "dy": -6},
      "encoding": {
        "x": {"field": "source", "type": "nominal", "axis": {"labelAngle": -20}},
        "y": {"field": "tokens", "type": "quantitative"},
        "text": {"field": "tokens", "type": "quantitative"}
      }
    }
  ]
}
```

256 of the 400 tokens (96 + 160) come from caches — only 144 are actually computed. The whole lookup machinery exists to maximize the left two bars.

---

## 3. The walkthrough — five engine steps, six code layers

This section traces R through the system. Each step names the exact code that runs, quotes the load-bearing snippet, and — the part most writeups skip — explains *why* the code does what it does.

### Step 0 — Arrival: per-request context and lazy keys

When R is added to the scheduler, the connector builds its per-request state (`offloading/scheduler.py:804-814`):

```python
def on_new_request(self, request: Request) -> None:
    req_context = _create_req_context(request)          # parse kv_transfer_params → TierFilter
    offloading_context = self.manager.on_new_request(req_context)   # tiers get a look-in
    req_status = RequestOffloadState(config=self.config, req=request,
                                     req_context=req_context,
                                     offloading_context=offloading_context)
    self._req_status[request.request_id] = req_status
```

Two things happen here that will matter later:

1. `_create_req_context` (`:431`) parses the request's `kv_transfer_params` — this is where a client can restrict which tiers participate in *this request's* lookups, e.g. `{"kv_load_tiers": [{"medium": "STORAGE", "locality": "LOCAL"}]}`, producing a `TierFilter` (`base.py:73-87`) that the tier walk checks per tier. A per-request routing hook — relevant for llm-d cache-aware routing.
2. `RequestOffloadState` is created with **empty** `offload_keys`. The keys are derived lazily at each admission attempt (§1.4) because the prompt hash list is only finalized then. `RequestOffloadState.update_offload_keys` (`:312`) appends any newly-available chunk keys — the `islice` start advances as `offload_keys` grows.

### Step 1 — First admission: the async existence check

#### 1a. The core scheduler asks the local cache first

R is popped from the waiting queue. The scheduler asks the KV cache manager how many tokens are computed locally (`get_computed_blocks`) — the answer is 100. Then, before touching the connector, it rounds that down to a **block boundary**:

```python
# vllm/v1/core/sched/scheduler.py:773-781
partial_tail = num_new_local_computed_tokens % self.block_size   # 100 % 16 = 4
block_aligned_local = num_new_local_computed_tokens - partial_tail  # 96
ext_tokens, load_kv_async = self.connector.get_num_new_matched_tokens(
    request, block_aligned_local
)
```

Why truncate 100 → 96? **Copy-on-write races.** The 100-token hit runs through a *partially filled* block 6 (tokens 96..99). If the request adopted that block from the cache and then wrote token 100 into it, the cache block would need copy-on-write. If the connector also loaded tokens 96..111 into a fresh block, the two writers would race on overlapping memory. The fix: the local hit is truncated to the last *full* block boundary, and the external load is allowed to cover the tail — the scheduler handles the supersede at `:791-801` (see Step 3c). Passing a block-aligned number also means the connector can never return a partial-block answer that conflicts with local state.

#### 1b. The connector's contract: `(num_tokens | None, async_flag)`

```python
# offloading/scheduler.py:816-871 (abridged)
def get_num_new_matched_tokens(self, request, num_computed_tokens):
    req_status = self._req_status[request.request_id]
    for group_state in req_status.group_states:
        group_state.block_ids.clear()          # reset per-step bookkeeping

    if req_status.transfer_jobs:               # in-flight transfers → wait
        return None, False

    req_status.update_offload_keys()
    req_status.num_locally_computed_tokens = num_computed_tokens   # 96
    num_hit_tokens = self._lookup(req_status)
    ...
    return num_hit_tokens, bool(num_hit_tokens)
```

The return contract is the whole game: **`None` means "I cannot decide yet — ask me again next step"** (and the scheduler then refuses to schedule R, §1e); otherwise the first element is the number of loadable tokens *beyond* the local hit, and the second says those tokens will arrive asynchronously, between scheduler steps. Note also the guard: if R already has in-flight transfers (`transfer_jobs` non-empty), it is deferred outright — a request can't start a second transfer while one is running.

#### 1c. `_lookup` — the chunk-granular scan

For our single-group dense model, `_lookup` (`offloading/scheduler.py:631`) reduces to a prefix scan:

```python
start_chunk_idx = num_computed_tokens // tokens_per_chunk   # 96 // 64 = 1
num_chunks = min(cdiv(query_max, tokens_per_chunk), len(offload_keys))  # min(7, 6) = 6
offload_keys = offload_keys[start_chunk_idx:num_chunks]     # k1..k5
num_hit_chunks = self._maximal_prefix_lookup(offload_keys, req_context, ...)
```

`_maximal_prefix_lookup` (`:545-577`) walks the keys and calls `self.manager.lookup(key, req_context)` once per chunk:

```python
case LookupResult.HIT:           hit_count += 1
case LookupResult.HIT_PENDING:   defer_lookup = True; hit_count += 1
case LookupResult.RETRY:         defer_lookup = True    # don't break — keep scanning!
case LookupResult.MISS:          break
```

The "don't break on `RETRY`" line is a deliberate optimization: by scanning *past* an unresolved key, the scheduler seeds async existence checks for as many keys as possible before the first true `MISS`, so the next step has a warm result set (§4a).

#### 1d. The tiered verdict — `TieringOffloadingManager.lookup`

This is the heart of the lookup. One call per chunk, this ordering (code at `tiering/manager.py:282-350`):

```python
self._maybe_process_finished_jobs()      # 1. poll tiers for finished jobs (once/step)
primary_hit = self.primary_tier.lookup(key, req_context)   # 2. CPU primary tier
if primary_hit is LookupResult.HIT:       return LookupResult.HIT
if primary_hit is LookupResult.HIT_PENDING: return LookupResult.HIT_PENDING
lookup_start = time.monotonic()
for tier in self.secondary_tiers:         # 3. walk secondaries in config order
    if tier is exclude_tier: continue
    if not req_context.load_tier_filter.allows(tier.medium, tier.locality): continue
    result = tier.lookup(key, req_context)
    if result is LookupResult.HIT:
        promoted = self._initiate_promotion(tier, key, req_context)
        return LookupResult.MISS if not promoted else LookupResult.RETRY
    if result is LookupResult.RETRY: any_retry = True
return LookupResult.RETRY if any_retry else LookupResult.MISS
```

Three design points:

- **Poll before look up.** The first call in a step drains each tier's finished-jobs queue (`_process_finished_jobs`, `:246`, gated by `_processed_jobs_this_step`). A promotion that completed since the last call must read as `HIT` *now*, not as a stale `MISS` or `HIT_PENDING`; and blocks freed by completed jobs must become evictable in time for a promotion this same call may initiate.
- **The CPU primary tier is a dict.** `cpu/manager.py:113` — key present and `is_ready` → `HIT`; present but not ready (`ref_cnt == -1`, store in flight) → `HIT_PENDING`; absent → `MISS`. That's the entire "fast path" existence check: one hash-map probe. (It also records a usage counter if the connector is configured to count lookups, `:114-121`.)
- **First secondary hit wins, and the walk stops there.** Tiers are consulted in config order; there is no latency-aware selection. The first tier that answers `HIT` triggers a promotion and the loop breaks. Whether ordering tiers by expected latency would measurably help is an open question (see the reference note's open threads).

**Our case:** CPU primary → `MISS` for `k1`. FS tier → `FileSystemTierManager.lookup` (`fs/manager.py:200`) delegates to the async subsystem:

```python
def lookup(self, key, req_context) -> LookupResult:
    result = self._lookup_manager.lookup(key, req_context)   # AsyncLookupManager
    if result is None: return LookupResult.RETRY
    return LookupResult.HIT if result else LookupResult.MISS
```

`AsyncLookupManager.lookup` (`async_lookup.py:125`) is a pure in-memory operation: record the key in this step's batch, return the cached result — which is `None` for a cold key. So every key in our scan returns `RETRY` → `defer_lookup` → `_lookup` returns `None` → `get_num_new_matched_tokens` returns `(None, False)`.

**Why async at all?** The scheduler thread cannot afford a synchronous `stat()` on a network filesystem for every chunk of every request — CephFS metadata operations are ~milliseconds, and the scheduler thread is shared by *all* requests. The async design bounds per-step scheduler latency at the cost of a one-step admission delay for cold keys. That is the trade the whole architecture makes, and it is visible in the module docstring (`async_lookup.py:10-31`).

#### 1e. The park

```python
# vllm/v1/core/sched/scheduler.py:783-789
if ext_tokens is None:
    request_queue.pop_request()
    step_skipped_waiting.prepend_request(request)   # untouched; re-examined next step
    continue
```

R is popped from the queue and **prepended back** — no GPU blocks allocated, nothing about R mutated. `prepend` (not `append`) is the fairness mechanism: R keeps its position ahead of newer arrivals, so repeated deferral cannot starve it behind fresh requests.

#### 1f. End of step — the batch goes to the background thread

`on_schedule_end` (`tiering/manager.py:728`) runs a fixed sequence per step (poll finished jobs, serve P2P requests, reset the poll gate, flush pending promotions, then each tier's own `on_schedule_end`). For the FS tier, `on_schedule_end` → `AsyncLookupManager.flush()` (`async_lookup.py:147`):

```python
self._need_to_drain = True
if self._lookup_batch:
    self._lookup_queue.put(self._lookup_batch)   # ONE item = this step's whole batch
    self._lookup_batch = []
```

The background thread (`_worker`, `:196`) groups the batch by request and calls the tier's `batch_lookup` — for FS, the C extension that **releases the GIL for the entire `faccessat()` batch** (`fs/manager.py:76-83`):

```python
def batch_lookup(self, keys, req_context):
    paths = [self._tier.file_mapper.get_file_name(k) for k in keys]
    if _HAS_BATCH_LOOKUP_C:
        # C extension: GIL released for the entire faccessat() batch.
        return batch_lookup_C(paths)
    return (os.path.exists(p) for p in paths)
```

The GIL detail is not trivia: `os.path.exists` per key would reacquire the GIL per syscall, serializing the whole batch; the C extension holds the GIL released for the *entire* batch of syscalls, which is the difference between a background batch and a scheduler stall on network filesystems. The worker runs during the model-execution window, so this work is free from the scheduler's perspective.

**There is no lock anywhere in this subsystem** — thread safety by ownership, spelled out in the module docstring (`async_lookup.py:10-31`): `_lookup_state`/`_lookup_batch` are scheduler-owned; `_lookup_queue` is scheduler-writes/worker-reads; `_pending_results` is worker-writes/scheduler-reads. Both are `queue.SimpleQueue`, safe for exactly one writer and one reader. The design avoids locks by being strict about *who touches what*, which keeps the scheduler thread on a lock-free path.

### Step 2 — Promotion is staged, still `RETRY`

Next step. The scan runs again, but this time `lookup()` **drains results first** (`drain_results`, `async_lookup.py:160`): the `faccessat` batch came back `k1,k2,k3 → True`, `k4 → False`. (The FS files for k1..k3 exist because R′ stored them, §2.3.)

The tier walk now reaches the interesting branch for `k1`:

```python
# tiering/manager.py:332-341
if result is LookupResult.HIT:
    promoted = self._initiate_promotion(tier, key, req_context)
    return LookupResult.MISS if not promoted else LookupResult.RETRY
```

`_initiate_promotion` (`:380-427`) is a **two-phase trick**: reserve the CPU slot *now*, do the I/O *later*.

**Phase 1 (now, on the scheduler thread):**

```python
primary_write_result = self.primary_tier.prepare_write([key], req_context)
if primary_write_result is None:
    return False          # CPU tier full → treat as MISS (recompute beats spinning)
# ... accumulate (key, block_id) into _pending_load_submissions[tier][req_id]
# NO submit_load yet — that is deferred to on_schedule_end
```

`prepare_write` allocates a CPU block with `ref_cnt = -1` (write in flight, `is_ready == False`). This is load-bearing: **any subsequent lookup of the same key within the same step now sees `HIT_PENDING` from the primary tier**, which suppresses duplicate promotion attempts. And the `None` return path is the deliberate `MISS`-not-`RETRY` choice from §1.6.

So the scan sees `k1,k2,k3 → RETRY` (promotion started), `k4 → MISS` → break. `_lookup` returns `None` again. **Second step of admission delay.** The `RETRY` here is not an error — it is the system saying "work is in progress; re-examine me."

**End of step 2 — the batched I/O flush.** `on_schedule_end` calls `_flush_pending_promotions` (`:429-451`), which issues **one batched `submit_load` per (tier, request)**:

```python
for tier, pending_by_ctx in self._pending_load_submissions.items():
    for entry in pending_by_ctx.values():     # one entry per request on this tier
        job_id = self._next_job_id()
        self._transfer_jobs[job_id] = JobMetadata(job_id=job_id, keys=entry.keys,
            block_ids=..., is_promotion=True, req_context=entry.req_context)
        tier.submit_load(job_metadata)
```

Batching per (tier, request) is what turns "3 chunk lookups" into "1 I/O job": one job record, one completion poll, one `batch_load_block` call. `FileSystemTierManager.submit_load` (`fs/manager.py:221-231`) enqueues on its dual-queue thread pool:

```python
task = functools.partial(
    batch_load_block,
    [self.file_mapper.get_file_name(key) for key in job_metadata.keys],   # k1..k3 → .bin paths
    self._primary_kv_view,          # <- the /dev/shm mmap; read INTO these offsets
    [int(bid) * self._block_size for bid in job_metadata.block_ids],
    self._block_size,
    self._use_o_direct,
)
self._pool.enqueue_load(job_metadata.job_id, 1, [task])
```

**This is the architecture's key trick, visible in one function signature:** the read thread writes directly into `primary_kv_view` — a slice of the `/dev/shm` region that the *worker process* will DMA from in Step 4. The scheduler-side view is created at `tiering/spec.py` and handed to each tier at construction; the worker carves its tensors from the same region. **No KV bytes are ever IPC'd.** The FS thread's `readv` fills exactly the buffer the CUDA copy engine reads later.

Two supporting details worth remembering:

- **The file layout** (`file_mapper.py:107-115`): `<base>_r<rank>/<hash[:3]>/<hash[3:5]>_g<group_idx>/<hash>.bin` — hash-fanned two levels deep to keep directory width bounded. The base path embeds the model name + a config hash so incompatible runs cannot collide.
- **The thread pool** (`fs/thread_pool.py`): read-priority threads (default 16) and write-priority threads (default 16); both groups can drain either queue so loads and stores don't starve each other.

### Step 3 — Promotion lands → real `HIT` → the load job is built

#### 3a. The poll that flips the state

At the top of the next lookup (or in `on_schedule_end`'s catch-all), `_maybe_process_finished_jobs` polls the tier: `get_finished_jobs()` → `_process_finished_jobs` (`:246-279`) dispatches on `JobMetadata.is_promotion`:

```python
if job_metadata.is_promotion:
    # secondary→primary transfer (promotion) completed → make blocks available
    self.primary_tier.complete_write(job_metadata.keys, ..., completed_job.success)
```

`complete_write` flips `ref_cnt` from `-1` to `0` — `is_ready` becomes `True`. Chunks 1..3 are now **ordinary CPU residents**, indistinguishable from blocks that were never offloaded. The poll-before-lookup ordering (§1d point 1) is what makes this read as `HIT` on this very step.

#### 3b. The lookup succeeds — the math

Scan: `k1,k2,k3 → HIT`, `k4 → MISS` → break. `num_hit_chunks = 3`, no deferral. `_lookup` closes the loop (`:736-744`):

$$
\text{max\_hit\_size\_tokens} = \min(400,\ 64 \times (1 + 3)) = 256
$$

$$
\text{num\_hit\_tokens} = 256 - 96 = 160
$$

`get_num_new_matched_tokens` returns **`(160, True)`** — 160 tokens loadable, arriving asynchronously. The deferred-lookup timer started back in Step 1 (`deferred_lookup_start_time`) is now observed into the `LOOKUP_ASYNC_DELAY` histogram — that measurement is the "promotion stall," see §6.

**Figure 2** — how far R has come, measured in the only clock that matters:

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 2 — Tokens of R's prompt accounted for (computed or loadable), by engine step",
  "data": {
    "values": [
      {"step": 1, "tokens": 96, "event": "admission 1: async lookup, RETRY"},
      {"step": 2, "tokens": 96, "event": "admission 2: promotion staged, RETRY"},
      {"step": 3, "tokens": 256, "event": "admission 3: promotion landed, HIT, park"},
      {"step": 4, "tokens": 256, "event": "forward pass: DMA load, prefill starts"},
      {"step": 5, "tokens": 400, "event": "prefill done, decoding"}
    ]
  },
  "mark": {"type": "line", "point": true},
  "encoding": {
    "x": {"field": "step", "type": "quantitative", "title": "Engine step", "axis": {"tickMinStep": 1}},
    "y": {"field": "tokens", "type": "quantitative", "title": "Prompt tokens accounted for"},
    "tooltip": [{"field": "event", "type": "nominal", "title": "What happened"}]
  }
}
```

Two steps of plateau (the RETRY steps) then a jump to 256 once the promotion lands. That plateau *is* the step-quantized admission cost.

#### 3c. GPU blocks allocated; the request is parked for real

Back in the core scheduler with `ext_tokens = 160` and `partial_tail = 4`:

```python
# vllm/v1/core/sched/scheduler.py:791-801
if partial_tail and ext_tokens > partial_tail:
    # remote strictly exceeds the local hit: drop the sub-block tail so no
    # CoW is needed, and let the load cover it
    new_computed_blocks = self.kv_cache_manager.truncate_computed_blocks(
        new_computed_blocks, block_aligned_local)
    num_new_local_computed_tokens = block_aligned_local   # 96
    num_external_computed_tokens = ext_tokens             # 160
```

Because 160 > 4, the local hit is truncated to 96 and the load owns tokens 96..255 entirely — no block is written twice (§1a's race is avoided by construction). Then the request is parked in the explicit waiting-for-KV state:

```python
# vllm/v1/core/sched/scheduler.py:1022-1041
request = request_queue.pop_request()
if load_kv_async:
    request.status = RequestStatus.WAITING_FOR_REMOTE_KVS
    step_skipped_waiting.prepend_request(request)
    request.num_computed_tokens = 256    # set now; unused until the transfer finishes
    self._inflight_prefills.add(request)
```

Note the comment in the code: `num_computed_tokens` is set even though the KV isn't loaded yet; it is not used anywhere until the transfer completes (and if the transfer fails, `_update_requests_with_invalid_blocks` re-sets it and only the successfully-loaded tokens are cached — see the failure handling in `_update_waiting_for_remote_kv`, `scheduler.py:2645-2663`).

#### 3d. `update_state_after_alloc` — build the `TransferJob`

The scheduler hands the connector the allocated blocks so it can translate "tokens" into "chunk→GPU-block copy operations" (`scheduler.py:1001`):

```python
self.connector.update_state_after_alloc(request,
    self.kv_cache_manager.get_blocks(request_id), num_external_computed_tokens)   # 160
```

Inside (`offloading/scheduler.py:873-970`), per group:

- `num_gpu_blocks = cdiv(256, 16) = 16`; `num_locally_computed_gpu_blocks = 6` → `num_pending_gpu_blocks = 10`
- `num_chunks = cdiv(256, 64) = 4`; `start_chunk_idx = 6 // 4 = 1` → `keys_to_load = offload_keys[1:4] = [k1, k2, k3]`
- `dst_block_ids = GPU blocks 6..15`; `group_sizes = [10]`; `block_indices = [6]`

Then the pin and the job record:

```python
src_spec = self.manager.prepare_load(keys_to_load, req_status.req_context)
#   cpu/manager.py:130-146 — per key: assert is_ready; if ref_cnt == 0: mark_non_evictable
#                            ref_cnt += 1   ← pinned against eviction during the DMA
dst_spec = GPULoadStoreSpec(dst_block_ids, group_sizes=group_sizes, block_indices=block_indices)
load_job_id = self._generate_job_id()
self._current_batch_load_jobs[load_job_id] = TransferJob(req_id, src_spec, dst_spec)
req_status.transfer_jobs.add(load_job_id)
self._jobs[load_job_id] = TransferJobStatus(req_id=request.request_id,
    pending_count=self.config.num_workers,   # every TP rank must ack
    keys=set(keys_to_load), is_store=False)
```

Two details that look obscure and matter a lot:

- **`block_indices = [6]`.** The offloaded chunk is 4× a GPU block, and our load starts *mid-chunk*: GPU blocks 0..5 (tokens 0..95) are already computed locally, so chunk 1 (tokens 64..127) is only needed from GPU block 6 onward. `block_indices` tells the worker where in the chunk to start; in Step 4 this becomes "skip the first 2 sub-blocks of the first CPU chunk." Without this, the offload format could not span misaligned request boundaries — which would be most requests.
- **`pending_count = num_workers`.** Under tensor parallelism, every rank must DMA its slice and ack before the job is complete. `update_connector_output` (`:1280`) decrements the count as acks arrive; `manager.complete_load` (`cpu/manager.py:153`) releases the pins when it hits zero.

There is also a subtle invariant enforced by the assert at `:960`: `not req_status.transfer_jobs` — a request can have a load *or* stores in flight, never both. The connector refuses to schedule a load on a request that still has pending store work, because the two directions would race on the same blocks.

### Step 4 — The worker executes the DMA

At the start of the forward pass, the scheduler pushes the job metadata to the workers; each TP rank runs:

```
start_load_kv (offloading_connector.py:103)
→ start_kv_transfers (offloading/worker.py:319)
→ CPUOffloadingWorker.submit_load (cpu/gpu_worker.py:537)
→ transfer_async (cpu/gpu_worker.py:240)
```

`transfer_async` is where the abstract machinery becomes byte movement:

1. **Build pinned descriptor arrays.** `src` pointers, `dst` pointers, and `sizes` are computed with a vectorized numpy expansion over sub-blocks (`compute_sub_block_ptrs`, `:73`). For our job: 10 GPU blocks × KV tensors per block = 10+ copy ops (one per (block, tensor)). The first CPU chunk is entered mid-buffer — `src_logical_blocks_to_skip = block_idx % src_blocks_per_chunk = 6 % 4 = 2` sub-blocks skipped (`:316-317`) — which is exactly the `block_indices = [6]` from Step 3d materializing in the pointers.
2. **Issue ONE batched `swap_blocks_batch`** on a pooled CUDA stream, bracketed by pooled start/end events (`:362-400`). Streams, events, and descriptor buffers are all pooled for reuse; the pool grows if a transfer needs more room.
3. **Ordering within a direction.** Each transfer's stream waits on the previous transfer's end event (`:379-383`), so submission order is preserved.
4. **Access-order semantics differ by direction** (`:384-390`): CPU→GPU (our case) uses `CU_MEMCPY_SRC_ACCESS_ORDER_ANY` — the host pinned source is never concurrently written, so the driver may pipeline source reads; GPU→CPU must keep STREAM ordering because it reads the live GPU KV cache that the compute stream is still writing.

Kernel selection was resolved once at init (`_select_swap_blocks_fn`, `:35-53`): GPU→CPU always uses the C++ DMA path (bandwidth-bound; the copy engine wins); CPU→GPU uses a Triton kernel only for small (< `THRESHOLD_BYTES` = 28 KiB), 8-byte-aligned pages, and falls back to DMA on ROCm/XPU or without Triton. For our 10-block load, the Triton path does not apply (too large); the batched DMA path runs.

#### Completion — the unpark

```python
# cpu/gpu_worker.py:419-429 — non-blocking CUDA event poll
while self._transfers and self._transfers[0].end_event.query():
    transfer = self._transfers.popleft()
    transfer_time = transfer.start_event.elapsed_time(transfer.end_event) * 1e-3
    ...  # transfer_size / transfer_time → the KV-transfer bandwidth metrics
```

`worker.get_finished` (`offloading/worker.py:346-381`) maps the finished job back to `req_id` and emits `finished_recving = {R}`. The core scheduler's `_try_promote_blocked_waiting_request` (`scheduler.py:2678-2693`) then calls `_update_waiting_for_remote_kv` (`:2635`), which caches the loaded blocks into the KV cache manager, flips R's status back to `WAITING`, and R is scheduled normally — decoding from token 256. Meanwhile the scheduler-side `update_connector_output` decrements `pending_count`; when all ranks ack, `manager.complete_load(keys)` (`cpu/manager.py:153-163`) drops `ref_cnt` 1→0 and the blocks become evictable again.

**Figure 3** — the two paths, side by side, in the units that matter:

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 3 — Engine steps elapsed before R runs: cold FS-tier hit vs warm CPU-tier hit",
  "data": {
    "values": [
      {"path": "Cold FS-tier hit", "phase": "Async existence check (RETRY)", "steps": 1},
      {"path": "Cold FS-tier hit", "phase": "Promotion staged, I/O submitted", "steps": 1},
      {"path": "Cold FS-tier hit", "phase": "Tier I/O lands, HIT, park", "steps": 1},
      {"path": "Cold FS-tier hit", "phase": "CPU\u2192GPU DMA, run", "steps": 1},
      {"path": "Warm CPU-tier hit", "phase": "Lookup, pin, park", "steps": 1},
      {"path": "Warm CPU-tier hit", "phase": "CPU\u2192GPU DMA, run", "steps": 1}
    ]
  },
  "mark": "bar",
  "encoding": {
    "x": {"field": "path", "type": "nominal", "title": "Where the blocks are found"},
    "y": {"field": "steps", "type": "quantitative", "title": "Engine steps before R runs"},
    "color": {"field": "phase", "type": "nominal", "title": "Phase"}
  }
}
```

The cold path costs **4 engine steps minimum**; the warm path costs **2** (and one of those overlaps normal scheduling). Everything in the middle of Figure 3 — the "staged promotion" machinery — exists to keep the scheduler thread responsive while paying this step tax.

---

## 4. The fast path — block already in the CPU primary tier

If `k1..k3` had been CPU residents (e.g., a sibling request just used them, so they hadn't been evicted to FS yet), Steps 1 and 2 collapse entirely:

| Stage | What happens | Code |
|---|---|---|
| 1. Lookup at admission | primary `lookup` → `HIT` ×3; no secondary walk | `cpu/manager.py:113` |
| 2. Park | `WAITING_FOR_REMOTE_KVS` | `scheduler.py:1026` |
| 3. Pin + build job | `prepare_load` (ref_cnt 0→1) + `GPULoadStoreSpec` + `TransferJob` | `offloading/scheduler.py:948-967` |
| 4. DMA | same `transfer_async` / `swap_blocks_batch` as Step 4 | `gpu_worker.py:240` |
| 5. Completion | `get_finished` → `finished_recving` | `gpu_worker.py:419`, `worker.py:346` |
| 6. Unpin | `complete_load` ref_cnt 1→0 | `cpu/manager.py:153` |

The **only** difference from the cold path is the absence of the async existence check and the promotion — two full steps of Figure 3's left bar. This is why the CPU tier is worth sizing generously: every block that stays in CPU saves two engine steps and one storage round-trip versus letting it fall to FS.

---

## 5. What changes with hybrid models

The walkthrough used one full-attention group, which reduced `_lookup` to a single prefix scan. Real models complicate it, and the complications are all in `_lookup` (`offloading/scheduler.py:631-802`):

1. **The convergence loop.** The loop iterates full-attention groups first (maximal-prefix scan) and sliding-window groups second (backwards suffix scan, `_sliding_window_lookup` `:579-610`). Each group may tighten `max_hit_size_tokens` — the sliding-window group can only use the last `sliding_window_size` chunks, so a shorter answer there *invalidates* the full-attention group's longer one — and when that happens the loop **re-runs** (`:746-757`) until the answer converges.
2. **Sliding-window scan direction.** `_sliding_window_lookup` scans from the *end* of the key list, looking for the last run of `sliding_window_size` consecutive hits — the window slides, so the *suffix* is what matters, not the prefix. `HIT_PENDING` counts toward the streak (it will be readable) but still defers; `RETRY` resets the streak to 0 but keeps scanning.
3. **The last prompt token is recomputed for SWA.** With any sliding-window group, `max_hit_size_tokens` is decremented by 1 (`:642-646`) because the final prompt token must be recomputed to produce logprobs — the hit window is capped one token short of the prompt.
4. **EAGLE/MTP draft groups.** The trailing chunk of a draft group has no stable hash — its draft-layer KV can be rewritten after spec-token rejection — so it is excluded from both store and load scheduling. The current code handles this with an explicit `eagle_verified` set (`:657-660`, `:732-734`): an extra chunk is queried and then popped once verified, and the pop is re-done whenever a non-EAGLE group tightens the boundary.
5. **SWA store-skip.** `is_store_reachable_swa_chunk` (`:122-139`) skips storing SWA chunks that could never serve a load hit (e.g., DeepSeek V4's smaller SWA blocks inside a full-attention alignment segment). The offloaded byte volume is therefore *not* proportional to token count on such models.

None of this changes the tier machinery — lookup, promotion, load are group-agnostic. It only changes what `_lookup` asks, and in what order.

---

## 6. The cost model

Let $s$ = engine steps. A cold secondary-tier hit costs at minimum:

$$
s_{\text{total}} \;\ge\; \underbrace{1}_{\text{async lookup returns RETRY}} \;+\; \underbrace{1}_{\text{promotion submitted at end of step}} \;+\; \underbrace{k}_{\text{tier I/O latency}} \;+\; \underbrace{1}_{\text{CPU}\to\text{GPU load}}
$$

where $k \ge 1$ is how many steps the tier's I/O takes relative to step duration. In our example $k = 1$ (the read completes within one step because it runs during the model-execution window), giving $s_{\text{total}} = 4$. A CPU-tier hit skips the first three terms entirely ($s_{\text{total}} = 2$).

The practical consequence, which is the reference note's most important observation and bears repeating: **this step-quantized latency — not raw device bandwidth — dominates TTFT for secondary tiers.** A fast NVMe device shrinks $k$ (fewer steps of I/O) and the per-byte cost inside Step 4, but it cannot remove the two structural steps (async lookup, promotion staging). That is why device-level benchmarks (NVMe IOPS, CephFS throughput) so often fail to explain end-to-end TTFT numbers on their own.

The metric that isolates this gap: `vllm:kv_offload_tiering_lookup_async_delay_seconds` — the wall-clock from a request's first *deferred* secondary lookup to allocation/finish, i.e., the promotion stall (`tiering/base.py:41`, observed from `tiering/manager.py:369-378`). Its sibling `vllm:kv_offload_tiering_lookup_sync_delay_seconds` (`tiering/base.py:40`) measures the blocking time the scheduler thread *did* spend on secondary queries. If the async-delay histogram is flat in steps (multiples of the step duration), the system is behaving as designed; if the sync-delay histogram is non-trivial, something is blocking the scheduler thread that shouldn't be.

---

## 7. Observability — watching this happen on a real engine

**Log lines** (scheduler debug level):

- `"Offloading manager delayed request {id} as backend requested"` — a `RETRY` deferral (`offloading/scheduler.py:762`)
- `"Request {id} hit {N} offloaded tokens after {M} GPU hit tokens"` — a successful lookup (`:795`)
- `"Delaying request {id} since some of its chunks are already being loaded"` — duplicate-load suppression (`:788`)
- `"Delaying request {id} since it still has in-flight transfers"` — the one-transfer-at-a-time invariant (`:845`)
- `"O_DIRECT is not supported at '{path}'; falling back to buffered I/O"` — the O_DIRECT probe (`fs/manager.py:179-181`)

**Metrics:**

| Metric | What it measures |
|---|---|
| `vllm:kv_offload_tiering_lookup_sync_delay_seconds` | Blocking scheduler time in secondary-tier lookups |
| `vllm:kv_offload_tiering_lookup_async_delay_seconds` | The promotion stall (the one to watch) |
| `vllm:kv_offload_lookup_sync_delay_seconds` / `..._async_delay_seconds` | Same, connector-level (`offloading/metrics.py:36-37`) |
| Load/store bytes + time histograms | From CUDA event timings (`gpu_worker.py:419-429`) |
| CPU cache usage / read-usage / write-usage gauges | `cpu/manager.py:293-327` |
| `BlockStored` / `BlockRemoved` KV events | Per-tier; see [[vLLM KV Events canonical form]] |

**Filesystem:** the `.bin` files under `<root>_r0/<hash[:3]>/` appear as stores complete. Two checks that have burned people: (1) `PYTHONHASHSEED` must be fixed across instances sharing a root dir, or identical content produces different filenames and a shared PVC yields 0% cross-instance hits that *look like* a working-but-cold cache (`fs/manager.py:97-105`); (2) grep the startup log for the O_DIRECT fallback warning — a silent switch to buffered I/O changes the page-cache story completely and can flatter or distort benchmark results.

---

## 8. Gotchas

1. **`RETRY` is not an error.** It means "promotion started" or "async lookup not yet resolved." Treating it as failure misreads the whole state machine.
2. **`HIT_PENDING` on the CPU tier means a store is in flight**, not a promotion. The two are easy to confuse because both mean "not readable yet," but they answer different questions about the block's lifecycle.
3. **A full CPU tier silently degrades secondary tiers to useless.** `_initiate_promotion` returns `MISS` when `prepare_write` fails, so an undersized `cpu_bytes_to_use` makes an FS/NVMe tier look like a cache miss rather than a capacity problem. Cross-check the CPU usage gauge before blaming storage.
4. **Secondary-tier I/O consumes scheduler-process CPU.** The threads live in the scheduler process; thread counts and I/O stalls compete with scheduling work, not with model execution.
5. **Without a fixed `PYTHONHASHSEED`, cross-instance FS sharing silently yields 0% hits.** Identical tokens, different filenames.
6. **O_DIRECT can silently fall back to buffered I/O.** Grep the startup log.
7. **Trailing partial chunks have no offload key** (§1.4) — the last `tokens_per_chunk % prompt_len` tokens of a prompt are never looked up and always computed. On short prompts this can be a surprisingly large fraction of the prompt.
8. **EAGLE/MTP draft groups exclude their trailing chunk** from store and load — its draft-layer KV has no stable hash.
9. **A request with in-flight transfers is deferred outright** (`offloading/scheduler.py:842-847`) — don't expect a request to both offload and load in the same step.

---

## 9. Code map

| Concern | File |
|---|---|
| Key packing, `LookupResult`, tier filter | `vllm/v1/kv_offload/base.py` |
| Scheduler-side connector: `_lookup`, job building | `.../offloading/scheduler.py` |
| Worker-side connector: submit/poll | `.../offloading/worker.py` |
| Connector glue (`start_load_kv`, `get_finished`) | `.../v1/offloading_connector.py` |
| Multi-tier orchestration, promotion | `vllm/v1/kv_offload/tiering/manager.py` |
| Async existence checks | `vllm/v1/kv_offload/tiering/async_lookup.py` |
| CPU primary tier (pool, `ref_cnt`, eviction) | `vllm/v1/kv_offload/cpu/manager.py` |
| `BlockStatus`, eviction policies | `vllm/v1/kv_offload/cpu/policies/` |
| GPU↔CPU DMA | `vllm/v1/kv_offload/cpu/gpu_worker.py` |
| Shared `/dev/shm` region | `vllm/v1/kv_offload/cpu/shared_offload_region.py` |
| FS tier + I/O + thread pool | `tiering/fs/manager.py` · `tiering/fs/io.py` · `tiering/fs/thread_pool.py` |
| FS file layout | `vllm/v1/kv_offload/file_mapper.py` |
| Tier registry (`example`/`fs`/`p2p`/`obj`) | `vllm/v1/kv_offload/tiering/factory.py` |
| Core scheduler integration | `vllm/v1/core/sched/scheduler.py` |

### Tests that exercise these state machines

- `tests/v1/kv_offload/tiering/test_tiering_offloading.py` — promotion/cascade sequencing
- `tests/v1/kv_offload/tiering/test_async_lookup.py` — lookup queueing
- `tests/v1/kv_offload/tiering/test_fs_tier.py` — FS tier
- `tests/v1/kv_connector/unit/test_offloading_connector.py` — connector-level

---

## 10. Suggested reading order

1. This article, in order — it is the tutorial.
2. [[vLLM KV offload retrieval path - lookup, promotion, and load]] — the reference map with every invariant and the full cost model.
3. [[vLLM KV block prefetch architecture]] — how this compares to LMCache and weight prefetch.
4. [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]] — how the tiers get configured in the first place.

## Provenance

Direct source reading of `vllm-project/vllm` at `0601850791155003afbe5a0d5d086350cada8deb` (local checkout, verified present on `upstream/main`, 2026-08-02), 2026-08-08. All line anchors verified against that commit. The scenario is pedagogical (see the fairness note in §2.3) but every mechanism it exercises is real and cited. Companion to [[vLLM KV offload retrieval path - lookup, promotion, and load]]; where the two differ in detail, this article's line anchors (pinned to `0601850`) take precedence.
