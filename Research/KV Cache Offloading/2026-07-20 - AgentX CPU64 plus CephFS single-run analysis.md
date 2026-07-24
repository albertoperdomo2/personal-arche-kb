---
title: AgentX CPU64 plus CephFS single-run analysis
date: 2026-07-20
type: research-report
model: Multiple/unspecified
topic: KV Cache Offloading
---

# 2026-07-20 — AgentX CPU64 + CephFS single-run mechanism audit

- MLflow run: [d2c57cdc56084c4193d71bfb8e1cfdfb](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/d2c57cdc56084c4193d71bfb8e1cfdfb?workspace=benchflow)
- Disposition: **CephFS writes proven; CephFS read hits not proven; reject as a clean v0.24 performance datapoint**
- Artifact root: /private/tmp/kv-cache-experiments/cephfs-run-2026-07-20
- Profiling window: 30 minutes, concurrency 128
- Time-series resolution: raw 15-second Prometheus samples; raw one-second warning counts
- Model: Qwen/Qwen3.6-35B-A3B, one replica, TP=2

## Executive verdict

CephFS was mounted and it did receive KV-cache data. The strongest proof is the 3 TiB rook-cephfs PVC mounted at /mnt/kv_cache and its retained footprint increasing from 0 to 113,648,861,184 bytes, or 105.84 GiB. This is far too large to be only a configuration file.

I cannot honestly claim that CephFS produced read hits. Every direct Ceph pool/MDS/OSD metric and both container filesystem I/O metrics returned **zero Prometheus series**, not zero-valued series. The available vLLM 0.23 connector counters stop at CPU↔GPU and do not label CPU hits separately from filesystem promotions. The observed 226.70 GiB CPU→GPU traffic and 6.07% average external prompt-token share prove connector reuse, but either can be satisfied entirely by the 64 GiB CPU tier.

The tier was also unhealthy under this load. The retained portion of the model log contains 107,670 “cannot store blocks” warnings for 398 requests, and that retained log covers only the final roughly nine minutes. The exact immediate cause in vLLM is CPU primary-tier store admission failure: there were not enough free or evictable CPU blocks. In the tiering implementation, CPU blocks are pinned while asynchronous CPU→CephFS cascades finish, so a slow/backlogged filesystem pipeline can exhaust evictable CPU capacity.

Finally, this run used **vllm/vllm-openai:v0.23.0**, image digest sha256:3a1e7f5904e1a1192a02aa0086ceaffc33985d7044c7bb25b3a43d61bdbe3ac0. It is not a v0.24 run. Version 0.23 performs each filesystem existence check synchronously with os.path.exists on the scheduler thread. Version 0.24 moves filesystem lookups to a background batched worker and includes multiple KV-offload fixes. Do not use this result to judge v0.24 CephFS performance.

## Configuration reconstructed from artifacts

| Item | Observed value |
|---|---|
| vLLM image | vllm/vllm-openai:v0.23.0 |
| Replica / tensor parallelism | 1 / TP=2 |
| GPU memory utilization | 0.9 |
| GPU KV blocks | 3,240 |
| Maximum model length | 131,072 tokens |
| CPU primary tier | 68,719,476,736 bytes = 64 GiB |
| Secondary tier | filesystem at /mnt/kv_cache |
| PVC | vllm-kv-cache, 3 TiB, RWX, rook-cephfs |
| Filesystem I/O workers | defaults: 16 read-priority + 16 write-priority |
| Hash stability | PYTHONHASHSEED=0 |
| Mamba cache mode | align, confirmed by cache_config_info |
| Workload | AgentX MVP, concurrency 128, duration 1,800 s, seed 42 |
| Pod health | Ready, zero restarts |

## What “worked” means

The v0.23 tiering path is:

$$
\text{store: GPU} \rightarrow \text{CPU primary} \rightarrow \text{CephFS secondary}
$$

and, only after a CPU miss and filesystem hit:

$$
\text{load: CephFS} \rightarrow \text{CPU promotion} \rightarrow \text{GPU}
$$

A successful GPU→CPU store is cascaded to every secondary tier. During the filesystem operation, vLLM increments the CPU block reference count. The warning occurs when:

$$
N_{\mathrm{free}} + N_{\mathrm{evictable}} < N_{\mathrm{new\ store\ blocks}}
$$

The code then returns no store allocation and emits “cannot store blocks”. Because the native CPU↔GPU copy cost averaged only 0.0247 s/GiB for loads and 0.0287 s/GiB for stores—about 40.4 and 34.8 GiB/s respectively—the sustained pinning is much more consistent with the secondary filesystem path than the CPU↔GPU copy itself. Direct tier queue-depth metrics were not captured, so this last attribution remains a strong inference rather than a directly measured Ceph latency result.

## CephFS write evidence

