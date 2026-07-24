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
| No offload | 0.363 | 235.1 | 11.469 | 79.430 | 19 |\n| CPU 64 GiB | 0.359 | 236.9 | 10.687 | 80.187 | 18 |\n| CPU 64 GiB + NVMe | 0.233 | 135.8 | 20.824 | 129.272 | 12 |\n| CPU 64 GiB + CephFS tuned | 0.257 | 151.4 | 15.950 | 114.149 | 12 |\n
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