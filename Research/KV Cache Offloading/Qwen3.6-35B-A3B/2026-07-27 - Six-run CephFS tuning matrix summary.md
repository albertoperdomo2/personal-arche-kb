Warning: truncated output (original token count: 118239)
Total output lines: 50

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

Figure 1 — Request throughput.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 1 \u2014 Request throughput","data":{"values":[{"configuration":"No offload","request_rps":0.5780003053419984,"output_tps":420.0651396006828},{"configuration":"CPU 64 GiB","request_rps":1.1911329977268466,"output_tps":854.7107294943331},{"configuration":"CPU 64 GiB + NVMe","request_rps":1.2844036448891545,"output_tps":924.0431793194367},{"configuration":"CPU 64 GiB + CephFS","request_rps":0.7112236354651219,"output_tps":496.0590787891447},{"configuration":"CPU 64 GiB + CephFS (tuned)","request_rps":1.2999884982351864,"output_tps":936.023627037797},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","request_rps":1.5183069594599292,"output_tps":1094.0362249264197}]},"mark":"bar","encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS","CPU 64 GiB + CephFS (tuned)","CPU 64 GiB + CephFS (tuned, 2)"]},"y":{"field":"request_rps","type":"quantitative","title":"Successful request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS","CPU 64 GiB + CephFS (tuned)","CPU 64 GiB + CephFS (tuned, 2)"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728","#9467bd","#e377c2"]}},"tooltip":[{"field":"configuration"},{"field":"request_rps"}]},"config":{"view":{"stroke":null}}}
```

Figure 2 — Output-token throughput.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 2 \u2014 Output-token throughput","data":{"values":[{"configuration":"No offload","request_rps":0.5780003053419984,"output_tps":420.0651396006828},{"configuration":"CPU 64 GiB","request_rps":1.1911329977268466,"output_tps":854.7107294943331},{"configuration":"CPU 64 GiB + NVMe","request_rps":1.2844036448891545,"output_tps":924.0431793194367},{"configuration":"CPU 64 GiB + CephFS","request_rps":0.7112236354651219,"output_tps":496.0590787891447},{"configuration":"CPU 64 GiB + CephFS (tuned)","request_rps":1.2999884982351864,"output_tps":936.023627037797},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","request_rps":1.5183069594599292,"output_tps":1094.0362249264197}]},"mark":"bar","encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS","CPU 64 GiB + CephFS (tuned)","CPU 64 GiB + CephFS (tuned, 2)"]},"y":{"field":"output_tps","type":"quantitative","title":"Output-token throughput (tokens/s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS","CPU 64 GiB + CephFS (tuned)","CPU 64 GiB + CephFS (tuned, 2)"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728","#9467bd","#e377c2"]}},"tooltip":[{"field":"configuration"},{"field":"output_tps"}]},"config":{"view":{"stroke":null}}}
```

Figure 3 — Mean TTFT and E2E latency.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 3 \u2014 Mean TTFT and E2E latency","data":{"values":[{"configuration":"No offload","metric":"TTFT mean","latency_s":32.9684395734475},{"configuration":"No offload","metric":"E2E mean","latency_s":50.51072524626206},{"configuration":"CPU 64 GiB","metric":"TTFT mean","latency_s":14.538710736265624},{"configuration":"CPU 64 GiB","metric":"E2E mean","latency_s":25.43860603640395},{"configuration":"CPU 64 GiB + NVMe","metric":"TTFT mean","latency_s":10.572176060322661},{"configuration":"CPU 64 GiB + NVMe","metric":"E2E mean","latency_s":20.287336753613314},{"configuration":"CPU 64 GiB + CephFS","metric":"TTFT mean","latency_s":20.579170687209835},{"configuration":"CPU 64 GiB + CephFS","metric":"E2E mean","latency_s":32.84110914882167},{"configuration":"CPU 64 GiB + CephFS (tuned)","metric":"TTFT mean","latency_s":10.494968588226829},{"configuration":"CPU 64 GiB + CephFS (tuned)","metric":"E2E mean","latency_s":20.12937747704105},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","metric":"TTFT mean","latency_s":0.5216268738018799},{"configuration":"CPU 64 GiB + CephFS (tuned, 2)","metric":"E2E mean","latency_s":13.111872515980115}]},"mark":"bar","encoding":{"x":{"field":"metric","type":"nominal","title":"Latency metric"},"xOffset":{"field":"configuration"},"y":{"field":"latency_s","type":"quantitative","title":"Mean latency (s)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS","CPU 64 GiB + CephFS (tuned)","CPU 64 GiB + CephFS (tuned, 2)"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728","#9467bd","#e377c2"]}},"tooltip":[{"field":"configuration"},{"field":"metric"},{"field":"latency_s"}]},"config":{"view":{"stroke":null}}}
```



## Full-granularity telemetry appendices

The full native-cadence scheduler and prompt-token time series are retained in the detailed matrix report and its comparison appendices:

- [Six-run CephFS tuning matrix](2026-07-24%20-%20Six-run%20CephFS%20tuning%20matrix.md)
- [Batch 3 Running and Waiting Requests comparison](2026-07-25%20-%20Standardized%20offload%20matrix%20batch%203/2026-07-25%20-%20Batch%203%20Running%20and%20Waiting%20Requests%20comparison.md)
- [Batch 3 Prompt Tokens by Source comparison — part 1](2026-07-25%20-%20Standardized%20offload%20matrix%20batch%203/2026-07-25%20-%20Batch%203%20Prompt%20Tokens%20by%20Source%20comparison%20%E2%80%94%20part%201.md)
- [Batch 3 Prompt Tokens by Source comparison — part 2](2026-07-25%20-%20Standardized%20offload%20matrix%20batch%203/2026-07-25%20-%20Batch%203%20Prompt%20Tokens%20by%20Source%20comparison%20%E2%80%94%20part%202.md)