Figure 1 shows the only storage-specific time series that returned data at the archive’s native 15-second resolution. Provenance: kubelet_volume_stats_used_bytes for PVC vllm-kv-cache, sampled by Benchflow. The single delayed step is a kubelet volume-stat observation, not an instantaneous 105.84 GiB write burst; use it as retained-footprint evidence, not device throughput.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":700,"height":300,"title":"Figure 1 — CephFS-backed PVC retained footprint (15 s samples)","data":{"values":[{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":0},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375},{"v":105.84375}]},"mark":{"type":"line","point":false,"interpolate":"step-after","strokeWidth":2.5},"encoding":{"x":{"field":"t","type":"quantitative","title":"Elapsed profiling time (s)","scale":{"zero":true}},"y":{"field":"v","type":"quantitative","title":"PVC retained data (GiB)","scale":{"zero":true}},"color":{"value":"#1f77b4"},"tooltip":[{"field":"t","type":"quantitative","title":"Elapsed time (s)"},{"field":"v","type":"quantitative","title":"Retained data (GiB)","format":".2f"}]},"transform":[{"window":[{"op":"row_number","as":"i"}]},{"calculate":"3 + 15 * (datum.i - 1)","as":"t"}]}
```

The net footprint change over the 30-minute profile was:

$$
\Delta S_{\mathrm{PVC}} = 113{,}648{,}861{,}184\ \mathrm{bytes} = 105.84\ \mathrm{GiB}
$$

If spread over the full window it is 60.2 MiB/s of net retained growth. That is not physical write bandwidth: duplicate blocks are skipped, overwritten temporary files are not retained, and the kubelet statistic updated coarsely.

Figure 2 explicitly plots telemetry availability rather than inventing zero I/O. Provenance: archived Prometheus result-set sizes for Ceph pool throughput/IOPS, OSD latency, MDS requests, and container filesystem throughput/IOPS.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":700,"height":300,"title":"Figure 2 — CephFS read/write telemetry availability","data":{"values":[{"metric":"Pool read/write throughput","series":0},{"metric":"Pool read/write IOPS","series":0},{"metric":"OSD read/write latency","series":0},{"metric":"MDS client requests","series":0},{"metric":"Container FS throughput","series":0},{"metric":"Container FS IOPS","series":0}]},"layer":[{"mark":{"type":"bar"},"encoding":{"x":{"field":"metric","type":"nominal","title":"Requested telemetry quantity","sort":null,"axis":{"labelAngle":-25}},"y":{"field":"series","type":"quantitative","title":"Returned Prometheus series (count)","scale":{"domain":[0,1],"zero":true}},"color":{"field":"metric","type":"nominal","scale":{"scheme":"category10"},"legend":null}}},{"mark":{"type":"point","filled":true,"size":110},"encoding":{"x":{"field":"metric","type":"nominal","sort":null},"y":{"field":"series","type":"quantitative"},"color":{"field":"metric","type":"nominal","scale":{"scheme":"category10"},"legend":null},"tooltip":[{"field":"metric","type":"nominal","title":"Metric"},{"field":"series","type":"quantitative","title":"Returned series"}]}},{"mark":{"type":"text","dy":-12},"encoding":{"x":{"field":"metric","type":"nominal","sort":null},"y":{"field":"series","type":"quantitative"},"text":{"value":"unavailable"}}}]}
```

The zeros in Figure 2 mean **no returned series**. Consequently, there is no valid CephFS read-throughput, write-throughput, read-IOPS, write-IOPS, or latency plot for this run. The report profile’s pool regex was broad enough to match common CephFS data-pool names, while Ceph health and MDS queries were also empty; this points more strongly to Ceph exporter visibility/access than to a narrow pool-name mismatch.

## Connector activity

