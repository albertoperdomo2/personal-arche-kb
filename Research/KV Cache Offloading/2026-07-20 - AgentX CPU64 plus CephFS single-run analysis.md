# 2026-07-20 — AgentX CPU64 + CephFS single-run mechanism audit

- MLflow run: [d2c57cdc56084c4193d71bfb8e1cfdfb](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/d2c57cdc56084c4193d71bfb8e1cfdfb?workspace=benchflow)
- Disposition: **CephFS writes proven; CephFS read hits not proven; reject as a clean v0.24 performance datapoint**
- Artifact root: /private/tmp/kv-cache-experiments/cephfs-run-2026-07-20
- Profiling window: 30 minutes, concurrency 128
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

Figure 1 shows the only storage-specific time series that returned data. Provenance: kubelet_volume_stats_used_bytes for PVC vllm-kv-cache, sampled by Benchflow. The single delayed step is a kubelet volume-stat observation, not an instantaneous 105.84 GiB write burst; use it as retained-footprint evidence, not device throughput.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": 700,
  "height": 300,
  "title": "Figure 1 — CephFS-backed PVC retained footprint",
  "data": {
    "values": [
      {
        "minute": 0,
        "gib": 0
      },
      {
        "minute": 9,
        "gib": 0
      },
      {
        "minute": 9.3,
        "gib": 105.84375
      },
      {
        "minute": 30,
        "gib": 105.84375
      }
    ]
  },
  "mark": {
    "type": "line",
    "point": true,
    "interpolate": "step-after",
    "strokeWidth": 3
  },
  "encoding": {
    "x": {
      "field": "minute",
      "type": "quantitative",
      "title": "Elapsed profiling time (min)",
      "scale": {
        "zero": true
      }
    },
    "y": {
      "field": "gib",
      "type": "quantitative",
      "title": "PVC retained data (GiB)",
      "scale": {
        "zero": true
      }
    },
    "color": {
      "value": "#1f77b4"
    },
    "tooltip": [
      {
        "field": "minute",
        "type": "quantitative",
        "title": "Elapsed time (min)"
      },
      {
        "field": "gib",
        "type": "quantitative",
        "title": "Retained data (GiB)",
        "format": ".2f"
      }
    ]
  }
}
```

The net footprint change over the 30-minute profile was:

$$
\Delta S_{\mathrm{PVC}} = 113{,}648{,}861{,}184\ \mathrm{bytes} = 105.84\ \mathrm{GiB}
$$

If spread over the full window it is 60.2 MiB/s of net retained growth. That is not physical write bandwidth: duplicate blocks are skipped, overwritten temporary files are not retained, and the kubelet statistic updated coarsely.

Figure 2 explicitly plots telemetry availability rather than inventing zero I/O. Provenance: archived Prometheus result-set sizes for Ceph pool throughput/IOPS, OSD latency, MDS requests, and container filesystem throughput/IOPS.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": 700,
  "height": 300,
  "title": "Figure 2 — CephFS read/write telemetry availability",
  "data": {
    "values": [
      {
        "metric": "Pool read/write throughput",
        "series": 0
      },
      {
        "metric": "Pool read/write IOPS",
        "series": 0
      },
      {
        "metric": "OSD read/write latency",
        "series": 0
      },
      {
        "metric": "MDS client requests",
        "series": 0
      },
      {
        "metric": "Container FS throughput",
        "series": 0
      },
      {
        "metric": "Container FS IOPS",
        "series": 0
      }
    ]
  },
  "layer": [
    {
      "mark": {
        "type": "bar"
      },
      "encoding": {
        "x": {
          "field": "metric",
          "type": "nominal",
          "title": "Requested telemetry quantity",
          "sort": null,
          "axis": {
            "labelAngle": -25
          }
        },
        "y": {
          "field": "series",
          "type": "quantitative",
          "title": "Returned Prometheus series (count)",
          "scale": {
            "domain": [
              0,
              1
            ],
            "zero": true
          }
        },
        "color": {
          "field": "metric",
          "type": "nominal",
          "scale": {
            "scheme": "category10"
          },
          "legend": null
        }
      }
    },
    {
      "mark": {
        "type": "point",
        "filled": true,
        "size": 110
      },
      "encoding": {
        "x": {
          "field": "metric",
          "type": "nominal",
          "sort": null
        },
        "y": {
          "field": "series",
          "type": "quantitative"
        },
        "color": {
          "field": "metric",
          "type": "nominal",
          "scale": {
            "scheme": "category10"
          },
          "legend": null
        },
        "tooltip": [
          {
            "field": "metric",
            "type": "nominal",
            "title": "Metric"
          },
          {
            "field": "series",
            "type": "quantitative",
            "title": "Returned series"
          }
        ]
      }
    },
    {
      "mark": {
        "type": "text",
        "dy": -12
      },
      "encoding": {
        "x": {
          "field": "metric",
          "type": "nominal",
          "sort": null
        },
        "y": {
          "field": "series",
          "type": "quantitative"
        },
        "text": {
          "value": "unavailable"
        }
      }
    }
  ]
}
```

