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

The four-run ABC report now includes CephFS and NVMe tiering async lookup telemetry. Both storage tiers show multi-second lookup delays and P99 values reaching the 10-second histogram ceiling; overflow is unavailable, so tail values are lower bounds.