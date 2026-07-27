---
title: Qwen3.6 AgentX six-run CephFS tuning matrix — summary
date: 2026-07-27
type: research-summary
experiment: KV Cache Offloading
model: Qwen3.6-35B-A3B
---

# Qwen3.6 AgentX six-run CephFS tuning matrix — summary

## Run table

| Configuration | Run | Requests/s | Output tokens/s | Mean TTFT (s) | Mean E2E (s) | Completed | Errors |
|---|---|---:|---:|---:|---:|---:|---:|
| No offload | `6ded92329a4844c5a4c6f11f5cab764c` | 0.578 | 420.1 | 32.968 | 50.511 | 1057 | 0 |
| CPU 64 GiB | `dc57aaea850048c1a22210debe712ed5` | 1.191 | 854.7 | 14.539 | 25.439 | 2176 | 0 |
| CPU 64 GiB + NVMe | `0ade4cb5a63941ffb7bbe49234d091fa` | 1.284 | 924.0 | 10.572 | 20.287 | 2343 | 0 |
| CPU 64 GiB + CephFS | `004647544b294b20931b7af7f94c6c83` | 0.711 | 496.1 | 20.579 | 32.841 | 1301 | 0 |
| CPU 64 GiB + CephFS (tuned) | `839fd11f9d6f4c1f9dac45e41314c8d1` | 1.300 | 936.0 | 10.495 | 20.129 | 2363 | 0 |
| CPU 64 GiB + CephFS (tuned, 2) | `98691417d921402f8c1d306a45a62c2c` | 1.518 | 1094.0 | 0.522 | 13.112 | 2766 | 0 |

## Request throughput

Figure 1 compares successful request throughput.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 1 — Successful request throughput","data":{"values":[{"configuration":"No offload","requests_per_second":0.578},{"configuration":"CPU 64 GiB","requests_per_second":1.191},{"configuration":"CPU 64 GiB + NVMe","requests_per_second":1.284},{"configuration":"CPU 64 GiB + CephFS","requests_per_second":0.711},{"configuration":"CPU 64 GiB + CephFS (tuned)","requests_per_second":1.300},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","requests_per_second":1.518}]},"mark":"bar","encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration"},"y":{"field":"requests_per_second","type":"quantitative","title":"Successful request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS","CPU 64 GiB + CephFS (tuned)","CPU 64 GiB + CephFS (tuned, 2)"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728","#9467bd","#e377c2"]}}},"config":{"view":{"stroke":null}}}
```

## Output throughput

Figure 2 compares output-token throughput.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 2 — Output-token throughput","data":{"values":[{"configuration":"No offload","output_tokens_per_second":420.1},{"configuration":"CPU 64 GiB","output_tokens_per_second":854.7},{"configuration":"CPU 64 GiB + NVMe","output_tokens_per_second":924.0},{"configuration":"CPU 64 GiB + CephFS","output_tokens_per_second":496.1},{"configuration":"CPU 64 GiB + CephFS (tuned)","output_tokens_per_second":936.0},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","output_tokens_per_second":1094.0}]},"mark":"bar","encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration"},"y":{"field":"output_tokens_per_second","type":"quantitative","title":"Output-token throughput (tokens/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS","CPU 64 GiB + CephFS (tuned)","CPU 64 GiB + CephFS (tuned, 2)"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728","#9467bd","#e377c2"]}}},"config":{"view":{"stroke":null}}}
```

## Mean latency

Figure 3 compares mean TTFT and E2E latency.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 3 — Mean client latency","data":{"values":[{"configuration":"No offload","metric":"TTFT","latency_seconds":32.968},{"configuration":"No offload","metric":"E2E","latency_seconds":50.511},{"configuration":"CPU 64 GiB","metric":"TTFT","latency_seconds":14.539},{"configuration":"CPU 64 GiB","metric":"E2E","latency_seconds":25.439},{"configuration":"CPU 64 GiB + NVMe","metric":"TTFT","latency_seconds":10.572},{"configuration":"CPU 64 GiB + NVMe","metric":"E2E","latency_seconds":20.287},{"configuration":"CPU 64 GiB + CephFS","metric":"TTFT","latency_seconds":20.579},{"configuration":"CPU 64 GiB + CephFS","metric":"E2E","latency_seconds":32.841},{"configuration":"CPU 64 GiB + CephFS (tuned)","metric":"TTFT","latency_seconds":10.495},{"configuration":"CPU 64 GiB + CephFS (tuned)","metric":"E2E","latency_seconds":20.129},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","metric":"TTFT","latency_seconds":0.522},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","metric":"E2E","latency_seconds":13.112}]},"mark":"bar","encoding":{"x":{"field":"metric","type":"nominal","title":"Latency metric"},"xOffset":{"field":"configuration"},"y":{"field":"latency_seconds","type":"quantitative","title":"Mean latency (seconds)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"scheme":"category10"}}},"config":{"view":{"stroke":null}}}
```

Full-granularity running/waiting request and prompt-token source time series remain in the detailed six-run report and its linked appendices.