---
title: "Qwen3-32B KV Cache Offloading Research"
date: "2026-09-09"
type: "research-index"
topic: "KV Cache Offloading"
model: "Qwen/Qwen3-32B"
status: "active"
---

# Qwen3-32B KV Cache Offloading Research

## Reports

- [[2026-09-09 - Qwen3-32B selective-loading crossover sweep]] — Six-prefix, four-concurrency comparison of forced loading and forced recomputation. Demonstrates that skipping 512–1,024-token restores improves throughput under heavy restore pressure, but does not establish a static crossover because one replica on `gjfjh` repeatedly underperformed. Includes nine Vega-Lite plots and a focused follow-up design.
- [[2026-09-09 - Qwen3-32B selective loading bimodal comparison]] — End-to-end mechanism validation of the 1,024-token selective-load policy. The observed performance comparison is invalid because the concurrent arms shared nodes and NVMe, unsaturated cells had little external reuse, and the only externally active cell was overloaded. Includes a corrected isolated experiment.

## Current conclusion

The selective router-to-vLLM control path works for Qwen3-32B, and the crossover sweep created substantial pressure in the external restore path. At C96, forced recomputation improved successful request throughput over forced loading by 27.6% for a 512-token reusable prefix and 36.8% for a 1,024-token prefix. The sweep does not establish a defensible static threshold because results were non-monotonic, useful external reuse remained low, NVMe was saturated, and one replica on `gjfjh` repeatedly ran 3–5× slower.

Exclude `gjfjh` with node affinity until it passes a separate balance diagnostic. The next calibration should run paired policies sequentially on the same known-balanced node, use a smaller external corpus, and repeat the focused 1K–8K range before selecting a threshold.

## Related methodology

- [[../01 - Calibration Protocol|KV Cache Offloading calibration protocol]]
- [[../03 - KV Transfer Metrics and PromQL|KV transfer metrics and PromQL]]
- [[Engineering/Projects/Selective KV Loading and Offloading/07 - Selective loading calibration test plan|Selective loading calibration test plan]]