Figure 3 plots native connector traffic. Provenance: archived vllm:kv_offload_total_bytes rates, aggregated by transfer_type and pod at the archive’s native 15-second resolution. These are GPU↔CPU transfers; they are **not** CephFS read/write counters.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":700,"height":300,"title":"Figure 3 — Native connector transfer throughput (15 s samples)","data":{"values":[{"a":4.31,"b":493.97},{"a":23.53,"b":373.18},{"a":43.68,"b":276.77},{"a":74.59,"b":278.1},{"a":108.61,"b":280.73},{"a":123.36,"b":281.47},{"a":143.81,"b":281.3},{"a":159.48,"b":280.9},{"a":171.42,"b":279.74},{"a":179.51,"b":279.59},{"a":198.79,"b":279.36},{"a":207.76,"b":280.66},{"a":218.5,"b":279.91},{"a":238.77,"b":282.28},{"a":264.91,"b":283.5},{"a":284.8,"b":279.19},{"a":312.09,"b":280.99},{"a":316.79,"b":284.78},{"a":328.93,"b":288.49},{"a":326.65,"b":285.54},{"a":311.01,"b":279.86},{"a":296.04,"b":277.48},{"a":279.55,"b":273.57},{"a":275.29,"b":276.44},{"a":271.4,"b":276.79},{"a":269.72,"b":275.96},{"a":268.05,"b":277.73},{"a":258.59,"b":276.47},{"a":243.03,"b":274.29},{"a":231.43,"b":274.01},{"a":225.79,"b":276.08},{"a":212.58,"b":277.13},{"a":202.66,"b":277.79},{"a":185.57,"b":282.36},{"a":165.42,"b":284.33},{"a":165.04,"b":283.12},{"a":162.45,"b":282.45},{"a":157.26,"b":284.95},{"a":155.35,"b":292.94},{"a":144.06,"b":305},{"a":133.6,"b":318.12},{"a":127.87,"b":317.34},{"a":118.86,"b":315.41},{"a":119.16,"b":316.64},{"a":121.91,"b":318.09},{"a":127.94,"b":323.34},{"a":133.97,"b":325.32},{"a":139.39,"b":326.52},{"a":146.34,"b":327.49},{"a":143.21,"b":330.56},{"a":137.03,"b":337.75},{"a":138.4,"b":337.91},{"a":141.3,"b":337.76},{"a":137.79,"b":343.19},{"a":134.28,"b":349.22},{"a":133.21,"b":351.33},{"a":132.14,"b":348.17},{"a":135.59,"b":344.83},{"a":139.49,"b":342.54},{"a":138.35,"b":344.9},{"a":135.91,"b":349.94},{"a":124.23,"b":359.87},{"a":113.39,"b":370.79},{"a":113.62,"b":374.72},{"a":113.85,"b":382.7},{"a":115.68,"b":392.69},{"a":114.62,"b":398.79},{"a":112.94,"b":395.13},{"a":114.55,"b":387.96},{"a":115.55,"b":398.14},{"a":116.16,"b":404.43},{"a":120.29,"b":411.68},{"a":124.41,"b":423.44},{"a":126.01,"b":449.76},{"a":128.37,"b":469.97},{"a":121.95,"b":479.46},{"a":113.48,"b":493.55},{"a":109.81,"b":514.56},{"a":105.62,"b":531.27},{"a":102.8,"b":527.69},{"a":101.81,"b":528.3},{"a":93.56,"b":547.66},{"a":85.32,"b":563.83},{"a":78.98,"b":575.07},{"a":76,"b":598.06},{"a":72.03,"b":616.08},{"a":62.95,"b":630.12},{"a":57.15,"b":644.41},{"a":53.1,"b":663.37},{"a":46.53,"b":694.34},{"a":39.97,"b":714.23},{"a":39.36,"b":714.88},{"a":37.99,"b":722.34},{"a":37.23,"b":732.63},{"a":37.23,"b":738.49},{"a":41.5,"b":732.72},{"a":45.77,"b":733.44},{"a":45.32,"b":762.92},{"a":44.86,"b":792.03},{"a":44.86,"b":786.07},{"a":50.13,"b":775.14},{"a":54.56,"b":772.3},{"a":50.36,"b":758.38},{"a":47,"b":752.8},{"a":48.37,"b":751.35},{"a":45.47,"b":745.33},{"a":41.19,"b":736.71},{"a":44.56,"b":725.67},{"a":47.92,"b":725.65},{"a":46.16,"b":713.17},{"a":44.41,"b":693.97},{"a":42.2,"b":697.08},{"a":39.99,"b":700.72},{"a":34.8,"b":706.76},{"a":33.19,"b":712.64},{"a":35.71,"b":700.18},{"a":34.64,"b":681.6},{"a":35.79,"b":678.38},{"a":31.66,"b":690.73},{"a":25.25,"b":698.18}]},"mark":{"type":"line","point":false,"strokeWidth":1.8},"encoding":{"x":{"field":"t","type":"quantitative","title":"Elapsed profiling time (s)","scale":{"zero":true}},"y":{"field":"v","type":"quantitative","title":"Transfer throughput (MiB/s)","scale":{"zero":true}},"color":{"field":"direction","type":"nominal","title":"Transfer direction","scale":{"scheme":"category10"}},"tooltip":[{"field":"t","type":"quantitative","title":"Elapsed time (s)"},{"field":"direction","type":"nominal","title":"Direction"},{"field":"v","type":"quantitative","title":"Throughput (MiB/s)","format":".1f"}]},"transform":[{"window":[{"op":"row_number","as":"i"}]},{"calculate":"3 + 15 * (datum.i - 1)","as":"t"},{"fold":["a","b"],"as":["k","v"]},{"calculate":"datum.k === 'a' ? 'CPU → GPU' : 'GPU → CPU'","as":"direction"}]}
```

Across the profile, GPU→CPU stored 857.37 GiB at 487.75 MiB/s on average, while CPU→GPU loaded 226.70 GiB at 128.97 MiB/s. Late in the run, stores rose to roughly 700–767 MiB/s while loads fell to roughly 34–48 MiB/s. The store stream therefore became increasingly dominant.

Comparing logical GPU→CPU traffic with retained PVC growth gives about 8.1× more logical store traffic than net filesystem growth:

$$
\frac{487.75\ \mathrm{MiB/s}}{60.21\ \mathrm{MiB/s}} \approx 8.1
$$

This ratio is not a Ceph bandwidth efficiency measurement because repeated block hashes are intentionally skipped and PVC usage is coarse. It does show that the connector generated far more store work than the net unique footprint.

Figure 4 shows where server-side prompt tokens came from. Provenance: archived vLLM prompt-token source counters kept at the archive’s native 15-second rate resolution and normalized at each timestamp.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":700,"height":300,"title":"Figure 4 — Prompt-token source share (15 s samples)","data":{"values":[{"a":207.73,"b":1083.18,"c":59699.02},{"a":929.82,"b":1083.18,"c":56417.47},{"a":1772.78,"b":1083.18,"c":53927.53},{"a":3145.37,"b":1083.18,"c":53780.83},{"a":4669.73,"b":1083.18,"c":54082.26},{"a":5116.29,"b":1059.19,"c":55231.78},{"a":5971.71,"b":985.6,"c":56487.86},{"a":6847.03,"b":1138.13,"c":57817.93},{"a":7351.34,"b":1290.67,"c":59019.44},{"a":7686.27,"b":1016.89,"c":59383.01},{"a":8311.93,"b":743.11,"c":59546.51},{"a":8721.36,"b":743.11,"c":60311.29},{"a":9232.19,"b":743.11,"c":60790.24},{"a":10283.7,"b":743.11,"c":63830.53},{"a":11388.47,"b":743.11,"c":67621.95},{"a":12320.18,"b":1071.64,"c":71653.13},{"a":13410.38,"b":1400.18,"c":75888.09},{"a":13647.99,"b":1400.18,"c":81045.56},{"a":14003.85,"b":1400.18,"c":86308.19},{"a":14068.27,"b":1400.18,"c":88168.56},{"a":13516.8,"b":1400.18,"c":89508.32},{"a":12769.78,"b":1400.18,"c":87859.73},{"a":12011.02,"b":1400.18,"c":85643.81},{"a":12073.6,"b":1400.18,"c":84741.7},{"a":11952.36,"b":1400.18,"c":83978.1},{"a":11733.33,"b":1028.62,"c":84399.69},{"a":11709.87,"b":657.07,"c":84715.78},{"a":11361.78,"b":657.07,"c":84226.35},{"a":10825.96,"b":657.07,"c":84062.72},{"a":10348.8,"b":1067.73,"c":83416.13},{"a":9918.58,"b":1478.4,"c":82729.64},{"a":9230.22,"b":1478.4,"c":82597.08},{"a":8702.22,"b":1478.4,"c":82539.63},{"a":7955.2,"b":1423.64,"c":82710.53},{"a":7243.38,"b":1368.89,"c":82524.47},{"a":7208.18,"b":1368.89,"c":81966.36},{"a":7200.36,"b":1368.89,"c":81478.91},{"a":7008.71,"b":1368.89,"c":81199.71},{"a":6789.69,"b":1368.89,"c":81636.2},{"a":6246.04,"b":1368.89,"c":82997.47},{"a":5714.13,"b":1368.89,"c":84177.44},{"a":5444.27,"b":1368.89,"c":84313.89},{"a":5041.42,"b":1368.89,"c":84512.29},{"a":5045.33,"b":1642.67,"c":84606.07},{"a":5170.49,"b":1916.44,"c":84266.63},{"a":5436.44,"b":1916.44,"c":84300.03},{"a":5710.22,"b":1916.44,"c":84351.61},{"a":5984,"b":1505.78,"c":84614.2},{"a":6261.69,"b":1095.11,"c":84904.75},{"a":6109.16,"b":1290.67,"c":85111.56},{"a":5972.27,"b":1486.22,"c":85594.13},{"a":6034.84,"b":1212.44,"c":86979.93},{"a":6070.04,"b":938.67,"c":88502.44},{"a":5960.53,"b":938.67,"c":89474.14},{"a":5815.82,"b":938.67,"c":90311.67},{"a":5725.87,"b":938.67,"c":90389.14},{"a":5671.11,"b":938.67,"c":89792.17},{"a":5909.69,"b":1102.93,"c":87712.83},{"a":6175.64,"b":1267.2,"c":86016.38},{"a":6136.53,"b":1623.11,"c":85188.59},{"a":5960.53,"b":1979.02,"c":84392.67},{"a":5397.33,"b":1705.24,"c":84999.09},{"a":4943.64,"b":1431.47,"c":85885.75},{"a":4955.38,"b":1431.47,"c":87575.95},{"a":4978.84,"b":1431.47,"c":89228.2},{"a":5096.18,"b":1431.47,"c":90384.71},{"a":5190.04,"b":1431.47,"c":91481.26},{"a":5166.58,"b":1235.91,"c":91543.57},{"a":5182.22,"b":1040.36,"c":91285.16},{"a":5240.89,"b":1040.36,"c":90578.99},{"a":5272.18,"b":1040.36,"c":90138.78},{"a":5471.64,"b":1040.36,"c":89052.97},{"a":5671.11,"b":1040.36,"c":87561.9},{"a":5729.78,"b":1040.36,"c":87277.36},{"a":5788.44,"b":1040.36,"c":87178.33},{"a":5448.18,"b":876.09,"c":88252.46},{"a":5053.16,"b":711.82,"c":88922.64},{"a":4888.89,"b":355.91,"c":89700.15},{"a":4779.38,"b":0,"c":91224.46},{"a":4669.87,"b":0,"c":90783.95},{"a":4560.36,"b":0,"c":90182.87},{"a":4216.18,"b":383.29,"c":88928.79},{"a":3852.44,"b":766.58,"c":87480.83},{"a":3508.27,"b":1122.49,"c":86949.1},{"a":3344,"b":1478.4,"c":86685.7},{"a":3152.36,"b":1834.31,"c":85643.14},{"a":2745.6,"b":2190.22,"c":84682.37},{"a":2483.56,"b":2190.22,"c":85172.3},{"a":2276.27,"b":2374.04,"c":85171.25},{"a":1963.38,"b":2557.87,"c":86157.49},{"a":1650.49,"b":2557.87,"c":87463.7},{"a":1630.93,"b":2749.51,"c":87240.15},{"a":1611.38,"b":2941.16,"c":87278.99},{"a":1584,"b":2941.16,"c":88217.91},{"a":1556.62,"b":2941.16,"c":89202.9},{"a":1752.18,"b":2941.16,"c":89222.67},{"a":1947.73,"b":2941.16,"c":88526.75},{"a":1936,"b":2941.16,"c":88859.17},{"a":1924.27,"b":2941.16,"c":89626.03},{"a":1924.27,"b":2557.87,"c":90205.22},{"a":1924.27,"b":2174.58,"c":90619.2},{"a":2127.64,"b":1818.67,"c":90559.47},{"a":2170.67,"b":1462.76,"c":90387.78},{"a":2010.31,"b":1298.49,"c":92043.41},{"a":2010.31,"b":1134.22,"c":93454.51},{"a":1873.42,"b":1134.22,"c":93096.68},{"a":1736.53,"b":950.4,"c":92959.96},{"a":1810.84,"b":1067.73,"c":92587.86},{"a":1982.93,"b":1368.89,"c":92622.4},{"a":2014.22,"b":1177.24,"c":93629.67},{"a":1947.73,"b":985.6,"c":93984.7},{"a":1857.78,"b":985.6,"c":92664.09},{"a":1767.82,"b":985.6,"c":92045.18},{"a":1494.04,"b":985.6,"c":92810.58},{"a":1306.31,"b":985.6,"c":93218.63},{"a":1454.93,"b":1267.2,"c":92357.01},{"a":1517.51,"b":1548.8,"c":91355.93},{"a":1564.44,"b":1548.8,"c":90905.7},{"a":1611.38,"b":1548.8,"c":90767.52},{"a":1329.78,"b":1548.8,"c":91098.01}]},"transform":[{"window":[{"op":"row_number","as":"i"}]},{"calculate":"3 + 15 * (datum.i - 1)","as":"t"},{"fold":["a","b","c"],"as":["k","rate"]},{"calculate":"datum.k === 'a' ? 'External KV transfer' : datum.k === 'b' ? 'Local GPU cache' : 'Local compute'","as":"source"},{"joinaggregate":[{"op":"sum","field":"rate","as":"total_rate"}],"groupby":["t"]},{"calculate":"100 * datum.rate / datum.total_rate","as":"share"}],"mark":{"type":"line","point":false,"strokeWidth":1.8},"encoding":{"x":{"field":"t","type":"quantitative","title":"Elapsed profiling time (s)","scale":{"zero":true}},"y":{"field":"share","type":"quantitative","title":"Prompt tokens by source (%)","scale":{"domain":[0,100],"zero":true}},"color":{"field":"source","type":"nominal","title":"Prompt-token source","scale":{"scheme":"category10"}},"tooltip":[{"field":"t","type":"quantitative","title":"Elapsed time (s)"},{"field":"source","type":"nominal","title":"Source"},{"field":"share","type":"quantitative","title":"Share (%)","format":".2f"},{"field":"rate","type":"quantitative","title":"Rate (tokens/s)","format":".0f"}]}}
```

