---
title: AgentX Weka concurrency 64 storage transfer and saturation time series
date: 2026-08-18
type: research_appendix
experiment: ABC
source_report: 2026-08-18 - AgentX Weka admission prefetch concurrency 64
---

# AgentX Weka concurrency 64 storage, transfer, and saturation time series

Return to [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64|main report]].

Native 15-second evidence is split to preserve every sample:

- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/NVMe throughput comparison|Runtime-node NVMe throughput]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/KV transfer throughput comparison|KV transfer throughput]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Preemption rate comparison|Preemption rate]]

The concurrency-64 cells drove approximately 0.91 GiB/s mean NVMe reads and 6.4–6.5 GiB/s mean CPU→GPU KV traffic, far above the concurrency-32 regime.