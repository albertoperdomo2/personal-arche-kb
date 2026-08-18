---
title: AgentX Weka concurrency 64 scheduler and latency time series
date: 2026-08-18
type: research_appendix
experiment: ABC
source_report: 2026-08-18 - AgentX Weka admission prefetch concurrency 64
---

# AgentX Weka concurrency 64 scheduler and latency time series

Return to [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64|main report]].

Native 15-second evidence is split by scheduler state and quantile to preserve every sample within the transport limit:

- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Requests running|Running requests]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Requests waiting|Waiting requests]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Queue time p50|Queue time p50]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Queue time p90|Queue time p90]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Queue time p99|Queue time p99]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/TTFT p50|TTFT p50]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/TTFT p90|TTFT p90]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/TTFT p99|TTFT p99]]

N=0 averaged 27.56 running and 6.03 waiting requests; N=100 averaged 27.47 running and 6.22 waiting. The pressure objective was achieved.