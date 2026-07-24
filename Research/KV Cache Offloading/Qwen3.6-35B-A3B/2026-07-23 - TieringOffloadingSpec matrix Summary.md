---
title: "Qwen3.6 AgentX TieringOffloadingSpec matrix — Summary"
date: 2026-07-23
type: research-summary
derived-from: 00 - Report.md
---

# Qwen3.6 TieringOffloadingSpec Matrix — Key Highlights

## The headline result: a real offload staircase

Four configurations tested on Qwen3.6-35B-A3B (TP=2, H100s, concurrency 32, AgentX workload):

| Configuration | req/s | vs. No offload | Mean TTFT | Mean E2E | Completed sessions |
|---|---:|---|---:|---:|---:|
| **No offload** | 0.578 | — | 33.0 s | 50.5 s | 26 |
| **CPU 64 GiB** | 1.191 | +106% | 14.5 s | 25.4 s | 41 |
| **CPU 64 GiB + NVMe** | 1.284 | +122% | 10.6 s | 20.3 s | 44 |
| **CPU 64 GiB + CephFS** | 0.711 | +23% | 20.6 s | 32.8 s | 29 |

CPU 64 GiB alone is already a huge win. NVMe adds a clear but modest step on top. CephFS improves over no-offload but is significantly worse than CPU-only.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":700,"height":260,"title":"Request throughput by configuration","data":{"values":[{"variant":"No offload","rps":0.578},{"variant":"CPU 64 GiB","rps":1.191},{"variant":"CPU 64 GiB + NVMe","rps":1.284},{"variant":"CPU 64 GiB + CephFS","rps":0.711}]},"mark":{"type":"bar","cornerRadiusTopLeft":3,"cornerRadiusTopRight":3},"encoding":{"x":{"field":"variant","type":"nominal","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS"],"title":""},"y":{"field":"rps","type":"quantitative","title":"Successful requests/s","scale":{"zero":true}},"color":{"field":"variant","type":"nominal","scale":{"domain":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"variant","title":"Config"},{"field":"rps","title":"req/s","format":".3f"}]},"config":{"view":{"stroke":null}}}
```

## Latency: NVMe wins at both ends

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":700,"height":260,"title":"Mean TTFT and E2E latency","data":{"values":[{"variant":"No offload","metric":"TTFT","s":33.0},{"variant":"No offload","metric":"E2E","s":50.5},{"variant":"CPU 64 GiB","metric":"TTFT","s":14.5},{"variant":"CPU 64 GiB","metric":"E2E","s":25.4},{"variant":"CPU 64 GiB + NVMe","metric":"TTFT","s":10.6},{"variant":"CPU 64 GiB + NVMe","metric":"E2E","s":20.3},{"variant":"CPU 64 GiB + CephFS","metric":"TTFT","s":20.6},{"variant":"CPU 64 GiB + CephFS","metric":"E2E","s":32.8}]},"mark":{"type":"bar","cornerRadiusTopLeft":2,"cornerRadiusTopRight":2},"encoding":{"x":{"field":"metric","type":"nominal","title":""},"xOffset":{"field":"variant","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS"]},"y":{"field":"s","type":"quantitative","title":"Latency (s)","scale":{"zero":true}},"color":{"field":"variant","type":"nominal","scale":{"domain":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}}},"config":{"view":{"stroke":null}}}
```

NVMe reduces mean TTFT by 68% and mean E2E by 60% versus no-offload. Even over CPU64, NVMe shaves another 27% off TTFT and 20% off E2E. The advantage is consistent across all prompt-length bins (Figure 19 in the report) — it is not a short-prompt artifact.

## Why NVMe helps: less recomputation, not more filesystem hits

