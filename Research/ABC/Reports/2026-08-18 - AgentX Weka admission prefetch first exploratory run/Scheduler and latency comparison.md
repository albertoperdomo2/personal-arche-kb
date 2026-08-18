---
title: AgentX Weka scheduler and latency comparison
date: 2026-08-18
type: research_appendix
experiment: ABC
source_report: 2026-08-18 - AgentX Weka admission prefetch first exploratory run
---

# AgentX Weka scheduler and latency comparison

Return to [[../2026-08-18 - AgentX Weka admission prefetch first exploratory run|main report]].

Native 15-second scheduler and TTFT evidence is split to preserve every sample:

- [[2026-08-18 - AgentX Weka admission prefetch first exploratory run/Running and waiting requests comparison|Running and waiting requests]]
- [[2026-08-18 - AgentX Weka admission prefetch first exploratory run/TTFT p50 and p90 comparison|TTFT p50 and p90]]
- [[2026-08-18 - AgentX Weka admission prefetch first exploratory run/TTFT p99 comparison|TTFT p99]]

N=0 averaged 11.05 running and 0.21 waiting requests; N=100 averaged 11.47 running and 0.24 waiting. Waiting p95 was only 2 and 1. This profile therefore offered little admission-prefetch lead time.