Over the complete window, prompt-token shares averaged 92.42% local compute, 6.07% external KV transfer, and 1.51% local GPU-cache hit. The external share peaks early and falls to about 1.6% at the end. The model log’s cumulative external prefix-cache hit rate similarly falls from 7.1% to 3.6%. This proves some offload-connector reuse, but not which external tier served it. It also shows that the 94.82% trace-theoretical reuse did not translate into actual cache reuse.

## Capacity pressure and failure mechanism

Figure 5 plots GPU KV-cache occupancy. Provenance: archived vLLM kv_cache_usage_perc, with the active kserve_vllm series selected and plotted at the archive’s native 15-second resolution. The red rule marks 90%.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":700,"height":300,"title":"Figure 5 — GPU KV-cache occupancy (15 s samples)","data":{"values":[{"v":8.799},{"v":8.799},{"v":40.445},{"v":40.445},{"v":77.987},{"v":77.987},{"v":96.45},{"v":96.45},{"v":95.369},{"v":95.369},{"v":97.16},{"v":97.16},{"v":97.993},{"v":97.993},{"v":97.808},{"v":97.808},{"v":97.654},{"v":97.654},{"v":98.487},{"v":98.487},{"v":99.568},{"v":99.568},{"v":95.925},{"v":95.925},{"v":97.993},{"v":97.993},{"v":97.129},{"v":97.129},{"v":99.568},{"v":99.568},{"v":99.784},{"v":99.784},{"v":98.024},{"v":98.024},{"v":98.333},{"v":98.333},{"v":93.208},{"v":93.208},{"v":94.906},{"v":94.906},{"v":94.103},{"v":94.103},{"v":97.592},{"v":97.592},{"v":96.357},{"v":96.357},{"v":96.82},{"v":96.82},{"v":97.87},{"v":97.87},{"v":91.386},{"v":91.386},{"v":98.919},{"v":98.919},{"v":99.352},{"v":99.352},{"v":98.487},{"v":98.487},{"v":98.425},{"v":98.425},{"v":95.585},{"v":95.585},{"v":92.652},{"v":92.652},{"v":98.271},{"v":98.271},{"v":96.635},{"v":96.635},{"v":97.098},{"v":97.098},{"v":96.573},{"v":96.573},{"v":96.264},{"v":96.264},{"v":97.561},{"v":97.561},{"v":98.364},{"v":98.364},{"v":99.105},{"v":99.105},{"v":95.276},{"v":95.276},{"v":98.981},{"v":98.981},{"v":98.672},{"v":98.672},{"v":95.122},{"v":95.122},{"v":96.326},{"v":96.326},{"v":98.549},{"v":98.549},{"v":92.189},{"v":92.189},{"v":98.055},{"v":98.055},{"v":94.412},{"v":94.412},{"v":91.757},{"v":91.757},{"v":98.425},{"v":98.425},{"v":95.739},{"v":95.739},{"v":96.635},{"v":96.635},{"v":95.369},{"v":95.369},{"v":94.412},{"v":94.412},{"v":97.715},{"v":97.715},{"v":97.931},{"v":97.931},{"v":99.228},{"v":99.228},{"v":96.882},{"v":96.882},{"v":93.98},{"v":93.98}]},"layer":[{"mark":{"type":"line","point":false,"strokeWidth":1.8},"encoding":{"x":{"field":"t","type":"quantitative","title":"Elapsed profiling time (s)","scale":{"zero":true}},"y":{"field":"v","type":"quantitative","title":"GPU KV-cache utilization (%)","scale":{"domain":[0,100],"zero":true}},"color":{"value":"#1f77b4"},"tooltip":[{"field":"t","type":"quantitative","title":"Elapsed time (s)"},{"field":"v","type":"quantitative","title":"GPU KV utilization (%)","format":".1f"}]}},{"mark":{"type":"rule","strokeDash":[6,4],"color":"#d62728"},"encoding":{"y":{"datum":90,"type":"quantitative"}}}],"transform":[{"window":[{"op":"row_number","as":"i"}]},{"calculate":"3 + 15 * (datum.i - 1)","as":"t"}]}
```

GPU KV usage averaged 94.09%, had a p50 of 97.11%, and peaked at 99.78%. This confirms that 0.9 GPU memory utilization did not leave the server underfilled; the cache stayed near full after ramp-up.

Figure 6 shows the scheduler backlog. Provenance: archived active vLLM running and waiting-request gauges at the native 15-second resolution.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":700,"height":300,"title":"Figure 6 — Model scheduler request pressure (15 s samples)","data":{"values":[{"a":3,"b":0},{"a":19,"b":62},{"a":19,"b":62},{"a":37,"b":93},{"a":37,"b":93},{"a":44,"b":80},{"a":44,"b":80},{"a":44,"b":78},{"a":44,"b":78},{"a":43,"b":82},{"a":43,"b":82},{"a":42,"b":83},{"a":42,"b":83},{"a":44,"b":82},{"a":44,"b":82},{"a":49,"b":81},{"a":49,"b":81},{"a":46,"b":87},{"a":46,"b":87},{"a":48,"b":91},{"a":48,"b":91},{"a":50,"b":92},{"a":50,"b":92},{"a":53,"b":95},{"a":53,"b":95},{"a":57,"b":79},{"a":57,"b":79},{"a":50,"b":82},{"a":50,"b":82},{"a":49,"b":77},{"a":49,"b":77},{"a":46,"b":84},{"a":46,"b":84},{"a":42,"b":88},{"a":42,"b":88},{"a":46,"b":82},{"a":46,"b":82},{"a":42,"b":81},{"a":42,"b":81},{"a":47,"b":78},{"a":47,"b":78},{"a":45,"b":84},{"a":45,"b":84},{"a":43,"b":85},{"a":43,"b":85},{"a":42,"b":85},{"a":42,"b":85},{"a":42,"b":86},{"a":42,"b":86},{"a":42,"b":81},{"a":42,"b":81},{"a":47,"b":81},{"a":47,"b":81},{"a":47,"b":79},{"a":47,"b":79},{"a":43,"b":79},{"a":43,"b":79},{"a":44,"b":84},{"a":44,"b":84},{"a":41,"b":82},{"a":41,"b":82},{"a":42,"b":85},{"a":42,"b":85},{"a":44,"b":81},{"a":44,"b":81},{"a":45,"b":78},{"a":45,"b":78},{"a":44,"b":82},{"a":44,"b":82},{"a":40,"b":84},{"a":40,"b":84},{"a":41,"b":88},{"a":41,"b":88},{"a":42,"b":91},{"a":42,"b":91},{"a":41,"b":90},{"a":41,"b":90},{"a":40,"b":90},{"a":40,"b":90},{"a":42,"b":88},{"a":42,"b":88},{"a":45,"b":87},{"a":45,"b":87},{"a":44,"b":86},{"a":44,"b":86},{"a":42,"b":84},{"a":42,"b":84},{"a":44,"b":86},{"a":44,"b":86},{"a":47,"b":89},{"a":47,"b":89},{"a":43,"b":89},{"a":43,"b":89},{"a":47,"b":81},{"a":47,"b":81},{"a":44,"b":84},{"a":44,"b":84},{"a":43,"b":84},{"a":43,"b":84},{"a":44,"b":85},{"a":44,"b":85},{"a":43,"b":90},{"a":43,"b":90},{"a":48,"b":80},{"a":48,"b":80},{"a":45,"b":83},{"a":45,"b":83},{"a":44,"b":85},{"a":44,"b":85},{"a":43,"b":86},{"a":43,"b":86},{"a":42,"b":87},{"a":42,"b":87},{"a":41,"b":89},{"a":41,"b":89},{"a":43,"b":91},{"a":43,"b":91},{"a":44,"b":84},{"a":44,"b":84},{"a":45,"b":84}]},"mark":{"type":"line","point":false,"strokeWidth":1.8},"encoding":{"x":{"field":"t","type":"quantitative","title":"Elapsed profiling time (s)","scale":{"zero":true}},"y":{"field":"v","type":"quantitative","title":"Requests (count)","scale":{"zero":true}},"color":{"field":"state","type":"nominal","title":"Scheduler state","scale":{"scheme":"category10"}},"tooltip":[{"field":"t","type":"quantitative","title":"Elapsed time (s)"},{"field":"state","type":"nominal","title":"State"},{"field":"v","type":"quantitative","title":"Requests"}]},"transform":[{"window":[{"op":"row_number","as":"i"}]},{"calculate":"3 + 15 * (datum.i - 1)","as":"t"},{"fold":["a","b"],"as":["k","v"]},{"calculate":"datum.k === 'a' ? 'Running' : 'Waiting'","as":"state"}]}
```

