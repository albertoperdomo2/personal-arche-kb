---
title: Qwen3.6 AgentX selected CephFS matrix summary
date: 2026-07-28
type: research-summary
experiment: KV Cache Offloading
model: Qwen3.6-35B-A3B
---

# Qwen3.6 AgentX selected matrix summary

This reduced view keeps configurations **1, 2, 3, and 6**: no offload, CPU 64 GiB, CPU 64 GiB + NVMe, and CPU 64 GiB + CephFS (tuned, 2).

## Run table

| Configuration | Run | Requests/s | Output tokens/s | Mean TTFT (s) | Mean E2E (s) | Completed | Errors |
|---|---|---:|---:|---:|---:|---:|---:|
| No offload | `6ded92329a4844c5a4c6f11f5cab764c` | 0.578 | 420.1 | 32.968 | 50.511 | 1057 | 0 |
| CPU 64 GiB | `dc57aaea850048c1a22210debe712ed5` | 1.191 | 854.7 | 14.539 | 25.439 | 2176 | 0 |
| CPU 64 GiB + NVMe | `0ade4cb5a63941ffb7bbe49234d091fa` | 1.284 | 924.0 | 10.572 | 20.287 | 2343 | 0 |
| CPU 64 GiB + CephFS (tuned, 2) | `98691417d921402f8c1d306a45a62c2c` | 1.518 | 1094.0 | 0.522 | 13.112 | 2766 | 0 |

## Prompt-token source comparison

Figure 1 compares integrated prompt-token source shares. Provenance: native prompt-token source counters.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 1 — Prompt tokens by source","data":{"values":[{"configuration":"No offload","source":"External KV transfer","share_percent":0},{"configuration":"No offload","source":"Local HBM hit","share_percent":2.809},{"configuration":"No offload","source":"Local compute","share_percent":97.191},{"configuration":"CPU 64 GiB","source":"External KV transfer","share_percent":76.607},{"configuration":"CPU 64 GiB","source":"Local HBM hit","share_percent":1.088},{"configuration":"CPU 64 GiB","source":"Local compute","share_percent":22.305},{"configuration":"CPU 64 GiB + NVMe","source":"External KV transfer","share_percent":88.646},{"configuration":"CPU 64 GiB + NVMe","source":"Local HBM hit","share_percent":0.209},{"configuration":"CPU 64 GiB + NVMe","source":"Local compute","share_percent":11.145},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","source":"External KV transfer","share_percent":34.743},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","source":"Local HBM hit","share_percent":54.570},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","source":"Local compute","share_percent":10.687}]},"mark":"bar","encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS (tuned, 2)"]},"y":{"field":"share_percent","type":"quantitative","title":"Prompt-token source share (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"Prompt-token source","scale":{"scheme":"category10"}}},"config":{"view":{"stroke":null}}}
```

## Headline performance

Figure 2 compares request throughput.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 2 — Successful request throughput","data":{"values":[{"configuration":"No offload","requests_per_second":0.578},{"configuration":"CPU 64 GiB","requests_per_second":1.191},{"configuration":"CPU 64 GiB + NVMe","requests_per_second":1.284},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","requests_per_second":1.518}]},"mark":"bar","encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS (tuned, 2)"]},"y":{"field":"requests_per_second","type":"quantitative","title":"Successful request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"scheme":"category10"}}},"config":{"view":{"stroke":null}}}
```

## Why tuned-2 has high local HBM hits

The source counters show a real shift: tuned-2 reports **54.57% local HBM hit**, compared with 1.09% for CPU-only and 0.21% for NVMe. External KV transfer is correspondingly lower at 34.74%, versus 76.61% for CPU-only and 88.65% for NVMe.

The counters alone do not prove that CephFS is faster. The most likely explanation is hierarchy state: tuned-2 retained a much larger reusable working set in HBM during this run, so requests were served locally instead of requiring external KV loads. That can result from different cache residency/eviction timing, request scheduling, or prefix reuse realization—not necessarily from lower CephFS bandwidth demand. The high local-hit share is consistent with less KV movement and therefore lower transfer demand.

This conclusion is also constrained by the run evidence: tuned-2 has the same seed and concurrency target, zero client errors, and the strongest end-to-end result, but it is still one repetition and lacks direct CephFS byte/IOPS telemetry. To distinguish “better HBM residency” from workload-phase variation, repeat tuned-2 and collect KV block residency/eviction, external-load counts, and Ceph client read/write counters at the same cadence.Traceback (most recent call last):
  File "<stdin>", line 14, in <module>
  File "<stdin>", line 4, in P
FileNotFoundError: [Errno 2] No such file or directory: 'experiment-256-clean/6ded92329a4844c5a4c6f11f5cab764c/benchmark/profile_export_aiperf.json'