---
title: AgentX Weka storage and transfer comparison
date: 2026-08-18
type: research_appendix
experiment: ABC
source_report: 2026-08-18 - AgentX Weka admission prefetch first exploratory run
---

# AgentX Weka storage and transfer comparison

Return to [[../2026-08-18 - AgentX Weka admission prefetch first exploratory run|main report]].

Native 15-second storage evidence is split to preserve every sample:

- [[2026-08-18 - AgentX Weka admission prefetch first exploratory run/Runtime-node NVMe throughput comparison|Runtime-node NVMe throughput]]
- [[2026-08-18 - AgentX Weka admission prefetch first exploratory run/KV offload transfer throughput comparison|KV offload transfer throughput]]

Mean runtime-node NVMe read/write rates were 50.41/287.81 MiB/s for N=0 and 50.30/285.68 MiB/s for N=100. Mean CPU→GPU and GPU→CPU KV rates were 337.35/576.95 MiB/s for N=0 and 346.07/576.00 MiB/s for N=100. Broad storage pressure was effectively unchanged.