The model averaged 43.6 running and 83.5 waiting requests. Median queue time was 103.9 seconds and p99 queue time averaged 235.6 seconds. The workload therefore saturated request admission at concurrency 128 even though there were no HTTP errors.

Figure 7 counts the store-admission warnings visible in the retained model log at the log’s native one-second timestamp resolution. Provenance: exact parsing of scheduler.py:802 warning lines. The artifact retained only the final portion of the server log, so 107,670 is a lower bound for the run.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":700,"height":300,"title":"Figure 7 — CPU-tier store admission failures (1 s counts)","data":{"values":[{"v":63},{"v":171},{"v":122},{"v":169},{"v":154},{"v":154},{"v":137},{"v":182},{"v":394},{"v":214},{"v":182},{"v":152},{"v":184},{"v":245},{"v":147},{"v":154},{"v":133},{"v":182},{"v":180},{"v":160},{"v":300},{"v":489},{"v":336},{"v":165},{"v":221},{"v":216},{"v":245},{"v":141},{"v":132},{"v":147},{"v":129},{"v":319},{"v":238},{"v":565},{"v":160},{"v":140},{"v":141},{"v":180},{"v":229},{"v":141},{"v":131},{"v":134},{"v":149},{"v":118},{"v":140},{"v":163},{"v":165},{"v":148},{"v":125},{"v":143},{"v":103},{"v":118},{"v":109},{"v":164},{"v":431},{"v":112},{"v":110},{"v":112},{"v":191},{"v":224},{"v":105},{"v":289},{"v":107},{"v":125},{"v":96},{"v":448},{"v":421},{"v":179},{"v":436},{"v":137},{"v":123},{"v":84},{"v":118},{"v":105},{"v":98},{"v":237},{"v":154},{"v":456},{"v":456},{"v":188},{"v":80},{"v":96},{"v":110},{"v":126},{"v":432},{"v":147},{"v":255},{"v":186},{"v":104},{"v":99},{"v":118},{"v":126},{"v":106},{"v":96},{"v":128},{"v":97},{"v":130},{"v":111},{"v":120},{"v":111},{"v":198},{"v":259},{"v":145},{"v":136},{"v":104},{"v":126},{"v":209},{"v":140},{"v":137},{"v":210},{"v":130},{"v":288},{"v":389},{"v":268},{"v":114},{"v":87},{"v":123},{"v":133},{"v":125},{"v":150},{"v":155},{"v":131},{"v":152},{"v":160},{"v":348},{"v":158},{"v":145},{"v":132},{"v":159},{"v":138},{"v":145},{"v":580},{"v":479},{"v":140},{"v":360},{"v":179},{"v":464},{"v":273},{"v":618},{"v":140},{"v":98},{"v":151},{"v":115},{"v":152},{"v":140},{"v":121},{"v":167},{"v":124},{"v":166},{"v":170},{"v":196},{"v":496},{"v":137},{"v":165},{"v":134},{"v":131},{"v":160},{"v":206},{"v":146},{"v":140},{"v":169},{"v":154},{"v":507},{"v":123},{"v":150},{"v":132},{"v":135},{"v":139},{"v":126},{"v":143},{"v":116},{"v":161},{"v":142},{"v":648},{"v":456},{"v":114},{"v":140},{"v":155},{"v":152},{"v":158},{"v":114},{"v":171},{"v":179},{"v":85},{"v":120},{"v":127},{"v":120},{"v":164},{"v":108},{"v":150},{"v":107},{"v":140},{"v":137},{"v":116},{"v":134},{"v":160},{"v":167},{"v":126},{"v":138},{"v":159},{"v":116},{"v":160},{"v":192},{"v":266},{"v":800},{"v":784},{"v":229},{"v":105},{"v":500},{"v":685},{"v":146},{"v":382},{"v":131},{"v":324},{"v":702},{"v":249},{"v":89},{"v":183},{"v":110},{"v":112},{"v":81},{"v":140},{"v":136},{"v":102},{"v":126},{"v":106},{"v":138},{"v":187},{"v":507},{"v":113},{"v":135},{"v":121},{"v":219},{"v":171},{"v":292},{"v":119},{"v":123},{"v":126},{"v":312},{"v":110},{"v":150},{"v":154},{"v":140},{"v":166},{"v":120},{"v":114},{"v":646},{"v":227},{"v":113},{"v":123},{"v":130},{"v":154},{"v":136},{"v":160},{"v":114},{"v":139},{"v":240},{"v":238},{"v":155},{"v":134},{"v":170},{"v":179},{"v":190},{"v":194},{"v":298},{"v":171},{"v":201},{"v":360},{"v":160},{"v":187},{"v":280},{"v":281},{"v":461},{"v":141},{"v":157},{"v":114},{"v":287},{"v":297},{"v":141},{"v":157},{"v":115},{"v":107},{"v":149},{"v":158},{"v":133},{"v":154},{"v":164},{"v":171},{"v":170},{"v":155},{"v":630},{"v":388},{"v":114},{"v":692},{"v":324},{"v":134},{"v":289},{"v":220},{"v":102},{"v":526},{"v":153},{"v":102},{"v":305},{"v":119},{"v":106},{"v":120},{"v":144},{"v":150},{"v":148},{"v":494},{"v":270},{"v":482},{"v":159},{"v":100},{"v":111},{"v":78},{"v":98},{"v":91},{"v":76},{"v":71},{"v":76},{"v":80},{"v":77},{"v":86},{"v":77},{"v":71},{"v":92},{"v":73},{"v":107},{"v":91},{"v":111},{"v":113},{"v":89},{"v":104},{"v":97},{"v":99},{"v":112},{"v":314},{"v":105},{"v":195},{"v":262},{"v":96},{"v":288},{"v":216},{"v":90},{"v":119},{"v":128},{"v":179},{"v":153},{"v":146},{"v":226},{"v":79},{"v":98},{"v":78},{"v":112},{"v":84},{"v":141},{"v":163},{"v":74},{"v":104},{"v":481},{"v":164},{"v":84},{"v":105},{"v":107},{"v":104},{"v":85},{"v":118},{"v":93},{"v":130},{"v":127},{"v":133},{"v":99},{"v":341},{"v":357},{"v":432},{"v":113},{"v":92},{"v":269},{"v":303},{"v":112},{"v":90},{"v":481},{"v":164},{"v":131},{"v":310},{"v":489},{"v":140},{"v":238},{"v":84},{"v":76},{"v":110},{"v":302},{"v":70},{"v":83},{"v":172},{"v":79},{"v":83},{"v":96},{"v":246},{"v":97},{"v":92},{"v":80},{"v":82},{"v":84},{"v":84},{"v":87},{"v":97},{"v":60},{"v":98},{"v":106},{"v":108},{"v":98},{"v":84},{"v":116},{"v":105},{"v":75},{"v":127},{"v":136},{"v":105},{"v":142},{"v":136},{"v":510},{"v":663},{"v":543},{"v":119},{"v":156},{"v":527},{"v":514},{"v":134},{"v":222},{"v":148},{"v":157},{"v":591},{"v":153},{"v":108},{"v":169},{"v":234},{"v":179},{"v":568},{"v":281},{"v":146},{"v":140},{"v":160},{"v":166},{"v":137},{"v":198},{"v":135},{"v":158},{"v":140},{"v":172},{"v":133},{"v":112},{"v":157},{"v":166},{"v":147},{"v":136},{"v":515},{"v":741},{"v":674},{"v":363},{"v":126},{"v":380},{"v":152},{"v":138},{"v":353},{"v":140},{"v":540},{"v":577},{"v":139},{"v":126},{"v":145},{"v":181},{"v":156},{"v":161},{"v":151},{"v":115},{"v":134},{"v":158},{"v":163},{"v":173},{"v":157},{"v":161},{"v":171},{"v":148},{"v":149},{"v":176},{"v":125},{"v":160},{"v":116},{"v":379},{"v":152},{"v":247},{"v":499},{"v":135},{"v":304},{"v":133},{"v":140},{"v":137},{"v":133},{"v":151},{"v":132},{"v":133},{"v":117},{"v":220},{"v":248},{"v":249},{"v":140},{"v":121},{"v":138},{"v":145},{"v":248},{"v":144},{"v":150},{"v":360},{"v":241},{"v":440},{"v":743},{"v":172},{"v":176},{"v":167},{"v":426},{"v":191},{"v":147},{"v":129},{"v":278},{"v":313},{"v":128},{"v":163},{"v":534},{"v":208},{"v":380},{"v":217},{"v":114},{"v":159},{"v":128},{"v":129},{"v":152},{"v":154},{"v":739},{"v":562},{"v":277},{"v":114},{"v":189},{"v":132},{"v":216},{"v":112},{"v":87},{"v":298},{"v":104},{"v":115},{"v":115},{"v":85},{"v":119},{"v":15}]},"mark":{"type":"line","point":false,"strokeWidth":1.5},"encoding":{"x":{"field":"t","type":"quantitative","title":"Elapsed profiling time (s)","scale":{"zero":true}},"y":{"field":"v","type":"quantitative","title":"Cannot-store warnings (count/s)","scale":{"zero":true}},"color":{"value":"#d62728"},"tooltip":[{"field":"t","type":"quantitative","title":"Elapsed time (s)"},{"field":"v","type":"quantitative","title":"Warnings (count/s)"}]},"transform":[{"window":[{"op":"row_number","as":"i"}]},{"calculate":"1276 + datum.i - 1","as":"t"}]}
```

The warnings are not Ceph I/O exceptions: there were no filesystem short-read/write, ENOSPC, I/O-error, traceback, OOM, or pod-restart records. They mean the CPU primary tier could not find enough evictable blocks. In tiered mode, outstanding filesystem cascades are one major reason blocks remain pinned. The current telemetry cannot separate filesystem pins from simultaneous CPU→GPU read pins, but the very fast CPU↔GPU copies and persistent warning flood make filesystem backlog the leading explanation.

## Request quality

Figure 8 distinguishes successful responses from work cancelled at the duration boundary. Provenance: AIPerf phase-completion counters.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":700,"height":300,"title":"Figure 8 — Profiling request outcomes","data":{"values":[{"outcome":"Completed","requests":1283},{"outcome":"Cancelled at boundary","requests":103},{"outcome":"HTTP errors","requests":0}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"outcome","type":"nominal","title":"Profiling outcome","sort":null},"y":{"field":"requests","type":"quantitative","title":"Requests (count)","scale":{"zero":true}},"color":{"field":"outcome","type":"nominal","scale":{"scheme":"category10"},"legend":null},"tooltip":[{"field":"outcome","type":"nominal","title":"Outcome"},{"field":"requests","type":"quantitative","title":"Requests"}]}}
```

