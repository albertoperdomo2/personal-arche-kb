# vLLM offloading specs: CPUOffloadingSpec vs TieringOffloadingSpec

vLLM has two distinct KV cache offloading implementations that differ fundamentally in how they manage CPU-side memory, coordinate across tensor-parallel workers, and use shared memory.

## CPUOffloadingSpec (older)

- Allocates CPU-side KV cache buffers through **per-worker heap/anonymous mmap** — each TP worker independently manages its own isolated CPU buffer.
- Does **not** use `/dev/shm` at all. The default 1 GiB `/dev/shm` in a Kubernetes pod spec is irrelevant to this implementation.
- No cross-worker block visibility: blocks evicted by worker 0 are not directly accessible by worker 3.
- Each worker runs its own independent LRU eviction policy.
- The report confirms this: "The CPU-only run instead initializes `CPUOffloadingSpec` independently in all four TP workers" (Nemotron report, line 39).
- CPU usage is lower than tiered runs (~532.8% vs ~724–780%) because there is no shared-mmap coordination overhead.

## TieringOffloadingSpec (newer)

- Creates a **single shared mmap file on `/dev/shm`** (tmpfs = RAM-backed filesystem) that all TP workers map into their address space.
- `/dev/shm` must be large enough to back the mmap: a 68.69 GiB shared mmap requires 128–200 GiB `/dev/shm` to account for metadata, page tables, and kernel accounting overhead.
- Full cross-worker block visibility: all workers share the same physical pages, so a block evicted by worker 0 is immediately accessible to worker 3 through the shared mapping.
- Single shared LRU across all workers.
- Supports secondary storage tiers (NVMe hostPath, CephFS PVC) that act as deeper overflow beyond the mmap buffer.
- The shared mmap is the "primary CPU tier"; secondary tiers extend capacity beyond RAM.

## Why they differ architecturally

The tiered spec was designed for **cross-worker block coordination**. When tensor-parallel workers share a model, they may need the same KV cache blocks. The shared mmap ensures block eviction and retrieval is globally consistent without per-worker duplication or explicit IPC. The original spec was simpler and sufficient when each worker operated independently.

## Practical implication for experiments

Comparing `CPUOffloadingSpec` with 1 GiB `/dev/shm` against `TieringOffloadingSpec` with 200 GiB `/dev/shm` changes **three things simultaneously**: spec implementation, shared-mmap size, and secondary tier presence. Any observed performance delta cannot be attributed to the storage medium alone. The proper control is `TieringOffloadingSpec` with the same `/dev/shm` size but **no secondary tier**.

## Source

- Session ses_074805ea3ffeVkSAg7HyfBNsD2 (2026-07-22): Q&A about `/dev/shm` and spec differences.
- Nemotron 3 Super 120B tier matrix report, lines 32–39, 91, 240–248.
- Qwen U=0.55 tier matrix report, lines 21–23, 28, 220, 228.
