---
title: "ABC KV lookup experiments"
date: "2026-08-10"
type: "research-index"
experiment: "ABC CPU offload KV lookup comparison"
status: "active"
---

# ABC

## Reports

- [[2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report|2026-08-10 — Nemotron no-offload versus CPU-offload KV lookup report]]

## Current conclusion

The CPU run exposes synchronous KV lookup P99 at roughly 30–33 ms late in the run. External secondary-tier async lookup telemetry is not populated in either run. The comparison is conditional because the vLLM images differ (`v0.26.0` versus `nightly`).