AIPerf completed 1,283 of 1,386 sent requests. It reported zero HTTP errors, but 103 requests—7.43% of those sent—were still in flight after the 30-second grace period and were cancelled. The extra 10-second cancellation-credit wait also expired. Although AIPerf marked the submission valid, this is not a clean steady-state capacity point.

Figure 9 summarizes client latency. Provenance: AIPerf completed-request distribution. ITL is plotted in seconds for a common axis.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","width":700,"height":300,"title":"Figure 9 — Client latency distribution","data":{"values":[{"metric":"TTFT","percentile":"p50","seconds":119.35},{"metric":"TTFT","percentile":"p90","seconds":130.25},{"metric":"TTFT","percentile":"p99","seconds":152.31},{"metric":"End-to-end","percentile":"p50","seconds":148.87},{"metric":"End-to-end","percentile":"p90","seconds":255.04},{"metric":"End-to-end","percentile":"p99","seconds":585.31},{"metric":"ITL","percentile":"p50","seconds":0.0854},{"metric":"ITL","percentile":"p90","seconds":0.1154},{"metric":"ITL","percentile":"p99","seconds":0.1377}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"metric","type":"nominal","title":"Latency metric","sort":null},"xOffset":{"field":"percentile"},"y":{"field":"seconds","type":"quantitative","title":"Latency (s)","scale":{"zero":true}},"color":{"field":"percentile","type":"nominal","title":"Percentile","scale":{"scheme":"category10"}},"tooltip":[{"field":"metric","type":"nominal","title":"Metric"},{"field":"percentile","type":"nominal","title":"Percentile"},{"field":"seconds","type":"quantitative","title":"Latency (s)","format":".3f"}]}}
```

Key completed-request results:

| Metric | Result |
|---|---:|
| Request throughput | 0.701 requests/s |
| Input throughput | 46,157.7 tokens/s |
| Output throughput | 482.0 tokens/s |
| Mean input length | 65,831 tokens |
| Mean output length | 687 tokens |
| Mean / p50 / p99 TTFT | 115.46 / 119.35 / 152.31 s |
| Mean / p50 / p99 end-to-end | 173.48 / 148.87 / 585.31 s |
| Mean / p50 / p99 ITL | 87.6 / 85.4 / 137.7 ms |
| Effective concurrency | 121.64 requests |
| Effective prefill concurrency | 80.96 requests |
| Effective decode concurrency | 40.69 requests |
| Mean tokens in flight | 8.24 million |
| Estimated preemptions | about 38 |
| HTTP errors / pod restarts | 0 / 0 |

GPU SM utilization and framebuffer memory metrics also returned no series. GPU KV occupancy is available from vLLM, but device-level GPU saturation cannot be assessed from this artifact.

## Root-cause conclusion

There are three separate conclusions:

1. **CephFS storage was activated and written.** The mount, storage class, tier configuration, and 105.84 GiB retained PVC footprint prove this.
2. **CephFS reads/hits are not demonstrated.** The connector shows external loads, but the metric boundary is CPU↔GPU; direct Ceph read telemetry is absent. Saying “CephFS got hits” from this run would overstate the evidence.
3. **The run hit offload store backpressure.** The exact immediate failure is CPU primary-tier exhaustion with non-evictable blocks. The most likely underlying mechanism is slow or backlogged CPU→CephFS cascades pinning blocks while GPU→CPU store demand climbed, compounded by v0.23 synchronous filesystem lookups and a concurrency-128 workload that already kept roughly 84 requests waiting.

The workload matters, but it is not the only issue. Concurrency 128 created a working set and write rate large enough to expose the tier. However, simply increasing concurrency or lowering GPU memory further would make this already overloaded point worse and would not make filesystem hits measurable.

## What to change before the next CephFS run

1. **Pin vLLM 0.24.0 first.** Verify the running image and digest from the pod artifact, not only the intended profile. This is mandatory because the async batched filesystem lookup change is directly relevant.
2. **Fix Ceph observability before benchmarking.** Require non-empty series for pool read/write bytes/s, pool read/write IOPS, OSD latency, and MDS requests. Also add vLLM filesystem-tier counters: lookup hit/miss, read/write bytes and operations, read/write latency, pending read/write tasks, in-flight jobs, pinned CPU blocks, and cannot-store count.
3. **Use a two-phase mechanism test.** Fill more than 64 GiB of unique KV data, wait for secondary writes to drain, then replay the evicted prefixes. A CephFS success criterion should require filesystem read bytes/IOPS above zero and external-token reuse above the CPU-only control.
4. **Do not lower GPU memory utilization yet.** At 0.9, GPU KV usage already averaged 94% and the store path was failing admission. Lowering it would increase offload write pressure before the filesystem path is healthy.
5. **Do not increase concurrency yet.** Start the v0.24 mechanism rerun at 64 or 96, choose the highest point with no cannot-store flood and a bounded queue, and then add 128 as a stress point.
6. **Keep CPU at 64 GiB for the first corrected run.** More CPU would hide CephFS reads; less CPU would intensify the existing pinning failure. After the v0.24 and telemetry gates pass, sweep 32 and 64 GiB.
7. **Tune filesystem workers only with direct metrics.** The defaults are 16 read-priority and 16 write-priority threads. Test 16/32/64 write workers only while watching MDS/OSD latency, Ceph throughput, pending jobs, and cannot-store count; higher concurrency can worsen Ceph metadata pressure.
8. **Repeat selected points at least three times.** This run is a mechanism/debug datapoint, not a performance comparison datapoint.

## Acceptance gates for claiming CephFS offload

A future report may claim CephFS hits only when all of these are true:

- running image is v0.24.0 or newer and has zero model restarts;
- CephFS PVC is mounted and its retained footprint increases;
- direct CephFS client/pool read bytes or read IOPS are non-zero during replay;
- filesystem-tier read-hit/load counters are non-zero;
- CPU-tier-only and CPU+CephFS runs use the same seed, ordering, concurrency, GPU utilization, CPU bytes, topology, and warmup state;
- cannot-store warnings are zero or explicitly bounded;
- no in-flight requests remain after the benchmark grace period;
- at least three repetitions establish variance.

## Artifact evidence map

- Deployment intent: metadata/run-plan.json and rendered-manifests/llminferenceservice.yaml
- Running image, mount, restart status: manifests/pods/cpu-offloading-kserve-6548dd5f5f-k5wwj.yaml
- Client results and phase boundary: results/profile_export_aiperf.json and logs/logs/aiperf.log
- Store-admission warnings: logs/model/cpu-offloading-kserve-6548dd5f5f-k5wwj_main.log
- Prometheus archives: metrics/prometheus/
- Resolved Ceph queries: metrics/resolved_queries.json and metrics/metrics_summary.json
- vLLM 0.23 mechanism: local repository tag v0.23.0, tiering/manager.py, tiering/fs/manager.py, cpu/manager.py, and offloading/scheduler.py
- vLLM 0.24 lookup change: local repository tag v0.24.0, tiering/async_lookup.py and tiering/fs/manager.py

## Related notes

- [[2026-07-20 - AgentX CPU64 plus CephFS pressure plan]]
- [[2026-07-19 - vLLM 0.24 fixed-EPP clean rerun analysis]]