The zeros in Figure 2 mean **no returned series**. Consequently, there is no valid CephFS read-throughput, write-throughput, read-IOPS, write-IOPS, or latency plot for this run. The report profile’s pool regex was broad enough to match common CephFS data-pool names, while Ceph health and MDS queries were also empty; this points more strongly to Ceph exporter visibility/access than to a narrow pool-name mismatch.

## Connector activity

Figure 3 plots native connector traffic. Provenance: archived vllm:kv_offload_total_bytes rates, aggregated by transfer_type and pod in two-minute bins. These are GPU↔CPU transfers; they are **not** CephFS read/write counters.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": 700,
  "height": 300,
  "title": "Figure 3 — Native connector transfer throughput",
  "data": {
    "values": [
      {
        "minute": 1,
        "direction": "CPU → GPU",
        "mib_s": 85.17
      },
      {
        "minute": 1,
        "direction": "GPU → CPU",
        "mib_s": 318.3
      },
      {
        "minute": 3,
        "direction": "CPU → GPU",
        "mib_s": 220.56
      },
      {
        "minute": 3,
        "direction": "GPU → CPU",
        "mib_s": 280.53
      },
      {
        "minute": 5,
        "direction": "CPU → GPU",
        "mib_s": 305.8
      },
      {
        "minute": 5,
        "direction": "GPU → CPU",
        "mib_s": 280.89
      },
      {
        "minute": 7,
        "direction": "CPU → GPU",
        "mib_s": 247.57
      },
      {
        "minute": 7,
        "direction": "GPU → CPU",
        "mib_s": 276.06
      },
      {
        "minute": 9,
        "direction": "CPU → GPU",
        "mib_s": 167.23
      },
      {
        "minute": 9,
        "direction": "GPU → CPU",
        "mib_s": 286.62
      },
      {
        "minute": 11,
        "direction": "CPU → GPU",
        "mib_s": 127.84
      },
      {
        "minute": 11,
        "direction": "GPU → CPU",
        "mib_s": 320.1
      },
      {
        "minute": 13,
        "direction": "CPU → GPU",
        "mib_s": 138.95
      },
      {
        "minute": 13,
        "direction": "GPU → CPU",
        "mib_s": 339.4
      },
      {
        "minute": 15,
        "direction": "CPU → GPU",
        "mib_s": 129.09
      },
      {
        "minute": 15,
        "direction": "GPU → CPU",
        "mib_s": 354.47
      },
      {
        "minute": 17,
        "direction": "CPU → GPU",
        "mib_s": 115.45
      },
      {
        "minute": 17,
        "direction": "GPU → CPU",
        "mib_s": 396.44
      },
      {
        "minute": 19,
        "direction": "CPU → GPU",
        "mib_s": 116.56
      },
      {
        "minute": 19,
        "direction": "GPU → CPU",
        "mib_s": 486.21
      },
      {
        "minute": 21,
        "direction": "CPU → GPU",
        "mib_s": 78.48
      },
      {
        "minute": 21,
        "direction": "GPU → CPU",
        "mib_s": 587.94
      },
      {
        "minute": 23,
        "direction": "CPU → GPU",
        "mib_s": 41.61
      },
      {
        "minute": 23,
        "direction": "GPU → CPU",
        "mib_s": 714.12
      },
      {
        "minute": 25,
        "direction": "CPU → GPU",
        "mib_s": 47.86
      },
      {
        "minute": 25,
        "direction": "GPU → CPU",
        "mib_s": 766.63
      },
      {
        "minute": 27,
        "direction": "CPU → GPU",
        "mib_s": 45.03
      },
      {
        "minute": 27,
        "direction": "GPU → CPU",
        "mib_s": 723.62
      },
      {
        "minute": 29,
        "direction": "CPU → GPU",
        "mib_s": 33.88
      },
      {
        "minute": 29,
        "direction": "GPU → CPU",
        "mib_s": 696.15
      }
    ]
  },
  "mark": {
    "type": "line",
    "point": true,
    "strokeWidth": 2.5
  },
  "encoding": {
    "x": {
      "field": "minute",
      "type": "quantitative",
      "title": "Elapsed profiling time (min)",
      "scale": {
        "zero": true
      }
    },
    "y": {
      "field": "mib_s",
      "type": "quantitative",
      "title": "Transfer throughput (MiB/s)",
      "scale": {
        "zero": true
      }
    },
    "color": {
      "field": "direction",
      "type": "nominal",
      "title": "Transfer direction",
      "scale": {
        "scheme": "category10"
      }
    },
    "tooltip": [
      {
        "field": "minute",
        "type": "quantitative",
        "title": "Elapsed time (min)"
      },
      {
        "field": "direction",
        "type": "nominal",
        "title": "Direction"
      },
      {
        "field": "mib_s",
        "type": "quantitative",
        "title": "Throughput (MiB/s)",
        "format": ".1f"
      }
    ]
  }
}
```

Across the profile, GPU→CPU stored 857.37 GiB at 487.75 MiB/s on average, while CPU→GPU loaded 226.70 GiB at 128.97 MiB/s. Late in the run, stores rose to roughly 700–767 MiB/s while loads fell to roughly 34–48 MiB/s. The store stream therefore became increasingly dominant.

Comparing logical GPU→CPU traffic with retained PVC growth gives about 8.1× more logical store traffic than net filesystem growth:

$$
\frac{487.75\ \mathrm{MiB/s}}{60.21\ \mathrm{MiB/s}} \approx 8.1
$$

This ratio is not a Ceph bandwidth efficiency measurement because repeated block hashes are intentionally skipped and PVC usage is coarse. It does show that the connector generated far more store work than the net unique footprint.

Figure 4 shows where server-side prompt tokens came from. Provenance: archived vLLM prompt-token source counters converted to two-minute rates and normalized within each time bin.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": 700,
  "height": 300,
  "title": "Figure 4 — Prompt-token source share",
  "data": {
    "values": [
      {
        "minute": 1,
        "source": "External KV transfer",
        "rate": 3582.56
      },
      {
        "minute": 1,
        "source": "Local GPU cache",
        "rate": 1074.85
      },
      {
        "minute": 1,
        "source": "Local compute",
        "rate": 55930.58
      },
      {
        "minute": 3,
        "source": "External KV transfer",
        "rate": 9411.93
      },
      {
        "minute": 3,
        "source": "Local GPU cache",
        "rate": 886.84
      },
      {
        "minute": 3,
        "source": "Local compute",
        "rate": 62769.51
      },
      {
        "minute": 5,
        "source": "External KV transfer",
        "rate": 13187.71
      },
      {
        "minute": 5,
        "source": "Local GPU cache",
        "rate": 1400.18
      },
      {
        "minute": 5,
        "source": "Local compute",
        "rate": 84895.49
      },
      {
        "minute": 7,
        "source": "External KV transfer",
        "rate": 10885.11
      },
      {
        "minute": 7,
        "source": "Local GPU cache",
        "rate": 1053.07
      },
      {
        "minute": 7,
        "source": "Local compute",
        "rate": 83765.69
      },
      {
        "minute": 9,
        "source": "External KV transfer",
        "rate": 7294.22
      },
      {
        "minute": 9,
        "source": "Local GPU cache",
        "rate": 1389.42
      },
      {
        "minute": 9,
        "source": "Local compute",
        "rate": 82131.66
      },
      {
        "minute": 11,
        "source": "External KV transfer",
        "rate": 5443.29
      },
      {
        "minute": 11,
        "source": "Local GPU cache",
        "rate": 1625.56
      },
      {
        "minute": 11,
        "source": "Local compute",
        "rate": 84392.77
      },
      {
        "minute": 13,
        "source": "External KV transfer",
        "rate": 5993.78
      },
      {
        "minute": 13,
        "source": "Local GPU cache",
        "rate": 1104.89
      },
      {
        "minute": 13,
        "source": "Local compute",
        "rate": 87658.47
      },
      {
        "minute": 15,
        "source": "External KV transfer",
        "rate": 5643.73
      },
      {
        "minute": 15,
        "source": "Local GPU cache",
        "rate": 1434.89
      },
      {
        "minute": 15,
        "source": "Local compute",
        "rate": 86445.43
      },
      {
        "minute": 17,
        "source": "External KV transfer",
        "rate": 5199.82
      },
      {
        "minute": 17,
        "source": "Local GPU cache",
        "rate": 1211.47
      },
      {
        "minute": 17,
        "source": "Local compute",
        "rate": 90461.71
      },
      {
        "minute": 19,
        "source": "External KV transfer",
        "rate": 5253.6
      },
      {
        "minute": 19,
        "source": "Local GPU cache",
        "rate": 633.11
      },
      {
        "minute": 19,
        "source": "Local compute",
        "rate": 88862.66
      },
      {
        "minute": 21,
        "source": "External KV transfer",
        "rate": 3482.84
      },
      {
        "minute": 21,
        "source": "Local GPU cache",
        "rate": 1245.69
      },
      {
        "minute": 21,
        "source": "Local compute",
        "rate": 86965.64
      },
      {
        "minute": 23,
        "source": "External KV transfer",
        "rate": 1753.16
      },
      {
        "minute": 23,
        "source": "Local GPU cache",
        "rate": 2750.49
      },
      {
        "minute": 23,
        "source": "Local compute",
        "rate": 87494.38
      },
      {
        "minute": 25,
        "source": "External KV transfer",
        "rate": 1995.64
      },
      {
        "minute": 25,
        "source": "Local GPU cache",
        "rate": 2266.98
      },
      {
        "minute": 25,
        "source": "Local compute",
        "rate": 90103.38
      },
      {
        "minute": 27,
        "source": "External KV transfer",
        "rate": 1904.22
      },
      {
        "minute": 27,
        "source": "Local GPU cache",
        "rate": 1100.49
      },
      {
        "minute": 27,
        "source": "Local compute",
        "rate": 93124.98
      },
      {
        "minute": 29,
        "source": "External KV transfer",
        "rate": 1505.78
      },
      {
        "minute": 29,
        "source": "Local GPU cache",
        "rate": 1302.4
      },
      {
        "minute": 29,
        "source": "Local compute",
        "rate": 91819.82
      }
    ]
  },
  "transform": [
    {
      "joinaggregate": [
        {
          "op": "sum",
          "field": "rate",
          "as": "total_rate"
        }
      ],
      "groupby": [
        "minute"
      ]
    },
    {
      "calculate": "100 * datum.rate / datum.total_rate",
      "as": "share"
    }
  ],
  "mark": {
    "type": "line",
    "point": true,
    "strokeWidth": 2.5
  },
  "encoding": {
    "x": {
      "field": "minute",
      "type": "quantitative",
      "title": "Elapsed profiling time (min)",
      "scale": {
        "zero": true
      }
    },
    "y": {
      "field": "share",
      "type": "quantitative",
      "title": "Prompt tokens by source (%)",
      "scale": {
        "domain": [
          0,
          100
        ],
        "zero": true
      }
    },
    "color": {
      "field": "source",
      "type": "nominal",
      "title": "Prompt-token source",
      "scale": {
        "scheme": "category10"
      }
    },
    "tooltip": [
      {
        "field": "minute",
        "type": "quantitative",
        "title": "Elapsed time (min)"
      },
      {
        "field": "source",
        "type": "nominal",
        "title": "Source"
      },
      {
        "field": "share",
        "type": "quantitative",
        "title": "Share (%)",
        "format": ".2f"
      },
      {
        "field": "rate",
        "type": "quantitative",
        "title": "Rate (tokens/s)",
        "format": ".0f"
      }
    ]
  }
}
```

