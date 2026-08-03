---
title: "Gemma 4 31B IT KV-cache offloading"
date: "2026-08-03"
type: "research-model-index"
model: "google/gemma-4-31B-it"
topic: "KV Cache Offloading"
---

# Gemma 4 31B IT

## Current conclusion

At TP2/U0.92/C8 with a 256 GiB host tier, offload is strongly beneficial. CephFS is the clean single-seed winner at 0.1844 req/s and 145.4 output tok/s, 3.10× and 4.41× the no-offload baseline; CPU-only reaches 0.1202 req/s. NVMe is active but conditionally accepted because 15 store refusals and deferred requests produce an extreme latency tail.

## Reports

- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison|2026-08-03 — Gemma 4 31B IT KV-cache offload comparison]]
- [[2026-07-25 - AgentX offload failure matrix|2026-07-25 — AgentX offload failure matrix]]