The counterintuitive finding: all three offload cells transfer a high fraction of tokens externally, but **NVMe recomputes fewer tokens** than CPU64, which recomputes far fewer than CephFS.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":700,"height":260,"title":"Prompt token source breakdown","data":{"values":[{"variant":"No offload","source":"Recompute","pct":97.2},{"variant":"No offload","source":"HBM hit","pct":2.8},{"variant":"CPU 64 GiB","source":"Recompute","pct":22.3},{"variant":"CPU 64 GiB","source":"HBM hit","pct":1.1},{"variant":"CPU 64 GiB","source":"External KV","pct":76.6},{"variant":"CPU 64 GiB + NVMe","source":"Recompute","pct":11.1},{"variant":"CPU 64 GiB + NVMe","source":"HBM hit","pct":0.2},{"variant":"CPU 64 GiB + NVMe","source":"External KV","pct":88.6},{"variant":"CPU 64 GiB + CephFS","source":"Recompute","pct":51.5},{"variant":"CPU 64 GiB + CephFS","source":"HBM hit","pct":9.1},{"variant":"CPU 64 GiB + CephFS","source":"External KV","pct":39.4}]},"mark":{"type":"bar","cornerRadiusTopLeft":2,"cornerRadiusTopRight":2},"encoding":{"x":{"field":"variant","type":"nominal","sort":["No offload","CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS"],"title":""},"y":{"field":"pct","type":"quantitative","stack":"zero","title":"Share (%)","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"","scale":{"domain":["Recompute","HBM hit","External KV"],"range":["#d62728","#2ca02c","#1f77b4"]}}},"config":{"view":{"stroke":null}}}
```

**NVMe retains more KV blocks in the tier hierarchy**, reducing recomputation from 22% (CPU64) to 11%. CephFS, despite some external transfer, still recomputes 51% of tokens — suggesting its secondary tier is barely effective.

## Transfer mechanics: CPU64 is store-heavy, NVMe is load-heavy

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":700,"height":260,"title":"Total KV transfer volume (GiB)","data":{"values":[{"variant":"CPU 64 GiB","direction":"GPU→CPU store","gib":1227},{"variant":"CPU 64 GiB","direction":"CPU→GPU load","gib":3243},{"variant":"CPU 64 GiB + NVMe","direction":"GPU→CPU store","gib":740},{"variant":"CPU 64 GiB + NVMe","direction":"CPU→GPU load","gib":4077},{"variant":"CPU 64 GiB + CephFS","direction":"GPU→CPU store","gib":341},{"variant":"CPU 64 GiB + CephFS","direction":"CPU→GPU load","gib":1121}]},"mark":{"type":"bar","cornerRadiusTopLeft":2,"cornerRadiusTopRight":2},"encoding":{"x":{"field":"variant","type":"nominal","sort":["CPU 64 GiB","CPU 64 GiB + NVMe","CPU 64 GiB + CephFS"],"title":""},"xOffset":{"field":"direction"},"y":{"field":"gib","type":"quantitative","title":"Transferred (GiB)","scale":{"zero":true}},"color":{"field":"direction","type":"nominal","title":"","scale":{"domain":["GPU→CPU store","CPU→GPU load"],"range":["#ff7f0e","#1f77b4"]}}},"config":{"view":{"stroke":null}}}
```

- **CPU64**: stores 1.2 TiB, loads 3.2 TiB (0.38:1 ratio) — heavy demotion churn, re-storing blocks that get evicted.
- **NVMe**: stores only 740 GiB, loads 4.1 TiB (0.18:1 ratio) — the secondary tier absorbs demotions, keeping CPU memory available and preserving more reuse.
- **CephFS**: low volume in both directions (341 GiB store, 1.1 TiB load) — not effectively engaging the tier.

The CPU→GPU copy speed is identical across all three (~20 ms/GiB), so **the gain is purely behavioral** (better hierarchy retention), not a faster copy primitive.

## CephFS: the bad news

CephFS is the clearest problem in this experiment:

