---
title: Qwen3.6-35B-A3B KV-cache offloading
date: 2026-07-25
type: research-model-index
model: Qwen3.6-35B-A3B
topic: KV Cache Offloading
---

# Qwen3.6-35B-A3B

## Reports

- [[2026-07-25 - Standardized offload matrix batch 3|2026-07-25 — Standardized offload matrix batch 3]]
- [[2026-07-25 - Standardized offload matrix pressure run|2026-07-25 — Standardized offload matrix pressure run]]
- [[2026-07-23 - TieringOffloadingSpec matrix|2026-07-23 — Clean TieringOffloadingSpec matrix]]
- [[2026-07-23 - TieringOffloadingSpec matrix Summary|2026-07-23 — Matrix summary]]
- [[2026-07-24 - Five-run CephFS tuning matrix|2026-07-24 — Five-run comparison with tuned CephFS]]

## Latest follow-up

The tuned CephFS run `839fd11f9d6f4c1f9dac45e41314c8d1` used the 200G Ceph data path and 64/32 secondary-tier threads. It restored Qwen throughput to 1.300 req/s with zero request errors; direct Ceph byte telemetry remains an acceptance gap.