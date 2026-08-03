---
title: "Gemma 4 31B IT KV-cache offloading"
date: "2026-08-03"
type: "research-model-index"
model: "google/gemma-4-31B-it"
topic: "KV Cache Offloading"
---

# Gemma 4 31B IT

## Current conclusion

At TP2/U0.92/C8 with a 256 GiB host tier, offload is strongly beneficial. CephFS remains the clean single-seed winner at 0.1844 req/s and 145.4 output tok/s, 3.10× and 4.41× the no-offload baseline; CPU-only reaches 0.1202 req/s. The tuned NVMe 64-read/32-write cell improves to 0.1113 req/s and 82.6 output tok/s, but remains conditionally accepted because 11 store refusals, sustained deferred requests, and 210.21-second P95 TTFT preserve a severe latency tail.

## Reports

- [[2026-08-03 - Gemma 4 31B IT KV-cache offload comparison|2026-08-03 — Gemma 4 31B IT KV-cache offload comparison]]
- [[2026-07-25 - AgentX offload failure matrix|2026-07-25 — AgentX offload failure matrix]]