1. **66,320+ `cannot store blocks` warnings** across 271 request IDs, with one request retried 4,082 times. This is a store-admission/backpressure failure.
2. **Zero evidence of actual Ceph read/write** — PVC footprint stayed at 0.00 GiB throughout; MDS/OSD/CSI metrics were unobservable due to missing RBAC permissions.
3. **51.5% recomputation** — nearly half of all prompt tokens are computed from scratch despite the tier being "configured."
4. **TTFT p95 of 139.5 s** vs. 21.7 s for NVMe — a 6.4x worse tail.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":700,"height":220,"title":"CephFS store-refusal warnings per minute","data":{"values":[{"minute":24,"warnings":3819},{"minute":25,"warnings":16438},{"minute":26,"warnings":12656},{"minute":27,"warnings":13178},{"minute":28,"warnings":9899},{"minute":29,"warnings":10330}]},"mark":{"type":"bar","color":"#d62728","cornerRadiusTopLeft":2,"cornerRadiusTopRight":2},"encoding":{"x":{"field":"minute","type":"ordinal","title":"Elapsed minute"},"y":{"field":"warnings","type":"quantitative","title":"Store-refusal count","scale":{"zero":true}}},"config":{"view":{"stroke":null}}}
```

The log is truncated (starts at ~24.8 min), so these numbers are **lower bounds**. The actual onset is unknown.

## NVMe device health: plenty of headroom

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","width":700,"height":220,"title":"NVMe device busy fraction over time","data":{"values":[{"min":0,"busy":6.2},{"min":2,"busy":9.7},{"min":5,"busy":11.1},{"min":8,"busy":12.5},{"min":10,"busy":9.2},{"min":12,"busy":11.0},{"min":14,"busy":9.4},{"min":16,"busy":7.8},{"min":18,"busy":7.0},{"min":20,"busy":7.3},{"min":22,"busy":8.0},{"min":24,"busy":9.4},{"min":26,"busy":9.3},{"min":28,"busy":11.2},{"min":30,"busy":12.3},{"min":32,"busy":14.1},{"min":34,"busy":17.2},{"min":36,"busy":18.9},{"min":38,"busy":17.7},{"min":40,"busy":17.4},{"min":42,"busy":16.7},{"min":44,"busy":16.4},{"min":46,"busy":16.1},{"min":48,"busy":17.2},{"min":50,"busy":16.7},{"min":52,"busy":17.2},{"min":54,"busy":16.1},{"min":56,"busy":18.0},{"min":58,"busy":19.1},{"min":60,"busy":20.4},{"min":62,"busy":21.0},{"min":64,"busy":23.0},{"min":66,"busy":21.9},{"min":68,"busy":21.1},{"min":70,"busy":20.7},{"min":72,"busy":22.2},{"min":74,"busy":21.7},{"min":76,"busy":20.6},{"min":78,"busy":19.8},{"min":80,"busy":19.4},{"min":82,"busy":19.1},{"min":84,"busy":19.5},{"min":86,"busy":18.0},{"min":88,"busy":18.6},{"min":90,"busy":18.3},{"min":92,"busy":17.7},{"min":94,"busy":18.4},{"min":96,"busy":18.6},{"min":98,"busy":19.7},{"min":100,"busy":19.7},{"min":102,"busy":19.5},{"min":104,"busy":19.4},{"min":106,"busy":18.7},{"min":108,"busy":17.3},{"min":110,"busy":19.4}]},"mark":{"type":"line","strokeWidth":1.7},"encoding":{"x":{"field":"min","type":"quantitative","title":"Elapsed (min)"},"y":{"field":"busy","type":"quantitative","title":"NVMe busy (%)","scale":{"domain":[0,100]}}},"config":{"view":{"stroke":null}}}
```

Mean busy: 15.8%, p95: 21.9%, max: 23.0%. Average traffic: 264 MiB/s reads + 218 MiB/s writes. **NVMe is far from saturated** — there is significant headroom for higher concurrency or larger tiers.

## What happened and what it means

**Good:**
- CPU 64 GiB offloading **just works** — +106% throughput, ~50% latency reduction, zero errors.
- NVMe as a second tier provides a genuine additional step (+7.8% throughput, -27% TTFT over CPU64) by reducing demotion churn and preserving more KV blocks.
- All offload cells had **zero restarts, zero CUDA OOMs, zero server tracebacks**.
- NVMe headroom is large (15.8% busy) — the system can scale further.

**Bad / concerning:**
- CephFS secondary tier is **broken in practice** — it recomputes 51% of tokens, floods with store-refusal warnings, and shows zero PVC footprint. Direct Ceph read/write proof is absent.
- The CephFS log truncation means the full failure history is lost. The 66K warnings are a lower bound.
- The no-offload baseline was run on a **different node** with different `/dev/shm` (1 GiB vs 200 GiB). The CPU64 vs NVMe vs CephFS comparison is clean; the baseline comparison is directional.
- Native vLLM prefix-cache hit rate was 0% across all configurations — the prefix cache is not engaging. The gains come entirely from the offload tiers, not from prefix reuse.
- The EPP auto-created an approximate prefix producer, but with one backend replica this has no measurable effect.

**What probably happened with CephFS:**
The TieringOffloadingSpec's secondary tier likely cannot keep up with CephFS write latency. KV blocks are demoted from GPU → CPU → CephFS, but CephFS writes are slow enough that the CPU tier fills up and starts refusing new stores. This creates a cascade: refused stores force recomputation instead, which loads the GPU with compute rather than serving cached KV, which increases contention, which triggers more evictions. The 4,082-retry request is likely a symptom of this feedback loop. The 128 GiB `/dev/shm` (vs 200 GiB for NVMe) and the `PYTHONHASHSEED=0` difference are confounds that need to be eliminated.

## Next steps (from the report)

1. **Isolate TieringOffloadingSpec from the implementation confound**: run CPU 64 GiB with TieringOffloadingSpec but **no secondary tier**, same 200 GiB `/dev/shm`. This separates the mmap/shared-memory effect from actual NVMe value.
2. **Fix CephFS observability**: grant `pods/log` access to MDS/OSD/CSI, make Prometheus queries return series, retain complete model logs.
3. **Run at least 3 paired repetitions** of each cell.
4. **Do not change U or concurrency yet** — U=0.55 at C=32 already shows strong offload benefit. Sweep later.