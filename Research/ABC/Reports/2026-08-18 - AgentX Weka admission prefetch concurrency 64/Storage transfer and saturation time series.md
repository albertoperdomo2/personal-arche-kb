---
title: AgentX Weka concurrency 64 storage transfer and saturation time series
date: 2026-08-18
type: research_appendix
experiment: ABC
source_report: 2026-08-18 - AgentX Weka admission prefetch concurrency 64
---

# AgentX Weka concurrency 64 storage, transfer, and saturation time series

Return to [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64|main report]].

Native 15-second evidence is split by direction to preserve every sample within the transport limit:

- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/NVMe read throughput|NVMe read throughput]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/NVMe write throughput|NVMe write throughput]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/CPU to GPU KV throughput|CPU→GPU KV throughput]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/GPU to CPU KV throughput|GPU→CPU KV throughput]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Preemption rate comparison|Preemption rate]]

The concurrency-64 cells drove approximately 0.91 GiB/s mean NVMe reads and 6.4–6.5 GiB/s mean CPU→GPU KV traffic, far above the concurrency-32 regime.