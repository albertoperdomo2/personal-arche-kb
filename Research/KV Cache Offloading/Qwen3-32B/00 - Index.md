---
title: "Qwen3-32B KV Cache Offloading Research"
date: "2026-09-11"
type: "research-index"
topic: "KV Cache Offloading"
model: "Qwen/Qwen3-32B"
status: "active"
---

# Qwen3-32B KV Cache Offloading Research

## Reports

- [[2026-09-11 - Qwen3-32B queue-aware selective-loading validation]] — Same-node, five-cell comparison of forced load, forced recompute, and an endpoint-local waiting-queue EWMA veto. Queue-aware achieved the highest throughput in four cells, was within 0.95% in the fifth, and improved geometric-mean throughput by 9.4% over the better static arm per cell. Includes four Vega-Lite plots, exact latency/throughput tables, mechanism evidence, limitations, and the complete 15-run registry.
- [[2026-09-10 - Qwen3-32B focused selective-loading crossover calibration]] — Same-node forced-policy calibration across 1K–8K reusable prefixes and C16–C64. Establishes that the preferred static action changes with endpoint pressure and motivates an EPP-visible waiting-queue veto.
- [[2026-09-09 - Qwen3-32B C96 selective-loading saturation snapshot]] — Compact view of the valid C96 subset with exact throughput, TTFT, queue, lookup, transfer, external-token-share, and NVMe-pressure measurements.
- [[2026-09-09 - Qwen3-32B selective-loading crossover sweep]] — Six-prefix, four-concurrency comparison of forced loading and forced recomputation. Demonstrates that skipping 512–1,024-token restores improves throughput under heavy restore pressure, but does not establish a static crossover because one replica on `gjfjh` repeatedly underperformed. Includes ten Vega-Lite plots and a focused follow-up design.
- [[2026-09-09 - Qwen3-32B selective loading bimodal comparison]] — End-to-end mechanism validation of the 1,024-token selective-load policy. The observed performance comparison is invalid because the concurrent arms shared nodes and NVMe, unsaturated cells had little external reuse, and the only externally active cell was overloaded. Includes a corrected isolated experiment.

## Current conclusion

The same-node queue-gate matrix is promising. With `minExternalReusableTokens: 1`, an endpoint-local waiting-queue EWMA that closes at 8 and reopens at 4 produced the highest request throughput in four of five pressure cells and finished 0.95% behind forced load in the fifth. Its geometric-mean throughput was 9.4% above an oracle selecting the better whole-run static policy per cell. At 8K/C32 and 8K/C64, it improved throughput over forced recompute by 15.9% and 19.6%, respectively, while avoiding forced load's extreme TTFT tails.

Keep the policy experimental. The matrix used one sequential run per cell and did not capture an unambiguous reason-labeled queue-veto decision counter. The next validation should interleave policy order and record load/recompute decisions with their cause, then correlate gate transitions with endpoint waiting. Do not add a storage-specific heuristic until this attribution confirms that the current queue signal is insufficient.

## Related methodology

- [[../01 - Calibration Protocol|KV Cache Offloading calibration protocol]]
- [[../03 - KV Transfer Metrics and PromQL|KV transfer metrics and PromQL]]
- [[Engineering/Projects/Selective KV Loading and Offloading/07 - Selective loading calibration test plan|Selective loading calibration test plan]]