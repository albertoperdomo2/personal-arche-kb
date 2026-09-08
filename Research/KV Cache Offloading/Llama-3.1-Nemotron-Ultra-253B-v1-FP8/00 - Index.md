---
title: "Llama 3.1 Nemotron Ultra 253B v1 FP8 — KV Cache Offloading"
date: "2026-07-31"
type: "research-index"
topic: "KV Cache Offloading"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
status: "active"
report_count: 4
---

# Llama 3.1 Nemotron Ultra 253B v1 FP8

## Reports

- [[2026-09-07 - Selective-load deployment calibration|2026-09-07 — Selective-load deployment calibration]] — Conditionally valid 160-pair CPU-load-versus-recompute calibration on the active Diadochos TP8 H100 deployment. Recommends 1,024 external reusable tokens for the balanced initial policy or 1,536 for a tail-conservative policy; no pending-byte EWMA veto was calibrated because the pressure pilots did not preserve CPU residency.

- [[2026-09-07 - Selective-load binary opt-out first AgentX run - diagnostic invalidation|2026-09-07 — Selective-load binary opt-out first AgentX run — diagnostic invalidation]] — Mechanism-valid but calibration-invalid C64 run. Binary load opt-out worked, but the static policy disabled every load while preserving all stores; the engine saturated and only 38 profiling requests completed. Records which telemetry is trustworthy and defines the same-image A/B matrix required to isolate instrumentation and store-path cost.
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2|2026-07-31 — FP8 Nemotron 253B KV-cache offload comparison — Revision 2]] — Four-cell AgentX comparison at TP8, U0.80, and concurrency 32. Offload is decisively active: CPU256 reaches 3.31× baseline request throughput, NVMe 5.74×, and CephFS 5.78×. NVMe and CephFS are at parity; CephFS is conditionally accepted because two requests disconnected and the cluster was HEALTH_WARN. The next clean center is TP8/U0.82/C32.
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison|2026-07-30 — FP8 Nemotron 253B KV-cache offload comparison]] — Four-cell AgentX comparison at TP8, U0.90, and concurrency 16. The tiered cells are at performance parity because the 2.31-million-token HBM shelf peaks at only ~50%. Revision 2 supersedes its U0.68-at-C16 next-step recommendation for the new C32 regime; the original report remains the canonical underfilled-regime record.

## Workload and methodology

- [[../AgentX Workload Definition|AgentX MVP workload definition]]
- [[../Experiment Methodology|Standardized experiment methodology]]
- [[../02 - Per-Model Methodology Template|Per-model methodology template]]