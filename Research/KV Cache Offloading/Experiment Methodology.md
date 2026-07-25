---
title: Standardized KV Cache Offloading experiment methodology
date: 2026-07-25
type: process
topic: KV Cache Offloading
---

# KV Cache Offloading — Experiment Methodology

## Purpose

This document defines the standardized methodology for KV-cache offloading tier-comparison experiments. It ensures reproducibility across models, batches, and researchers.

## Four-cell matrix

Every standardized comparison uses exactly four cells:

| Cell | Configuration |
|---|---|
| **HBM-only** | No offload connector; baseline control |
| **CPU 64 GiB** | `TieringOffloadingSpec`, 64 GiB CPU tier, no secondary tier |
| **CPU 64 GiB + NVMe** | `TieringOffloadingSpec`, 64 GiB CPU tier, hostPath NVMe secondary |
| **CPU 64 GiB + CephFS** | `TieringOffloadingSpec`, 64 GiB CPU tier, CephFS PVC secondary |

All cells use the same `TieringOffloadingSpec` connector. CPU64 has no secondary tier, so the comparison isolates secondary-tier behavior.

## Controlled variables (must be identical across all four cells)

| Parameter | Standard value | Notes |
|---|---|---|
| Connector | `TieringOffloadingSpec` | Not `CPUOffloadingSpec` |
| CPU tier size | 64 GiB (`68719476736` bytes) | Matches across all cells |
| `/dev/shm` | 200 GiB | Must be identical; earlier runs used 128 GiB for CephFS — this is a confound |
| Workload | AgentX MVP (WEKA trace) | Same trace repository for all cells |
| Seed | 42 | Controls initial trajectory selection |
| Concurrency | 32 | Do not sweep until offload path is validated |
| Duration | 1,800 seconds | Fixed send window |
| Context length | 131,072 tokens (`max-model-len`) | Same for all cells |
| Prefix caching | enabled | — |
| Streaming | enabled | — |
| GPU memory utilization | model-specific (0.55, 0.64, or 0.85) | Fixed per model, same across all cells |
| Replicas | 1 | One model replica per cell |
| Image digest | same across all cells | Record the full `sha256:` digest |
| Node class | same GPU node class | Pin to specific nodes for cell-level repeatability |

## Variables that differ between cells (intentionally)

| Variable | HBM-only | CPU64 | CPU64+NVMe | CPU64+CephFS |
|---|---|---|---|---|
| Connector | `TieringOffloadingSpec` | `TieringOffloadingSpec` | `TieringOffloadingSpec` | `TieringOffloadingSpec` |
| CPU tier bytes | 64 GiB | 64 GiB | 64 GiB | 64 GiB |
| Secondary tier | none | none | hostPath NVMe | CephFS PVC |
| `PYTHONHASHSEED` | unset | unset | unset | 0 (historical; normalize to same value) |

## Deployment checklist

Before each batch:

1. Verify all four cells use the same image digest, TP, node class, and `gpu-memory-utilization`.
2. Verify `/dev/shm` is 200 GiB for every cell (including HBM-only).
3. Verify `TieringOffloadingSpec` is used for all four cells (not `CPUOffloadingSpec`).
4. For NVMe: use a run-scoped empty directory or explicit pre-run deletion; record clean-state proof.
5. For CephFS: provision a newly named PVC per run; record empty-state proof.
6. Verify `PYTHONHASHSEED` is consistent across all cells.
7. Confirm the AgentX/WEKA trace repository and seed are identical.

## Measurement requirements

### Minimum telemetry (must be present for a cell to be accepted)

- AIPerf profile export (request throughput, output tokens/s, TTFT, E2E, ITL distributions)
- Native vLLM prompt-token-source counters (external KV, local GPU hit, local compute)
- Native vLLM KV-offload byte counters (GPU→CPU store, CPU→GPU load)
- Native vLLM `kv_cache_usage_perc` gauge
- Native vLLM `num_requests_running` and `num_requests_waiting` gauges
- CAdvisor working-set and CPU utilization for the model container
- Complete model server log (not a truncated tail)

### Desired telemetry (strengthen conclusions)

- Node-exporter NVMe byte rates and operation rates
- NVMe device busy fraction
- Ceph pool bytes/s, IOPS, MDS client operations, OSD latency
- kubelet PVC used bytes
- GPU SM utilization, VRAM, PCIe RX/TX
- Per-tier hit/miss/latency/queue-depth counters (not yet available in vLLM)

## Acceptance gates

### Cell-level acceptance

A cell is accepted when:

1. Zero model pod restarts, zero CUDA OOMs, zero server tracebacks
2. For tiered cells: zero `cannot store blocks` warnings (or documented and explained)
3. Profiling phase completes without boundary-cancellation dominance
4. Prompt-token-source counters are non-zero and consistent

### Secondary-tier acceptance (NVMe or CephFS)

A secondary tier is accepted when ALL of:

1. Zero `cannot store blocks` warnings
2. No sustained filesystem store queue
3. Stable external-token share over time (not decaying)
4. No p99 tail explosion (matched-request paired analysis)
5. Direct secondary read/write counters present and non-zero
6. Repeatable improvement over capacity-matched CPU64 control
7. At least three paired repetitions per cell

### CephFS-specific additional gates

- Ceph pool/MDS/OSD Prometheus queries return non-zero series
- PVC usage shows growth (proves allocation)
- Model log retains complete startup and profiling window (not truncated)
- `benchflow-runner` service account has `pods/log` permission in `rook-ceph`

## Repetition requirements

- **Minimum**: 3 paired repetitions per cell (same batch, same node class)
- **Preferred**: 5 repetitions with different seeds for variance estimation
- Report intervals (mean ± CI), not single-run point estimates
- Pair matched requests by AgentX `conversation_id`, turn, and depth for tail analysis

## What NOT to change (until offload path is validated)

- Do not lower `gpu-memory-utilization` below the model's standard value
- Do not increase concurrency beyond 32 until store-refusal issues are resolved
- Do not use `CPUOffloadingSpec` in matrix comparisons (use `TieringOffloadingSpec` for all cells)
- Do not treat single-run results as publication-grade

## Run documentation requirements

Each run must record:

1. MLflow run ID and link
2. Full image digest (`sha256:...`)
3. Node name(s) used
4. Deployment manifest differences between cells
5. PVC name and provisioning state (for CephFS)
6. NVMe path and clean-state proof
7. Boundary cancellation count and profiling grace-timeouts
8. Store-refusal warning count and affected request IDs
9. Disposition decision (accept / reject / directionally usable)

## Related

- [[Cross-Model Synthesis|Cross-model KV Cache Offloading research synthesis]]
- [[00 - Index|KV Cache Offloading index]]
