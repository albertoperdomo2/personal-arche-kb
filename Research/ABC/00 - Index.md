---
title: "ABC KV lookup experiments"
date: "2026-08-10"
type: "research-index"
experiment: "ABC CPU offload KV lookup comparison"
status: "active"
---

# ABC

## Reports

- [[2026-08-11 - ABC Nemotron four-tier KV lookup comparison - Revision 1|2026-08-11 — Four-tier KV lookup comparison — Revision 1]]
- [[2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report|2026-08-10 — Nemotron no-offload versus CPU-offload KV lookup report]]

## Current conclusion

The added CephFS and NVMe runs expose tiering async lookup telemetry. Both show multi-second lookup delays and P99 values reaching the 10-second histogram ceiling; overflow is unavailable, so tail values are lower bounds. The comparison remains conditional because runtime images and storage thread settings differ.