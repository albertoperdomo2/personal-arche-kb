---
title: AgentX Weka concurrency 64 admission-prefetch mechanism time series
date: 2026-08-18
type: research_appendix
experiment: ABC
source_report: 2026-08-18 - AgentX Weka admission prefetch concurrency 64
---

# AgentX Weka concurrency 64 admission-prefetch mechanism time series

Return to [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64|main report]].

The native vLLM metric-log evidence is split to preserve all 181 profiling-window records:

- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Mechanism cumulative selection|Cumulative selection and submission]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64/Mechanism cumulative resolution|Cumulative resolution and lateness]]

Exact totals are 123,419 attempted, 90,316 redundant, 33,103 promoted, 19,508 useful, 12,506 load-failed, 594 wasted, and 14,031 late. Late overlaps final outcomes; 495 promotions remained unresolved at the final profiling log record.