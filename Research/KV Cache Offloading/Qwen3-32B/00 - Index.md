---
title: "Qwen3-32B KV Cache Offloading Research"
date: "2026-09-13"
type: "research-index"
topic: "KV Cache Offloading"
model: "Qwen/Qwen3-32B"
status: "active"
---

# Qwen3-32B KV Cache Offloading Research

## Reports

- [[2026-09-13 - Qwen3-32B combined token and queue selective-loading validation]] — Twenty-run same-node comparison of forced load, forced recompute, token-only 1,024, and combined 1,024-token plus 8-waiting-request selective loading. Includes request, total-token, and output-token throughput, mean and p99 TTFT, mean and p99 ITL, queue evidence, eight Vega-Lite plots, and the complete run registry. The combined configuration improves geometric-mean throughput by 8.8% over the better static policy per cell, but is 0.58% below token-only. Archived queue samples never reach 8, so the outcome validates the deployable configuration against static controls without proving incremental queue-veto value.
- [[2026-09-11 - Qwen3-32B queue-aware selective-loading validation]] — Same-node, five-cell comparison of forced load, forced recompute, and an endpoint-local waiting-queue EWMA veto with a one-token evidence threshold. Queue-aware achieved the highest throughput in four cells and improved geometric-mean throughput by 9.4% over the better static arm per cell. The follow-up threshold-4 sensitivity run was worse than threshold 8.
- [[2026-09-10 - Qwen3-32B focused selective-loading crossover calibration]] — Same-node forced-policy calibration across 1K-8K reusable prefixes and C16-C64. Establishes that the preferred static action changes with endpoint pressure and motivates an EPP-visible waiting-queue veto.
- [[2026-09-09 - Qwen3-32B C96 selective-loading saturation snapshot]] — Compact view of the valid C96 subset with exact throughput, TTFT, queue, lookup, transfer, external-token-share, and NVMe-pressure measurements.
- [[2026-09-09 - Qwen3-32B selective-loading crossover sweep]] — Six-prefix, four-concurrency comparison of forced loading and forced recomputation. Demonstrates that skipping 512-1,024-token restores improves throughput under heavy restore pressure, but does not establish a static crossover because one replica on `gjfjh` repeatedly underperformed.
- [[2026-09-09 - Qwen3-32B selective loading bimodal comparison]] — End-to-end mechanism validation of the 1,024-token selective-load policy. The observed performance comparison is invalid because the concurrent arms shared nodes and NVMe, unsaturated cells had little external reuse, and the only externally active cell was overloaded.

## Current conclusion

The same-node combined-policy matrix validates `minExternalReusableTokens: 1024` plus `maxWaitingRequests: 8` as a useful experimental configuration for this Qwen3-32B deployment. It beats the better forced-load or forced-recompute arm in four of five cells and improves geometric-mean throughput by 8.8% over the per-cell best static action. It also avoids the severe forced-load TTFT tail collapse in the high-pressure cells.

The queue gate's incremental benefit remains unproven. Token-only 1,024 achieved 9.4% geometric-mean throughput improvement over the best static arm and exceeded the combined policy by 0.58%. No archived 15-second waiting sample reached 8 in a combined run, though shorter EPP-visible spikes cannot be excluded. Treat the 1,024-token threshold as the demonstrated selector and the queue threshold of 8 as an experimental safety gate.

If further queue evidence is required before the PR, compare token-only and combined policies at C96 or C128 with enough sustained waiting pressure to cross 8. Reason-labeled action telemetry is still the preferred follow-up because it can prove whether the queue veto mutates requests.

## Related methodology

- [[../01 - Calibration Protocol|KV Cache Offloading calibration protocol]]
- [[../03 - KV Transfer Metrics and PromQL|KV transfer metrics and PromQL]]
- [[Engineering/Projects/Selective KV Loading and Offloading/07 - Selective loading calibration test plan|Selective loading calibration test plan]]