Over the complete window, prompt-token shares averaged 92.42% local compute, 6.07% external KV transfer, and 1.51% local GPU-cache hit. The external share peaks early and falls to about 1.6% at the end. The model log’s cumulative external prefix-cache hit rate similarly falls from 7.1% to 3.6%. This proves some offload-connector reuse, but not which external tier served it. It also shows that the 94.82% trace-theoretical reuse did not translate into actual cache reuse.

## Capacity pressure and failure mechanism

Figure 5 plots GPU KV-cache occupancy. Provenance: archived vLLM kv_cache_usage_perc, with the active kserve_vllm series selected and averaged in two-minute bins. The red rule marks 90%.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": 700,
  "height": 300,
  "title": "Figure 5 — GPU KV-cache occupancy",
  "data": {
    "values": [
      {
        "minute": 1,
        "percent": 65.04
      },
      {
        "minute": 3,
        "percent": 98.43
      },
      {
        "minute": 5,
        "percent": 98.31
      },
      {
        "minute": 7,
        "percent": 98.62
      },
      {
        "minute": 9,
        "percent": 97.25
      },
      {
        "minute": 11,
        "percent": 97.78
      },
      {
        "minute": 13,
        "percent": 96.88
      },
      {
        "minute": 15,
        "percent": 96.29
      },
      {
        "minute": 17,
        "percent": 97.14
      },
      {
        "minute": 19,
        "percent": 97.82
      },
      {
        "minute": 21,
        "percent": 97.01
      },
      {
        "minute": 23,
        "percent": 96.28
      },
      {
        "minute": 25,
        "percent": 96.9
      },
      {
        "minute": 27,
        "percent": 96.22
      },
      {
        "minute": 29,
        "percent": 97.01
      }
    ]
  },
  "layer": [
    {
      "mark": {
        "type": "line",
        "point": true,
        "strokeWidth": 2.5
      },
      "encoding": {
        "x": {
          "field": "minute",
          "type": "quantitative",
          "title": "Elapsed profiling time (min)",
          "scale": {
            "zero": true
          }
        },
        "y": {
          "field": "percent",
          "type": "quantitative",
          "title": "GPU KV-cache utilization (%)",
          "scale": {
            "domain": [
              0,
              100
            ],
            "zero": true
          }
        },
        "color": {
          "value": "#1f77b4"
        },
        "tooltip": [
          {
            "field": "minute",
            "type": "quantitative",
            "title": "Elapsed time (min)"
          },
          {
            "field": "percent",
            "type": "quantitative",
            "title": "GPU KV utilization (%)",
            "format": ".1f"
          }
        ]
      }
    },
    {
      "mark": {
        "type": "rule",
        "strokeDash": [
          6,
          4
        ],
        "color": "#d62728"
      },
      "encoding": {
        "y": {
          "datum": 90,
          "type": "quantitative"
        }
      }
    }
  ]
}
```

GPU KV usage averaged 94.09%, had a p50 of 97.11%, and peaked at 99.78%. This confirms that 0.9 GPU memory utilization did not leave the server underfilled; the cache stayed near full after ramp-up.

Figure 6 shows the scheduler backlog. Provenance: archived active vLLM running and waiting-request gauges, averaged in two-minute bins.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": 700,
  "height": 300,
  "title": "Figure 6 — Model scheduler request pressure",
  "data": {
    "values": [
      {
        "minute": 1,
        "state": "Running",
        "requests": 30.88
      },
      {
        "minute": 1,
        "state": "Waiting",
        "requests": 68.5
      },
      {
        "minute": 3,
        "state": "Running",
        "requests": 43.88
      },
      {
        "minute": 3,
        "state": "Waiting",
        "requests": 82
      },
      {
        "minute": 5,
        "state": "Running",
        "requests": 48.75
      },
      {
        "minute": 5,
        "state": "Waiting",
        "requests": 89.5
      },
      {
        "minute": 7,
        "state": "Running",
        "requests": 52.75
      },
      {
        "minute": 7,
        "state": "Waiting",
        "requests": 83
      },
      {
        "minute": 9,
        "state": "Running",
        "requests": 44.13
      },
      {
        "minute": 9,
        "state": "Waiting",
        "requests": 83
      },
      {
        "minute": 11,
        "state": "Running",
        "requests": 44
      },
      {
        "minute": 11,
        "state": "Waiting",
        "requests": 84
      },
      {
        "minute": 13,
        "state": "Running",
        "requests": 44.63
      },
      {
        "minute": 13,
        "state": "Waiting",
        "requests": 82.25
      },
      {
        "minute": 15,
        "state": "Running",
        "requests": 42.63
      },
      {
        "minute": 15,
        "state": "Waiting",
        "requests": 83.75
      },
      {
        "minute": 17,
        "state": "Running",
        "requests": 43.25
      },
      {
        "minute": 17,
        "state": "Waiting",
        "requests": 82.13
      },
      {
        "minute": 19,
        "state": "Running",
        "requests": 41.75
      },
      {
        "minute": 19,
        "state": "Waiting",
        "requests": 89.75
      },
      {
        "minute": 21,
        "state": "Running",
        "requests": 43.5
      },
      {
        "minute": 21,
        "state": "Waiting",
        "requests": 86
      },
      {
        "minute": 23,
        "state": "Running",
        "requests": 45.5
      },
      {
        "minute": 23,
        "state": "Waiting",
        "requests": 87
      },
      {
        "minute": 25,
        "state": "Running",
        "requests": 44
      },
      {
        "minute": 25,
        "state": "Waiting",
        "requests": 86.5
      },
      {
        "minute": 27,
        "state": "Running",
        "requests": 44.25
      },
      {
        "minute": 27,
        "state": "Waiting",
        "requests": 84.5
      },
      {
        "minute": 29,
        "state": "Running",
        "requests": 43
      },
      {
        "minute": 29,
        "state": "Waiting",
        "requests": 88.5
      }
    ]
  },
  "mark": {
    "type": "line",
    "point": true,
    "strokeWidth": 2.5
  },
  "encoding": {
    "x": {
      "field": "minute",
      "type": "quantitative",
      "title": "Elapsed profiling time (min)",
      "scale": {
        "zero": true
      }
    },
    "y": {
      "field": "requests",
      "type": "quantitative",
      "title": "Requests (count)",
      "scale": {
        "zero": true
      }
    },
    "color": {
      "field": "state",
      "type": "nominal",
      "title": "Scheduler state",
      "scale": {
        "scheme": "category10"
      }
    },
    "tooltip": [
      {
        "field": "minute",
        "type": "quantitative",
        "title": "Elapsed time (min)"
      },
      {
        "field": "state",
        "type": "nominal",
        "title": "State"
      },
      {
        "field": "requests",
        "type": "quantitative",
        "title": "Requests",
        "format": ".1f"
      }
    ]
  }
}
```

