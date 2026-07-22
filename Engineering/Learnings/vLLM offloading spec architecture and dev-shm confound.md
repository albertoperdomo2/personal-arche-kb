# vLLM offloading spec architecture: CPUOffloadingSpec vs TieringOffloadingSpec

## Summary

vLLM has two KV-cache offloading implementations that differ fundamentally in memory model and /dev/shm usage. Mixing them in an experiment matrix without controlling for the spec change introduces a hidden confound.

## CPUOffloadingSpec (older)

- Each tensor-parallel worker independently allocates its own CPU-side KV buffer via **regular heap memory** (anonymous mmap, PyTorch host allocation).
- Does **not** use /dev/shm at all. The default 1 GiB Kubernetes volume is irrelevant.
- Cross-worker block visibility: **none** — each worker maintains its own isolated LRU.
- No secondary tier support.
- Report evidence: the CPU-only run "initializes CPUOffloadingSpec independently in all four TP workers" (2026-07-22 report, line 39).

## TieringOffloadingSpec (newer)

- Creates a **single shared mmap file on /dev/shm** (tmpfs, RAM-backed). All TP workers map the same file, giving shared access to the same physical pages.
- Requires /dev/shm sized to accommodate the mmap plus metadata/page-table overhead. A 68.69 GiB mmap needs ~200 GiB /dev/shm for headroom.
- Cross-worker block visibility: **full** — when worker 0 evicts a KV block and worker 3 later needs it, they access the same physical memory location through the shared mapping.
- Supports multiple storage tiers (CPU primary + NVMe/CephFS secondary).
- Uses a single shared LRU across all workers, not per-worker isolated LRUs.

## The /dev/shm confound in experiment design

When comparing a CPUOffloadingSpec cell to a TieringOffloadingSpec cell, **three variables change simultaneously**: (1) the spec implementation, (2) the /dev/shm size, and (3) a secondary storage tier may be added. A performance difference could come from any combination. Without a control cell using TieringOffloadingSpec with 200 GiB /dev/shm but **no secondary tier**, you cannot isolate whether the gain comes from the actual NVMe device, the tiering implementation, or the larger shared mmap buffer.

## How to isolate the NVMe effect

The correct control is: TieringOffloadingSpec, same CPU allowance (e.g., 64 GiB), 200 GiB /dev/shm, and no secondary tier. This holds the spec implementation and mmap size constant while removing only the storage medium.

## Source

Derived from analysis of the 2026-07-22 Nemotron 3 Super 120B tier matrix report and Q&A on the KV-cache offloading architecture (session ses_074805ea3ffeVkSAg7HyfBNsD2).
