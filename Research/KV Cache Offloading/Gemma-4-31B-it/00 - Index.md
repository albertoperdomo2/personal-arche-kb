---
title: "Gemma 4 31B IT KV-cache offloading"
date: "2026-08-05"
type: "research-model-index"
model: "google/gemma-4-31B-it"
topic: "KV Cache Offloading"
---

# Gemma 4 31B IT

## Current conclusion

The clean single-replica TP2/U0.92/C8 matrix with a 256 GiB host tier still shows strong offload benefit: NVMe reaches 0.1771 req/s and 146.1 output tok/s, CephFS reaches 0.1844 req/s and 145.4 output tok/s, and CPU-only reaches 0.1202 req/s.

The first two-replica TP2/U0.92/C16 matrix does **not** reproduce that scaling. No offload reaches 0.1208 req/s—2.03× the prior single-replica baseline—while CPU reaches 0.1118, NVMe 0.0663, and CephFS 0.0222 req/s. The storage-tier cells begin near baseline generation speed and collapse as queues and external-cache activity grow. The leading hypothesis is replica-local KV ownership combined with approximate prefix routing and asynchronous promotion retries; insufficient AIPerf concurrency is rejected as the primary cause because the baseline scales ideally and the tiered cells already have persistent waiting queues. Host co-tenancy is a material confound because all four configurations shared the same two nodes.

Keep TP2 and U0.92 for the next diagnostic run. Isolate one configuration at a time, pin or make sessions sticky to replicas, compare queue-only/current approximate/precise KV-event routing, use separate CephFS roots per replica, and only then run a C8/C12/C16/C24 response curve.

## Reports

- [[2026-08-05 - Gemma 4 31B IT two-replica KV-cache offload comparison|2026-08-05 — Gemma 4 31B IT two-replica KV-cache offload comparison]]
- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison|2026-08-03 — Gemma 4 31B IT KV-cache offload comparison]]
- [[2026-07-25 - AgentX offload failure matrix|2026-07-25 — AgentX offload failure matrix]]