The model averaged 43.6 running and 83.5 waiting requests. Median queue time was 103.9 seconds and p99 queue time averaged 235.6 seconds. The workload therefore saturated request admission at concurrency 128 even though there were no HTTP errors.

Figure 7 counts the store-admission warnings visible in the retained model log. Provenance: exact parsing of scheduler.py:802 warning lines. The artifact retained only the final portion of the server log, so 107,670 is a lower bound for the run.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": 700,
  "height": 300,
  "title": "Figure 7 — CPU-tier store admission failures in retained model log",
  "data": {
    "values": [
      {
        "minute": 21.5,
        "warnings": 2957
      },
      {
        "minute": 22.5,
        "warnings": 11508
      },
      {
        "minute": 23.5,
        "warnings": 11374
      },
      {
        "minute": 24.5,
        "warnings": 10665
      },
      {
        "minute": 25.5,
        "warnings": 13078
      },
      {
        "minute": 26.5,
        "warnings": 12987
      },
      {
        "minute": 27.5,
        "warnings": 8316
      },
      {
        "minute": 28.5,
        "warnings": 11133
      },
      {
        "minute": 29.5,
        "warnings": 13378
      },
      {
        "minute": 30.5,
        "warnings": 12274
      }
    ]
  },
  "mark": {
    "type": "bar"
  },
  "encoding": {
    "x": {
      "field": "minute",
      "type": "quantitative",
      "title": "Elapsed profiling time (min)",
      "scale": {
        "zero": true
      }
    },
    "y": {
      "field": "warnings",
      "type": "quantitative",
      "title": "Cannot-store warnings (count/min)",
      "scale": {
        "zero": true
      }
    },
    "color": {
      "value": "#d62728"
    },
    "tooltip": [
      {
        "field": "minute",
        "type": "quantitative",
        "title": "Elapsed time (min)"
      },
      {
        "field": "warnings",
        "type": "quantitative",
        "title": "Warnings (count/min)"
      }
    ]
  }
}
```

The warnings are not Ceph I/O exceptions: there were no filesystem short-read/write, ENOSPC, I/O-error, traceback, OOM, or pod-restart records. They mean the CPU primary tier could not find enough evictable blocks. In tiered mode, outstanding filesystem cascades are one major reason blocks remain pinned. The current telemetry cannot separate filesystem pins from simultaneous CPU→GPU read pins, but the very fast CPU↔GPU copies and persistent warning flood make filesystem backlog the leading explanation.

## Request quality

Figure 8 distinguishes successful responses from work cancelled at the duration boundary. Provenance: AIPerf phase-completion counters.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": 700,
  "height": 300,
  "title": "Figure 8 — Profiling request outcomes",
  "data": {
    "values": [
      {
        "outcome": "Completed",
        "requests": 1283
      },
      {
        "outcome": "Cancelled at boundary",
        "requests": 103
      },
      {
        "outcome": "HTTP errors",
        "requests": 0
      }
    ]
  },
  "mark": {
    "type": "bar"
  },
  "encoding": {
    "x": {
      "field": "outcome",
      "type": "nominal",
      "title": "Profiling outcome",
      "sort": null
    },
    "y": {
      "field": "requests",
      "type": "quantitative",
      "title": "Requests (count)",
      "scale": {
        "zero": true
      }
    },
    "color": {
      "field": "outcome",
      "type": "nominal",
      "scale": {
        "scheme": "category10"
      },
      "legend": null
    },
    "tooltip": [
      {
        "field": "outcome",
        "type": "nominal",
        "title": "Outcome"
      },
      {
        "field": "requests",
        "type": "quantitative",
        "title": "Requests"
      }
    ]
  }
}
```

