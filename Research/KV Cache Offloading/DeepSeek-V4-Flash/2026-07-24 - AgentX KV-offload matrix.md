---
title: DeepSeek-V4-Flash AgentX KV-offload matrix
date: 2026-07-24
type: research-report
experiment: DeepSeek-V4-Flash KV Cache Offloading
model: DeepSeek-V4-Flash
---

# 2026-07-24 — DeepSeek-V4-Flash AgentX KV-offload matrix

## Executive summary

This four-run matrix compares HBM-only, 64 GiB CPU tier, 64 GiB CPU+NVMe, and 64 GiB CPU+CephFS tuned configurations for DeepSeek-V4-Flash. All runs use TP=8, eight H100s, U=0.85, 200 GiB `/dev/shm`, vLLM 0.23, max context 131,072, and AgentX concurrency 32. The CPU-only and HBM-only points are nearly identical; adding NVMe or tuned CephFS is worse in this run because both secondary-tier configurations reduce throughput and increase latency.

| Configuration | Req/s | Output tok/s | Mean TTFT (s) | Mean E2E (s) | Sessions |
|---|---:|---:|---:|---:|---:|
| No offload | 0.363 | 235.1 | 11.469 | 79.430 | 19 |
| CPU 64 GiB | 0.359 | 236.9 | 10.687 | 80.187 | 18 |
| CPU 64 GiB + NVMe | 0.233 | 135.8 | 20.824 | 129.272 | 12 |
| CPU 64 GiB + CephFS tuned | 0.257 | 151.4 | 15.950 | 114.149 | 12 |

Figure 1 compares the four headline outcomes. Provenance: native AIPerf profile exports.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 1 — DeepSeek offload outcome comparison","data":{"values":[{"variant":"No offload","rps":0.3634279458188405,"out":235.09286419017178,"ttft":11.469111727481872,"e2e":79.42987142817523,"sessions":19,"errors":[]},{"variant":"CPU 64 GiB","rps":0.35944963774249594,"out":236.90525628987322,"ttft":10.68735214260366,"e2e":80.18725255642377,"sessions":18,"errors":[]},{"variant":"CPU 64 GiB + NVMe","rps":0.23304043891587561,"out":135.79693069389467,"ttft":20.82369749263615,"e2e":129.27248758293427,"sessions":12,"errors":[]},{"variant":"CPU 64 GiB + CephFS tuned","rps":0.25746424483642655,"out":151.43762632006178,"ttft":15.950297512713375,"e2e":114.14902632882165,"sessions":12,"errors":[]}]},"mark":"bar","encoding":{"x":{"field":"variant","type":"nominal","title":"Configuration"},"y":{"field":"rps","type":"quantitative","title":"Request throughput (requests/s)","scale":{"zero":true}},"color":{"field":"variant","type":"nominal","title":"Configuration","scale":{"scheme":"category10"}},"tooltip":[{"field":"variant"},{"field":"rps","title":"Requests/s","format":".3f"},{"field":"out","title":"Output tokens/s","format":".1f"},{"field":"ttft","title":"Mean TTFT (s)","format":".3f"},{"field":"e2e","title":"Mean E2E (s)","format":".3f"}]},"config":{"view":{"stroke":null}}}
```

Figure 2 shows prompt-token source composition. Provenance: native vLLM prompt-source counters sampled every 15 seconds; these are integrated shares, not raw CephFS byte rates.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 2 — Prompt-token source shares","data":{"values":[{"variant":"No offload","source":"external kv transfer","share":0.0},{"variant":"No offload","source":"local cache hit","share":4.528269316675813},{"variant":"No offload","source":"local compute","share":95.47173068332417},{"variant":"CPU 64 GiB","source":"external kv transfer","share":0.0},{"variant":"CPU 64 GiB","source":"local cache hit","share":4.84710699520494},{"variant":"CPU 64 GiB","source":"local compute","share":95.15289300479506},{"variant":"CPU 64 GiB + NVMe","source":"external kv transfer","share":2.9811548187248382},{"variant":"CPU 64 GiB + NVMe","source":"local cache hit","share":5.488016611399811},{"variant":"CPU 64 GiB + NVMe","source":"local compute","share":91.53082856987534},{"variant":"CPU 64 GiB + CephFS tuned","source":"external kv transfer","share":2.701931450688317},{"variant":"CPU 64 GiB + CephFS tuned","source":"local cache hit","share":5.751286001797521},{"variant":"CPU 64 GiB + CephFS tuned","source":"local compute","share":91.54678254751416}]},"mark":"bar","encoding":{"x":{"field":"variant","type":"nominal","title":"Configuration"},"y":{"field":"share","type":"quantitative","title":"Prompt-token share (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"Source","scale":{"scheme":"category10"}},"tooltip":[{"field":"variant"},{"field":"source"},{"field":"share","title":"Share (%)","format":".2f"}]},"config":{"view":{"stroke":null}}}
```

