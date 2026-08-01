# Learnings

TILs, patterns, and gotchas that are worth keeping but aren't tied to a single incident — tooling tricks, mental models, config that bit you, benchmarking notes.

One file per learning keeps them searchable. Name them descriptively (e.g. `vLLM paged-attention memory math.md`).

## Learnings
- [[Arche Vega-Lite renderer guardrails]] — why valid charts can fail Arche's pre-render safety/feature guard, plus safe condition and composition workarounds.
- [[CephFS performance tuning for KV cache offloading]] — shareable, end-to-end CephFS tuning guide for vLLM KV-cache workloads, including OpenShift commands, manifests, validation, performance results, and rollback.
- [[CephFS tuning guide - concepts and rationale]] — concise, implementation-oriented guide to the Ceph/Rook, 200G networking, multi-NIC OSD, StorageClass, PVC, and vLLM settings used for KV-cache offloading, including verification and rollback.
- [[Routing Ceph over 200G NICs on OCP with PCI passthrough VFs]] — full investigation of 4 networking approaches (3 failed, 1 succeeded), big-bang MON migration procedure, 6 hidden gotchas, and monmap repair for cluster recovery. 1.3 GB/s write, 1.7 GB/s read verified on 200G fabric.
- [[Forge extends only applies to presets]] — Forge `extends` merges preset-level fields, not top-level config.
- [[vLLM offloading spec architecture and dev-shm confound]] — how vLLM's offloading spec determines /dev/shm sizing and the confound with shared_memory_size.
- [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]] — the two offloading spec types, when to use each, and how tier configuration works.
- [[vLLM KV Events canonical form]] — canonical reference for vLLM's KV cache event types (BlockStored, BlockRemoved, AllBlocksCleared), their fields, wire format, key source files, and schema evolution notes.
- [[vLLM KV block prefetch architecture]] — the three distinct KV block prefetch mechanisms in vLLM (TieringOffloadingManager, LMCache MP Connector, Weight Prefetch), their tiers, state machines, and async design principles.
- [[vLLM KV offload retrieval path - lookup, promotion, and load]] — deep dive on how offloaded KV blocks get back to GPU from CPU/FS/NVMe/obj/P2P tiers: admission-time lookup, staged promotion, async batched existence checks, ref_cnt pinning, and the step-quantized latency cost. Code permalinks pinned to `vllm@4ee9702`.