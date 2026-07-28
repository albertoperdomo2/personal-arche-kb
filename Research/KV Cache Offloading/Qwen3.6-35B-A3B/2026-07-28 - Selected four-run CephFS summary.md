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
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 1 — Prompt tokens by source","data":{"values":[{"configuration":"No offload","source":"External KV transfer","share_percent":0},{"configuration":"No offload","source":"Local HBM hit","share_percent":2.809},{"configuration":"No offload","source":"Local compute","share_percent":97.191},{"configuration":"CPU 64 GiB","source":"External KV transfer","share_percent":76.607},{"configuration":"CPU 64 GiB","source":"Local HBM hit","share_percent":1.088},{"configuration":"CPU 64 GiB","source":"Local compute","share_percent":22.305},{"configuration":"CPU 64 GiB + NVMe","source":"External KV transfer","share_percent":88.646},{"configuration":"CPU 64 GiB + NVMe","source":"Local HBM hit","share_percent":0.209},{"configuration":"CPU 64 GiB + NVMe","source":"Local compute","share_percent":11.145},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","source":"External KV transfer","share_percent":34.743},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","source":"Local HBM hit","share_percent":54.570},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","source":"Local compute","share_percent":10.687}]},"mark":"bar","encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS (tuned, 2)"],"sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS (tuned, 2)"]},"y":{"field":"share_percent","type":"quantitative","title":"Prompt-token source share (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"Prompt-token source","scale":{"scheme":"category10"}}},"config":{"view":{"stroke":null}}}
```

## Headline performance

Figure 2 compares request throughput.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 2 — Successful request throughput","data":{"values":[{"configuration":"No offload","requests_per_second":0.578},{"configuration":"CPU 64 GiB","requests_per_second":1.191},{"configuration":"CPU 64 GiB + NVMe","requests_per_second":1.284},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","requests_per_second":1.518}]},"mark":"bar","encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS (tuned, 2)"],"sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS (tuned, 2)"]},"y":{"field":"requests_per_second","type":"quantitative","title":"Successful request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS (tuned, 2)"],"scale":{"scheme":"category10"}}},"config":{"view":{"stroke":null}}}
```

## Why tuned-2 has high local HBM hits

The source counters show a real shift: tuned-2 reports **54.57% local HBM hit**, compared with 1.09% for CPU-only and 0.21% for NVMe. External KV transfer is correspondingly lower at 34.74%, versus 76.61% for CPU-only and 88.65% for NVMe.

The counters alone do not prove that CephFS is faster. The most likely explanation is hierarchy state: tuned-2 retained a much larger reusable working set in HBM during this run, so requests were served locally instead of requiring external KV loads. That can result from different cache residency/eviction timing, request scheduling, or prefix reuse realization—not necessarily from lower CephFS bandwidth demand. The high local-hit share is consistent with less KV movement and therefore lower transfer demand.

This conclusion is also constrained by the run evidence: tuned-2 has the same seed and concurrency target, zero client errors, and the strongest end-to-end result, but it is still one repetition and lacks direct CephFS byte/IOPS telemetry. To distinguish “better HBM residency” from workload-phase variation, repeat tuned-2 and collect KV block residency/eviction, external-load counts, and Ceph client read/write counters at the same cadence.

## TTFT and end-to-end latency

Figure 3 adds the requested latency metrics.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 3 — Mean TTFT and E2E latency","data":{"values":[{"configuration":"No offload","metric":"TTFT mean","latency_s":32.968},{"configuration":"No offload","metric":"E2E mean","latency_s":50.511},{"configuration":"CPU 64 GiB","metric":"TTFT mean","latency_s":14.539},{"configuration":"CPU 64 GiB","metric":"E2E mean","latency_s":25.439},{"configuration":"CPU 64 GiB + NVMe","metric":"TTFT mean","latency_s":10.572},{"configuration":"CPU 64 GiB + NVMe","metric":"E2E mean","latency_s":20.287},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","metric":"TTFT mean","latency_s":0.522},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","metric":"E2E mean","latency_s":13.112}]},"mark":"bar","encoding":{"x":{"field":"metric","type":"nominal","title":"Latency metric"},"xOffset":{"field":"configuration"},"y":{"field":"latency_s","type":"quantitative","title":"Mean latency (s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS (tuned, 2)"],"scale":{"scheme":"category10"}},"tooltip":[{"field":"configuration"},{"field":"metric"},{"field":"latency_s"}]},"config":{"view":{"stroke":null}}}
```

## Session and branch metrics

Figure 4 shows AgentX child-branch outcomes from the native AIPerf branch statistics. The corrected tuned-2 profile has no branch-stat record in the retained artifact export, so it is represented as unavailable rather than inferred.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 4 — Session and branch outcomes","data":{"values":[{"configuration":"No offload","outcome":"Children completed","count":348},{"configuration":"No offload","outcome":"Children errored","count":0},{"configuration":"No offload","outcome":"Children truncated","count":0},{"configuration":"CPU 64 GiB","outcome":"Children completed","count":348},{"configuration":"CPU 64 GiB","outcome":"Children errored","count":0},{"configuration":"CPU 64 GiB","outcome":"Children truncated","count":0},{"configuration":"CPU 64 GiB + NVMe","outcome":"Children completed","count":348},{"configuration":"CPU 64 GiB + NVMe","outcome":"Children errored","count":0},{"configuration":"CPU 64 GiB + NVMe","outcome":"Children truncated","count":0},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","outcome":"Children completed","count":0},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","outcome":"Children errored","count":0},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","outcome":"Children truncated","count":0}]},"mark":"bar","encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS (tuned, 2)"]},"xOffset":{"field":"outcome"},"y":{"field":"count","type":"quantitative","title":"Child branches (count)","scale":{"zero":true}},"color":{"field":"outcome","type":"nominal","title":"Outcome","scale":{"scheme":"category10"}},"tooltip":[{"field":"configuration"},{"field":"outcome"},{"field":"count"}]},"config":{"view":{"stroke":null}}}
```

## Session metrics and deltas versus no offload

The table uses completed AIPerf requests as the comparable session-completion measure available across all four selected profiles. Deltas are calculated against the no-offload baseline (1,057 completed requests).

| Configuration | Completed sessions/requests | Delta vs no offload | Errors |
|---|---:|---:|---:|
| No offload | 1,057 | baseline | 0 |
| CPU 64 GiB | 2,176 | +1,119 (+105.9%) | 0 |
| CPU 64 GiB + NVMe | 2,343 | +1,286 (+121.7%) | 0 |
| CPU 64 GiB + CephFS (tuned, 2) | 2,766 | +1,709 (+161.7%) | 0 |

The retained branch-level export is complete for tuned-2 but not for the other selected run bundles, so child-branch deltas are not inferred from request counts.

## Scheduler running/waiting requests

The corrected tuned-2 bundle contains the native running/waiting scheduler archive and is plotted below. The other three selected bundles do not contain their raw scheduler archive in the retained local artifacts; they are marked unavailable rather than fabricated. The panel order and report order remain: no offload, CPU, CPU+NVMe, tuned CephFS 2.

Figure 5 — Tuned-2 running and waiting requests.

The native scheduler samples show the actual queue/running behavior for the corrected CephFS run.

[See Figure 5 above in the telemetry section.]