## Main takeaways

- CPU64 does not improve this workload: 0.359 req/s versus 0.363 without offload, with nearly identical latency.
- NVMe is the weakest point: throughput falls to 0.233 req/s and mean E2E rises to 129.3 s.
- Tuned CephFS is better than NVMe (0.257 req/s, 114.1 s mean E2E) but still below CPU/HBM-only.
- External KV transfer appears only in NVMe and CephFS at roughly 2.7–3.0% of prompt-token sourcing; most prompts remain local compute. This workload/configuration does not create enough reusable offloaded traffic to amortize secondary-tier overhead.
- All runs ended under grace timeout with 29–30 incomplete sessions; errors were low (0–8), so the performance gap is primarily throughput/queueing, not a crash storm.

## Interpretation and next steps

The workload is not saturating HBM-only through useful prefix reuse: theoretical shared-prefix opportunity is high, but actual external sourcing is low. The secondary tiers add transfer and coordination overhead without enough readback demand. Before changing Ceph or NVMe hardware, repeat with a fixed-request replay that forces known prefix reuse and collect per-tier hit/miss, bytes, queue depth, and latency. Also compare concurrency 32/64/128 and inspect GPU KV occupancy; the current point may be below the offload benefit knee.

## Artifact registry

- No offload: run `de401ffc904a48d3b1e1dd43fccf31e4`
- CPU 64 GiB: run `a9aa2956879f4feea607e3dca61256b9`
- CPU 64 GiB + NVMe: run `efa0fc8253574aa58d7696429fa59eef`
- CPU 64 GiB + CephFS tuned: run `67135ac10f4a430aa1f6af878360d5f0`

## Deployment and workload audit

All four runs use `deepseek-ai/DeepSeek-V4-Flash`, vLLM `v0.23.0`, TP=8 on eight GPUs, `gpu-memory-utilization=0.85`, `max-model-len=131072`, FP8 KV cache, prefix caching, one replica, AgentX MVP, WEKA traces, seed 42, concurrency 32, 1,800-second profiling, and 200 GiB `/dev/shm`. The CPU and tiered cells use a 64 GiB CPU tier. The differences are the secondary tier: none, NVMe hostPath, or tuned CephFS (`n_read_threads=64`, `n_write_threads=32`, `PYTHONHASHSEED=0`).

| Run | Configuration | MLflow | Errors | Restarts | Grace timeout |
|---|---|---|---:|---:|---|
| `de401ffc904a48d3b1e1dd43fccf31e4` | HBM/no offload | [link](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/259/runs/de401ffc904a48d3b1e1dd43fccf31e4?workspace=benchflow) | 0 | 0 | yes |
| `a9aa2956879f4feea607e3dca61256b9` | CPU64 | [link](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/259/runs/a9aa2956879f4feea607e3dca61256b9?workspace=benchflow) | 0 | 0 | yes |
| `efa0fc8253574aa58d7696429fa59eef` | CPU64 + NVMe | [link](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/259/runs/efa0fc8253574aa58d7696429fa59eef?workspace=benchflow) | 7 | 0 | yes |
| `67135ac10f4a430aa1f6af878360d5f0` | CPU64 + tuned CephFS | [link](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/259/runs/67135ac10f4a430aa1f6af878360d5f0?workspace=benchflow) | 8 | 0 | yes |

## Scheduler behavior

