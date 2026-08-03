---
title: "Gemma 4 31B IT KV-cache offloading"
date: "2026-08-03"
type: "research-model-index"
model: "google/gemma-4-31B-it"
topic: "KV Cache Offloading"
---

# Gemma 4 31B IT

## Current conclusion

At TP2/U0.92/C8 with a 256 GiB host tier, offload is strongly beneficial. The selected NVMe 64-read/64-write cell and CephFS form the clean top tier: NVMe reaches 0.1771 req/s and 146.1 output tok/s with zero store refusals, while CephFS reaches 0.1844 req/s and 145.4 output tok/s. CPU-only reaches 0.1202 req/s. The thread sweep identifies write-side filesystem concurrency as the leading explanation for the earlier NVMe regressions; this remains a single-seed conclusion pending repetition.

## Reports

- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison|2026-08-03 — Gemma 4 31B IT KV-cache offload comparison]]
- [[2026-07-25 - AgentX offload failure matrix|2026-07-25 — AgentX offload failure matrix]]