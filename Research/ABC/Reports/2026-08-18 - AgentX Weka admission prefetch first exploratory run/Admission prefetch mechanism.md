---
title: AgentX Weka admission prefetch mechanism time series
date: 2026-08-18
type: research_appendix
experiment: ABC
source_report: 2026-08-18 - AgentX Weka admission prefetch first exploratory run
---

# AgentX Weka admission prefetch mechanism time series

Return to [[../2026-08-18 - AgentX Weka admission prefetch first exploratory run|main report]].

The native ten-second vLLM mechanism evidence is split across the following articles to preserve all 179 profiling-window samples without exceeding the renderer payload limit:

- [[2026-08-18 - AgentX Weka admission prefetch first exploratory run/Mechanism cumulative selection|Cumulative selection and submission]]
- [[2026-08-18 - AgentX Weka admission prefetch first exploratory run/Mechanism cumulative resolution|Cumulative final resolution]]
- [[2026-08-18 - AgentX Weka admission prefetch first exploratory run/Mechanism interval submission and lateness|Per-interval submission and lateness]]
- [[2026-08-18 - AgentX Weka admission prefetch first exploratory run/Mechanism interval resolution|Per-interval useful and failed outcomes]]

## Exact totals

| Outcome | Blocks | Fraction |
|---|---:|---:|
| Attempted | 85,010 | 100% |
| Redundant | 77,356 | 90.99% of attempts |
| Promoted | 7,654 | 9.00% of attempts |
| Useful | 990 | 12.93% of promotions |
| Load failed | 6,664 | 87.08% of promotions |
| Late | 7,539 | 98.50% of promotions; overlaps resolution |

Useful first appeared after approximately 5.2 profiling minutes. The first five minutes contained 2,329 promotions and 2,329 failures.