Figure 3 shows the available scheduler running/waiting-request summary. Provenance: native `requests_running_waiting` gauges sampled every 15 seconds; source archives contain 288 samples per run.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":320,"title":"Figure 3 — Scheduler running and waiting requests","data":{"values":[{"variant":"No offload","state":"Running","mean":25.42,"max":53},{"variant":"No offload","state":"Waiting","mean":4.20,"max":29},{"variant":"CPU 64 GiB","state":"Running","mean":25.53,"max":46},{"variant":"CPU 64 GiB","state":"Waiting","mean":3.58,"max":26},{"variant":"CPU 64 GiB + NVMe","state":"Running","mean":26.06,"max":53},{"variant":"CPU 64 GiB + NVMe","state":"Waiting","mean":4.47,"max":31},{"variant":"CPU 64 GiB + CephFS tuned","state":"Running","mean":26.88,"max":51},{"variant":"CPU 64 GiB + CephFS tuned","state":"Waiting","mean":3.78,"max":30}]},"mark":"bar","encoding":{"x":{"field":"variant","type":"nominal","title":"Configuration"},"y":{"field":"mean","type":"quantitative","title":"Mean requests (count)","scale":{"zero":true}},"color":{"field":"state","type":"nominal","title":"Scheduler state","scale":{"scheme":"category10"}},"xOffset":{"field":"state"},"tooltip":[{"field":"variant"},{"field":"state"},{"field":"mean","title":"Mean requests","format":".2f"},{"field":"max","title":"Maximum requests","format":".0f"}]},"config":{"view":{"stroke":null}}}
```

Queue behavior is not the primary explanation for the NVMe/CephFS slowdown: mean waiting counts are similar (3.6–4.5), while latency and throughput diverge substantially. The secondary tier appears to add service overhead rather than causing a runaway queue.

## Resource and storage telemetry

The artifact set includes CPU, memory, KV occupancy, offload byte/time, NVMe filesystem, PVC, network, and Ceph health archives. Ceph pool bytes/IOPS, MDS request rate, and OSD latency series are empty, so CephFS read/write throughput cannot be attributed directly. NVMe device utilization is available in the raw archives and should be inspected in follow-up repetitions; this report does not infer saturation from request latency alone.

## Limitations and next experiment

All four runs end with the AIPerf grace timeout and retain 29–30 incomplete sessions, so the final tail is censored. The small error counts occur only in the tiered cells and do not explain the large throughput gap. The key missing evidence is per-tier hit/miss, bytes, queue depth, and latency—especially Ceph client/MDS/OSD counters and per-interface 200G NIC counters.

The next experiment should repeat each point at least three times, add a fixed-prefix replay that forces known reuse, and sweep concurrency 32/64/128. Accept a tier only when it shows measurable readback, no error increase, no queue explosion, and a repeatable improvement over the HBM/CPU control.

## Latency comparison

Figure 4 compares mean and p95 time-to-first-token (TTFT) and end-to-end latency. Provenance: native AIPerf request distributions; latency values are seconds and include the censored profiling tail only for completed requests.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":760,"height":340,"title":"Figure 4 — TTFT and end-to-end latency","data":{"values":[{"variant":"No offload","metric":"TTFT mean","latency_s":11.469},{"variant":"No offload","metric":"TTFT p95","latency_s":26.141},{"variant":"No offload","metric":"E2E mean","latency_s":79.430},{"variant":"No offload","metric":"E2E p95","latency_s":255.054},{"variant":"CPU 64 GiB","metric":"TTFT mean","latency_s":10.687},{"variant":"CPU 64 GiB","metric":"TTFT p95","latency_s":24.266},{"variant":"CPU 64 GiB","metric":"E2E mean","latency_s":80.187},{"variant":"CPU 64 GiB","metric":"E2E p95","latency_s":249.477},{"variant":"CPU 64 GiB + NVMe","metric":"TTFT mean","latency_s":20.824},{"variant":"CPU 64 GiB + NVMe","metric":"TTFT p95","latency_s":58.456},{"variant":"CPU 64 GiB + NVMe","metric":"E2E mean","latency_s":129.272},{"variant":"CPU 64 GiB + NVMe","metric":"E2E p95","latency_s":365.871},{"variant":"CPU 64 GiB + CephFS tuned","metric":"TTFT mean","latency_s":15.950},{"variant":"CPU 64 GiB + CephFS tuned","metric":"TTFT p95","latency_s":39.881},{"variant":"CPU 64 GiB + CephFS tuned","metric":"E2E mean","latency_s":114.149},{"variant":"CPU 64 GiB + CephFS tuned","metric":"E2E p95","latency_s":336.987}]},"mark":"bar","encoding":{"x":{"field":"variant","type":"nominal","title":"Configuration"},"y":{"field":"latency_s","type":"quantitative","title":"Latency (seconds)","scale":{"zero":true}},"color":{"field":"metric","type":"nominal","title":"Latency metric","scale":{"scheme":"category10"}},"xOffset":{"field":"metric"},"tooltip":[{"field":"variant"},{"field":"metric"},{"field":"latency_s","title":"Latency (s)","format":".3f"}]},"config":{"view":{"stroke":null}}}
```

CPU64 leaves latency essentially unchanged from HBM-only. NVMe has the worst TTFT and E2E tail, while tuned CephFS is better than NVMe but still materially slower than the no-offload and CPU64 controls.