AIPerf completed 1,283 of 1,386 sent requests. It reported zero HTTP errors, but 103 requests—7.43% of those sent—were still in flight after the 30-second grace period and were cancelled. The extra 10-second cancellation-credit wait also expired. Although AIPerf marked the submission valid, this is not a clean steady-state capacity point.

Figure 9 summarizes client latency. Provenance: AIPerf completed-request distribution. ITL is plotted in seconds for a common axis.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "width": 700,
  "height": 300,
  "title": "Figure 9 — Client latency distribution",
  "data": {
    "values": [
      {
        "metric": "TTFT",
        "percentile": "p50",
        "seconds": 119.35
      },
      {
        "metric": "TTFT",
        "percentile": "p90",
        "seconds": 130.25
      },
      {
        "metric": "TTFT",
        "percentile": "p99",
        "seconds": 152.31
      },
      {
        "metric": "End-to-end",
        "percentile": "p50",
        "seconds": 148.87
      },
      {
        "metric": "End-to-end",
        "percentile": "p90",
        "seconds": 255.04
      },
      {
        "metric": "End-to-end",
        "percentile": "p99",
        "seconds": 585.31
      },
      {
        "metric": "ITL",
        "percentile": "p50",
        "seconds": 0.0854
      },
      {
        "metric": "ITL",
        "percentile": "p90",
        "seconds": 0.1154
      },
      {
        "metric": "ITL",
        "percentile": "p99",
        "seconds": 0.1377
      }
    ]
  },
  "mark": {
    "type": "bar"
  },
  "encoding": {
    "x": {
      "field": "metric",
      "type": "nominal",
      "title": "Latency metric",
      "sort": null
    },
    "xOffset": {
      "field": "percentile"
    },
    "y": {
      "field": "seconds",
      "type": "quantitative",
      "title": "Latency (s)",
      "scale": {
        "zero": true
      }
    },
    "color": {
      "field": "percentile",
      "type": "nominal",
      "title": "Percentile",
      "scale": {
        "scheme": "category10"
      }
    },
    "tooltip": [
      {
        "field": "metric",
        "type": "nominal",
        "title": "Metric"
      },
      {
        "field": "percentile",
        "type": "nominal",
        "title": "Percentile"
      },
      {
        "field": "seconds",
        "type": "quantitative",
        "title": "Latency (s)",
        "format": ".3f"
      }
    ]
  }
}
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