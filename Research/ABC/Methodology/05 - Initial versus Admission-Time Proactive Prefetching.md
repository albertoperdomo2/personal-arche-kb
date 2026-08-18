---
title: "Initial versus Admission-Time Proactive Prefetching"
date: "2026-08-18"
type: "design-explanation"
experiment: "ABC"
status: "current"
initial_design: "post-miss same-request read-ahead"
current_design: "queued-request oracle admission prefetch"
repository: "/Users/aperdomo/workspace/redhat/vllm"
---

# Initial versus Admission-Time Proactive Prefetching

The current implementation is genuinely proactive: it starts loading KV blocks from NVMe into CPU when a request enters the scheduler, while that request is still waiting. The original approach started only after demand lookup had already encountered a miss.

## Current request flow

```text
Request admitted
      │
      ├─ Server configured with admission_prefetch_chunks=N?
      ├─ Request contains abc_admission_prefetch is exactly true?
      │
      ▼
Derive the request’s ordered KV block keys
      │
      ▼
Select the first N complete prompt chunks
      │
      ▼
Check CPU only
  ├─ HIT / HIT_PENDING → redundant
  └─ MISS → reserve CPU slot
                 │
                 ▼
       Batch all selected blocks
                 │
                 ▼
      One asynchronous NVMe→CPU load
                 │
         request remains queued
                 ▼
Normal demand lookup occurs later
  ├─ CPU HIT         → useful
  ├─ CPU HIT_PENDING → late, check again later
  └─ CPU MISS        → wasted/failed; use reactive fallback
```

The admission trigger lives in:

```text
vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py
OffloadingConnectorScheduler._maybe_prefetch_on_admission()
```

It requires both:

- `admission_prefetch_chunks > 0` in the server configuration.
- `kv_transfer_params.abc_admission_prefetch is True` on that specific HTTP request.

The strict Boolean check prevents values such as `1` or `"true"` from accidentally enabling storage operations.

## Candidate selection

The scheduler already has the request’s prefix-chained block hashes. It converts those into ordered offload keys and takes:

```python
keys = request_offload_keys[:N]
```

So, yes, it still needs block keys because NVMe files must be identified somehow. But it does not:

- search for missing CPU blocks and then decide what to prefetch;
- query NVMe membership;
- scan beyond a demand miss;
- predict a different request.

It simply says:

> For this marked request, assume its first N keys exist on NVMe and try to bring them into CPU now.

That logic calls `prefetch_assume_resident()` in:

```text
vllm/v1/kv_offload/tiering/manager.py
TieringOffloadingManager.prefetch_assume_resident()
```

## What “assume resident” means

For each selected block, the manager checks only CPU:

- CPU `HIT`: already available, so it is redundant.
- CPU `HIT_PENDING`: already being loaded, also redundant.
- CPU `MISS`: reserve a CPU slot and queue an NVMe load.
- CPU capacity unavailable or tier filtered: skipped.

The deliberate omission is:

```text
No secondary_tier.lookup(key)
```

The benchmark guarantees NVMe residency by first populating NVMe with the same deterministic prompts. This is the “oracle” part of the proof of concept.

In a production heuristic, blindly assuming residency would be risky. Here it isolates the question we actually wanted to test:

> If we know which NVMe blocks will soon be requested, is starting their transfer early valuable?

## Batching

Reserving a CPU slot does not immediately launch one I/O operation per block. Promotions are accumulated by request and tier.

At scheduler-step end, `_flush_pending_promotions()` submits one batched filesystem load:

```text
100 selected blocks → one submit_load() job
```

The filesystem worker threads then copy those blocks from NVMe into CPU asynchronously.

This is **NVMe→CPU prefetch**. It is not CPU→GPU prefetch. CPU→GPU loading still occurs through the normal demand path when the request is scheduled.

## When prefetch becomes useful

Later, ordinary demand lookup checks CPU. The tracking code classifies the result:

- `HIT` → `useful`: the proactive CPU copy was consumed.
- First `HIT_PENDING` → `late`: demand arrived before the copy finished.
- Later `HIT` after `HIT_PENDING` → still `useful`; late and useful can both describe the same block.
- `MISS` → `wasted`: the proactive copy disappeared before demand.
- Failed asynchronous load → `load_failed`, not wasted.

“Useful” means the prefetched CPU copy was eventually consumed. It does not by itself prove latency improved. A useful-but-late block may still stall the request.

## Why the original approach failed

The original algorithm ran inside demand prefix lookup:

```text
Demand lookup:
    block 0 → HIT
    block 1 → HIT
    block 2 → MISS
                    │
                    ▼
          Try to prefetch blocks 3...N
```

That looked reasonable as conventional read-ahead, but it conflicts with vLLM’s prefix-chained hashes.

Conceptually:

```text
K1 = H(K0, tokens1)
K2 = H(K1, tokens2)
K3 = H(K2, tokens3)
```

The filesystem tier stores these prefix blocks append-style. Therefore, under normal storage behavior:

```text
K3 exists → K2 exists → K1 exists
```

Consequently, if demand lookup has resolved `K2` as absent from NVMe, then later continuation keys such as `K3`, `K4`, and `K5` normally cannot exist either.

The old prefetcher therefore selected blocks precisely after the point where storage had demonstrated that the continuation was unavailable. That is why the NVMe experiment produced:

```text
attempted > 0
promoted = 0
skipped = attempted
```

It also began too late: demand lookup was already on the request’s TTFT critical path. There was little or no queue time left to hide the NVMe operation.

## The essential difference

| Property | Original approach | Current approach |
|---|---|---|
| Trigger | After a terminal demand miss | At request admission |
| Candidate blocks | N blocks after the miss | First N complete request blocks |
| NVMe membership | Queried candidate by candidate | Assumed by benchmark construction |
| Opportunity to overlap | Minimal; request already executing | Queue time before request executes |
| Candidate validity | Structurally poor | Guaranteed by deterministic replay |
| Data movement | NVMe→CPU | NVMe→CPU |
| Correctness | Reactive fallback | Same reactive fallback |
| Purpose | Read-ahead policy | Controlled proof of proactive timing |

## What the repaired-image validation demonstrated

The repaired-image N=100 treatment demonstrates the difference concretely:

- 25,600 selected;
- 25,344 actually promoted;
- all 25,344 eventually consumed;
- 92.97% ready before first demand;
- 7.03% initially late;
- zero load failures.

See [[Reports/2026-08-18 - Phase 1 admission prefetch repaired-image validation|the repaired-image validation report]] for the full evidence.

## Conclusion

The current implementation proves that admission-time NVMe→CPU prefetch can work. What it does not yet prove is that `N=100` is optimal or that this naive oracle policy consistently improves performance. That is the next experimental step.

The original approach was post-miss read-ahead using a structurally invalid continuation candidate set. The current approach deliberately moves candidate selection to request admission, assumes benchmark-controlled secondary residency, and overlaps NVMe→CPU movement with queue time while preserving the unchanged reactive correctness fallback.