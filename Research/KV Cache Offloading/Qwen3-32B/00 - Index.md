---
title: "Qwen3-32B KV Cache Offloading Research"
date: "2026-09-14"
type: "research-index"
topic: "KV Cache Offloading"
model: "Qwen/Qwen3-32B"
status: "active"
---

# Qwen3-32B KV Cache Offloading Research

## Reports

- [[2026-09-13 - Qwen3-32B combined token and queue selective-loading validation]] — Fifteen-run same-node validation of the exact combined 1,024-token plus 8-waiting-request policy against forced load/current behavior and forced recompute. Includes request, total-token, and output-token throughput, mean and p99 TTFT, mean and p99 ITL, combined-arm queue/mechanism evidence, eight Vega-Lite plots, and the complete selected-run registry. The proposed configuration improves geometric-mean throughput by 8.8% over the better static action per cell and avoids forced-load TTFT tail collapse. Archived 15-second queue samples peak at 7 and no reason-labeled action identifies gate decisions, so the report does not claim the queue gate fired.
- [[2026-09-11 - Qwen3-32B queue-aware selective-loading validation]] — Same-node, five-cell comparison of forced load, forced recompute, and an endpoint-local waiting-queue EWMA veto with a one-token evidence threshold. Queue-aware achieved the highest throughput in four cells and improved geometric-mean throughput by 9.4% over the better static arm per cell. The follow-up threshold-4 sensitivity run was worse than threshold 8.
- [[2026-09-10 - Qwen3-32B focused selective-loading crossover calibration]] — Same-node forced-policy calibration across 1K-8K reusable prefixes and C16-C64. Establishes that the preferred static action changes with endpoint pressure and motivates an EPP-visible waiting-queue veto.
- [[2026-09-09 - Qwen3-32B C96 selective-loading saturation snapshot]] — Compact view of the valid C96 subset with exact throughput, TTFT, queue, lookup, transfer, external-token-share, and NVMe-pressure measurements.
- [[2026-09-09 - Qwen3-32B selective-loading crossover sweep]] — Six-prefix, four-concurrency comparison of forced loading and forced recomputation. Demonstrates that skipping 512-1,024-token restores improves throughput under heavy restore pressure, but does not establish a static crossover because one replica on `gjfjh` repeatedly underperformed.
- [[2026-09-09 - Qwen3-32B selective loading bimodal comparison]] — End-to-end mechanism validation of the 1,024-token selective-load policy. The observed performance comparison is invalid because the concurrent arms shared nodes and NVMe, unsaturated cells had little external reuse, and the only externally active cell was overloaded.

## Current conclusion

The same-node 15-run matrix validates `minExternalReusableTokens: 1024` plus `maxWaitingRequests: 8` as a useful proposed configuration for this Qwen3-32B deployment against forced load/current behavior and forced recompute. It beats the better static request throughput in four of five cells, improves geometric-mean throughput by 8.8% over the per-cell better static action, improves geometric-mean throughput by 20.6% over forced load and 12.6% over forced recompute, and avoids severe forced-load TTFT tail collapse in the high-pressure cells.

The queue component's action remains unverified. No archived 15-second waiting sample reached 8 in a combined run—the maximum was 7—and no reason-labeled action or request trace records whether waiting pressure changed a load decision. Treat the result as validation of the exact combined configuration against static controls, not as evidence that the queue gate fired or as an estimate of either component's incremental benefit.

If direct gate evidence is required before the PR, add reason-labeled action telemetry and run the combined policy under sustained endpoint-local waiting EWMA above 8 on the same pinned node.
## Related methodology

- [[../01 - Calibration Protocol|KV Cache Offloading calibration protocol]]
- [[../03 - KV Transfer Metrics and PromQL|KV transfer metrics and PromQL]]
- [[Engineering/Projects/Selective KV Loading and Offloading/07 - Selective loading calibration test plan|Selective loading calibration test plan]]