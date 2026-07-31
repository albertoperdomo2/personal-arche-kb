---
title: "Llama 3.1 Nemotron Ultra 253B v1 FP8 — KV Cache Offloading"
date: "2026-07-31"
type: "research-index"
topic: "KV Cache Offloading"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
status: "active"
report_count: 2
---

# Llama 3.1 Nemotron Ultra 253B v1 FP8

## Reports

- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2|2026-07-31 — FP8 Nemotron 253B KV-cache offload comparison — Revision 2]] — Four-cell AgentX comparison at TP8, U0.80, and concurrency 32. Offload is decisively active: CPU256 reaches 3.31× baseline request throughput, NVMe 5.74×, and CephFS 5.78×. NVMe and CephFS are at parity; CephFS is conditionally accepted because two requests disconnected and the cluster was HEALTH_WARN. The next clean center is TP8/U0.82/C32.
- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison|2026-07-30 — FP8 Nemotron 253B KV-cache offload comparison]] — Four-cell AgentX comparison at TP8, U0.90, and concurrency 16. The tiered cells are at performance parity because the 2.31-million-token HBM shelf peaks at only ~50%. Revision 2 supersedes its U0.68-at-C16 next-step recommendation for the new C32 regime; the original report remains the canonical underfilled-regime record.

## Workload and methodology

- [[../AgentX Workload Definition|AgentX MVP workload definition]]
- [[../Experiment Methodology|Standardized experiment methodology]]
- [[../02 - Per-Model Methodology Template|Per-model methodology template]]