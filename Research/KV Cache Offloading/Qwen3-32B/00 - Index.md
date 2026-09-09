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

- [[2026-09-09 - Qwen3-32B selective loading bimodal comparison]] — End-to-end mechanism validation of the 1,024-token selective-load policy. The observed performance comparison is invalid because the concurrent arms shared nodes and NVMe, unsaturated cells had little external reuse, and the only externally active cell was overloaded. Includes a corrected isolated experiment.

## Current conclusion

The selective router-to-vLLM control path works for Qwen3-32B, but no performance benefit or model-specific threshold has yet been established. The next experiment must force external reuse at stable concurrency, isolate the treatments, and separately validate the 512-token recompute and 8,192-token load branches before evaluating the mixed workload.

## Related methodology

- [[../01 - Calibration Protocol|KV Cache Offloading calibration protocol]]
- [[../03 - KV Transfer Metrics and PromQL|KV transfer metrics and PromQL]]
- [[Engineering/Projects/Selective KV Loading and Offloading/07 - Selective loading calibration test plan|Selective loading calibration test plan]]