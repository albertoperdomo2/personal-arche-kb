---
title: Selective loading token and EWMA pressure calibration
date: 2026-09-07
type: experiment-report
experiment: Selective KV loading threshold and pressure-signal calibration
topic: KV Cache Offloading
status: conditionally-valid
model: nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
model_revision: unknown
vllm_version: v0.27.0-based selective-load build
vllm_image: quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v2
vllm_image_digest: sha256:a52851b8bf871f4726a57907414a13363b6dcc2a111651aa534a642e358bb6a2
router_image: quay.io/rh-ee-aperdomo/llm-d-router-endpoint-picker:dev-ac5446eb
router_commit: ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4
tensor_parallelism: 8
replicas: 1
gpu: 8x H100
model_node: diadochos-hqxzk-gpu-h100-fx7c8
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: default
cpu_bytes: 274877906944
offload_spec: CPU primary tier via OffloadingConnector
secondary_tier: none
secondary_tier_threads: not-applicable
dev_shm: 300Gi
workload: paired targeted prefix sweep and controlled external-load pressure pilots
random_seed:
  break_even_and_unique_pressure: 20260907
  shared_prefix_pressure: 20260908
duration:
  batched_run_seconds: 1782.5
  shared_prefix_run_seconds: 428.2
cache_cleaning_state: unique prefixes seeded, evicted from HBM by 116 x approximately 16K-token unique churn requests, retained in CPU
configuration:
  load_allowed: omit kv_load_tiers
  forced_recompute: kv_load_tiers=[]
  offload_policy: preserve
  telemetry_sample_rate: 1.0
  pending_bytes_ewma_alpha: 0.2
  requested_scrape_interval_ms:
    unique_prefix: 100
    shared_prefix: 50
---

# Selective loading token and EWMA pressure calibration

## Executive summary

This experiment revisited the static external-token threshold with 100 randomized paired comparisons and tested whether an llm-d-router plugin can derive a useful exponentially weighted moving average (EWMA) from vLLM's existing rank-0 pending-load-bytes gauge. The deployment remained pinned to one H100 host, used one Nemotron 253B FP8 TP8 replica, retained the same vLLM image and CPU offload capacity, and kept telemetry sampling at 1.0 throughout.

At low transfer pressure, loading was faster in the median at every tested external-prefix size from 512 through 2,048 tokens. It won 17/20 pairs at 512 tokens and 20/20 at 2,048. This confirms that the existing provisional 1,024-token floor is conservative, but it does not identify the true lower crossing because the sweep did not repeat 128, 256, and 384 tokens.

The proposed router-side pending-bytes EWMA could not be calibrated. Across 4,070 successful metric scrapes, every observed rank-0 pending-load value was zero, even though vLLM structured events recorded 24 load submissions and an event-time peak of 1 GiB pending. Effective scrape cadence was 450–519 ms at p50 because serializing and transporting the full metrics endpoint took longer than the requested 50–100 ms interval. The transfer backlog existed for tens of milliseconds, then disappeared before the next scrape. With the current signal and scrape path, the router EWMA remains zero and cannot gate these requests.

## Validity verdict: Conditionally valid

The low-pressure token sweep is valid for this exact deployment fingerprint: all 100 load-allowed probes restored external KV, all 100 forced-recompute probes avoided it, every size has 20 paired repetitions, the model remained on the same node, and the pod stayed Ready with zero restarts.

The unique-prefix pressure matrix is a valid non-saturating pilot, but only one pair was collected per concurrency and load waits remained 8.5–14.1 ms. It cannot define a pressure threshold. The shared-prefix c32 experiment is diagnostic only: vLLM coalesced 32 identical demands into two load operations, and request latency was dominated by engine scheduling after the prefix became available. Do not use either pilot to rank policy performance or choose a byte threshold.

## Main takeaways

- Measured: loading had a positive median benefit at 512, 768, 1,024, 1,536, and 2,048 external tokens.
- Measured: load wins were 17/20, 18/20, 17/20, 19/20, and 20/20 as prefix size increased.
- Decision: retain 1,024 external reusable tokens as the conservative provisional static floor. The data supports it, but does not prove it is optimal.
- Measured: the c1–c16 unique-prefix pilot generated 2–17 load operations per cell, yet mean scheduler load wait stayed below 15 ms per operation.
- Measured: every scraped pending-load-bytes and pending-load-jobs sample was zero, while event-time telemetry reached 1 GiB and three pending scheduler jobs.
- Conclusion: applying an EWMA to asynchronously scraped instantaneous pending bytes is not viable with the current metrics path. Smoothing cannot recover impulses that were never sampled.
- Next design: expose a persistent or cumulative pressure signal from vLLM, then calibrate the router gate against request-visible load wait and paired load-versus-recompute latency.

## Headline metrics

There is no MLflow/AgentX throughput row for this direct targeted experiment. Wall time approximates one-token request TTFT plus response overhead and is compared only within randomized pairs.

| External tokens | Median external tokens observed | Mean benefit (ms) | Median benefit (ms) | P10 benefit (ms) | P90 benefit (ms) | Load wins |
|---:|---:|---:|---:|---:|---:|---:|
| 512 | 512 | 67.4 | 66.7 | -63.6 | 121.8 | 17/20 |
| 768 | 768 | 92.6 | 91.8 | 10.8 | 186.3 | 18/20 |
| 1,024 | 1,024 | 123.7 | 106.5 | -15.7 | 233.5 | 17/20 |
| 1,536 | 1,536 | 142.0 | 146.6 | 27.9 | 222.9 | 19/20 |
| 2,048 | 2,048 | 208.3 | 191.6 | 119.4 | 350.8 | 20/20 |

Positive benefit means recomputation took longer than loading.

## Low-pressure break-even evidence

Figure 1 contains every one of the 100 paired observations at the finest available request-level grain. The orange rule is the provisional 1,024-token router floor. No observations were removed or downsampled.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Paired request-latency benefit by external reusable tokens","width":720,"height":380,"data":{"values":[{"tokens":512,"repetition":3,"benefit_ms":498.952},{"tokens":512,"repetition":2,"benefit_ms":48.696},{"tokens":512,"repetition":4,"benefit_ms":36.851},{"tokens":512,"repetition":1,"benefit_ms":119.71},{"tokens":512,"repetition":8,"benefit_ms":-128.658},{"tokens":512,"repetition":7,"benefit_ms":14.497},{"tokens":512,"repetition":6,"benefit_ms":101.782},{"tokens":512,"repetition":5,"benefit_ms":140.342},{"tokens":528,"repetition":10,"benefit_ms":62.078},{"tokens":512,"repetition":12,"benefit_ms":18.329},{"tokens":512,"repetition":9,"benefit_ms":-56.356},{"tokens":512,"repetition":11,"benefit_ms":23.125},{"tokens":512,"repetition":16,"benefit_ms":4.241},{"tokens":512,"repetition":13,"benefit_ms":119.383},{"tokens":512,"repetition":15,"benefit_ms":82.845},{"tokens":512,"repetition":14,"benefit_ms":71.247},{"tokens":512,"repetition":17,"benefit_ms":96.48},{"tokens":512,"repetition":18,"benefit_ms":115.04},{"tokens":512,"repetition":20,"benefit_ms":116.759},{"tokens":512,"repetition":19,"benefit_ms":-138.206},{"tokens":768,"repetition":4,"benefit_ms":90.261},{"tokens":768,"repetition":1,"benefit_ms":-14.004},{"tokens":768,"repetition":3,"benefit_ms":112.853},{"tokens":768,"repetition":2,"benefit_ms":185.953},{"tokens":768,"repetition":8,"benefit_ms":73.796},{"tokens":768,"repetition":5,"benefit_ms":73.074},{"tokens":768,"repetition":7,"benefit_ms":93.258},{"tokens":768,"repetition":6,"benefit_ms":123.785},{"tokens":768,"repetition":10,"benefit_ms":33.912},{"tokens":768,"repetition":12,"benefit_ms":13.596},{"tokens":768,"repetition":11,"benefit_ms":-28.653},{"tokens":768,"repetition":9,"benefit_ms":54.226},{"tokens":768,"repetition":16,"benefit_ms":118.278},{"tokens":768,"repetition":14,"benefit_ms":116.046},{"tokens":768,"repetition":15,"benefit_ms":175.62},{"tokens":768,"repetition":13,"benefit_ms":64.312},{"tokens":768,"repetition":20,"benefit_ms":238.847},{"tokens":768,"repetition":19,"benefit_ms":189.627},{"tokens":768,"repetition":18,"benefit_ms":103.672},{"tokens":768,"repetition":17,"benefit_ms":33.071},{"tokens":1024,"repetition":2,"benefit_ms":396.356},{"tokens":1024,"repetition":1,"benefit_ms":138.378},{"tokens":1024,"repetition":3,"benefit_ms":55.113},{"tokens":1024,"repetition":4,"benefit_ms":129.914},{"tokens":1040,"repetition":7,"benefit_ms":215.383},{"tokens":1024,"repetition":8,"benefit_ms":107.08},{"tokens":1024,"repetition":5,"benefit_ms":29.885},{"tokens":1024,"repetition":6,"benefit_ms":-80.869},{"tokens":1024,"repetition":10,"benefit_ms":55.876},{"tokens":1024,"repetition":9,"benefit_ms":158.768},{"tokens":1040,"repetition":12,"benefit_ms":111.733},{"tokens":1024,"repetition":11,"benefit_ms":605.77},{"tokens":1024,"repetition":16,"benefit_ms":98.17},{"tokens":1024,"repetition":13,"benefit_ms":184.383},{"tokens":1024,"repetition":14,"benefit_ms":132.438},{"tokens":1024,"repetition":15,"benefit_ms":-31.287},{"tokens":1040,"repetition":20,"benefit_ms":16.476},{"tokens":1024,"repetition":18,"benefit_ms":-13.924},{"tokens":1024,"repetition":17,"benefit_ms":57.929},{"tokens":1024,"repetition":19,"benefit_ms":105.852},{"tokens":1536,"repetition":1,"benefit_ms":29.197},{"tokens":1536,"repetition":4,"benefit_ms":138.849},{"tokens":1536,"repetition":3,"benefit_ms":150.431},{"tokens":1536,"repetition":2,"benefit_ms":115.452},{"tokens":1536,"repetition":5,"benefit_ms":126.141},{"tokens":1536,"repetition":6,"benefit_ms":331.804},{"tokens":1536,"repetition":8,"benefit_ms":15.749},{"tokens":1536,"repetition":7,"benefit_ms":220.69},{"tokens":1536,"repetition":11,"benefit_ms":208.634},{"tokens":1536,"repetition":12,"benefit_ms":-122.096},{"tokens":1536,"repetition":9,"benefit_ms":211.82},{"tokens":1552,"repetition":10,"benefit_ms":166.959},{"tokens":1552,"repetition":15,"benefit_ms":212.032},{"tokens":1536,"repetition":16,"benefit_ms":37.211},{"tokens":1536,"repetition":14,"benefit_ms":142.717},{"tokens":1536,"repetition":13,"benefit_ms":126.654},{"tokens":1536,"repetition":17,"benefit_ms":172.879},{"tokens":1536,"repetition":19,"benefit_ms":242.626},{"tokens":1536,"repetition":20,"benefit_ms":139.147},{"tokens":1536,"repetition":18,"benefit_ms":173.534},{"tokens":2064,"repetition":3,"benefit_ms":176.468},{"tokens":2048,"repetition":4,"benefit_ms":230.153},{"tokens":2048,"repetition":1,"benefit_ms":8.308},{"tokens":2048,"repetition":2,"benefit_ms":351.906},{"tokens":2048,"repetition":6,"benefit_ms":251.592},{"tokens":2048,"repetition":8,"benefit_ms":160.663},{"tokens":2064,"repetition":5,"benefit_ms":350.692},{"tokens":2048,"repetition":7,"benefit_ms":178.156},{"tokens":2048,"repetition":9,"benefit_ms":176.143},{"tokens":2048,"repetition":10,"benefit_ms":234.487},{"tokens":2048,"repetition":11,"benefit_ms":187.707},{"tokens":2048,"repetition":12,"benefit_ms":170.029},{"tokens":2048,"repetition":13,"benefit_ms":14.288},{"tokens":2064,"repetition":16,"benefit_ms":247.988},{"tokens":2048,"repetition":14,"benefit_ms":131.053},{"tokens":2048,"repetition":15,"benefit_ms":196.068},{"tokens":2048,"repetition":19,"benefit_ms":410.531},{"tokens":2048,"repetition":18,"benefit_ms":167.542},{"tokens":2048,"repetition":17,"benefit_ms":195.56},{"tokens":2048,"repetition":20,"benefit_ms":325.839}]},"layer":[{"mark":{"type":"rule","color":"#666666","strokeDash":[4,4]},"encoding":{"y":{"datum":0}}},{"mark":{"type":"rule","color":"#ff7f0e","strokeDash":[6,4]},"encoding":{"x":{"datum":1024}}},{"mark":{"type":"point","filled":true,"size":55,"opacity":0.72},"encoding":{"x":{"field":"tokens","type":"quantitative","title":"External reusable tokens (tokens)"},"y":{"field":"benefit_ms","type":"quantitative","title":"Recompute minus load wall time (ms)"},"color":{"field":"tokens","type":"nominal","title":"External tokens","scale":{"scheme":"category10"}},"tooltip":[{"field":"tokens","type":"quantitative","title":"External tokens"},{"field":"repetition","type":"ordinal","title":"Repetition"},{"field":"benefit_ms","type":"quantitative","title":"Benefit (ms)"}]}}]}
```

The relationship is directionally monotonic in median benefit, but fixed scheduler variance still produces occasional losses below 2,048 tokens. The 512-token distribution includes three losses, while the 2,048-token bucket has none. Because the sweep begins at 512, it bounds rather than locates the lower crossing.

## Pressure pilot

The pressure pilot launched distinct 4,096-token external-prefix requests alongside a 1,024- or 2,048-token target. Both treatments retained the same background loads; only the target request changed between load allowed and forced recompute.

| Background concurrency | Target tokens | Load target (ms) | Recompute target (ms) | Benefit (ms) | Mean load wait per operation (ms) | Load operations |
|---:|---:|---:|---:|---:|---:|---:|
| 1 | 2,048 | 793.0 | 915.8 | 122.8 | 10.4 | 2 |
| 2 | 1,024 | 648.7 | 588.6 | -60.1 | 12.6 | 3 |
| 4 | 1,024 | 966.1 | 1123.2 | 157.1 | 8.5 | 5 |
| 8 | 1,024 | 1875.0 | 2345.0 | 470.0 | 14.1 | 9 |
| 16 | 1,024 | 2438.9 | 1985.4 | -453.5 | 13.7 | 17 |

Figure 2 shows the single paired observation at each pressure level. It is intentionally presented as pilot evidence without uncertainty bars; one pair per level is insufficient to infer a crossing.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2. Pilot target latency under concurrent external-load demand","width":700,"height":360,"data":{"values":[{"concurrency":1,"treatment":"Load target","latency_ms":792.993},{"concurrency":1,"treatment":"Recompute target","latency_ms":915.801},{"concurrency":2,"treatment":"Load target","latency_ms":648.657},{"concurrency":2,"treatment":"Recompute target","latency_ms":588.559},{"concurrency":4,"treatment":"Load target","latency_ms":966.095},{"concurrency":4,"treatment":"Recompute target","latency_ms":1123.175},{"concurrency":8,"treatment":"Load target","latency_ms":1875.014},{"concurrency":8,"treatment":"Recompute target","latency_ms":2344.993},{"concurrency":16,"treatment":"Load target","latency_ms":2438.884},{"concurrency":16,"treatment":"Recompute target","latency_ms":1985.398}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"concurrency","type":"ordinal","title":"Background external-load concurrency (requests)"},"xOffset":{"field":"treatment"},"y":{"field":"latency_ms","type":"quantitative","title":"Target request wall time (ms)","scale":{"zero":true}},"color":{"field":"treatment","type":"nominal","title":"Target policy","scale":{"scheme":"category10"}},"tooltip":[{"field":"concurrency","type":"ordinal","title":"Background concurrency"},{"field":"treatment","type":"nominal"},{"field":"latency_ms","type":"quantitative","title":"Wall time (ms)"}]}}
```

Loading lost at c2 and c16 and won at c1, c4, and c8. This non-monotonic single-repetition result is not evidence for a c16 veto. More importantly, transfer wait stayed small even when total request wall time rose, so the cells stressed engine scheduling more than the CPU-to-GPU transfer path.

## Pending-bytes EWMA observability

For a sampled instantaneous gauge (x_t), the prototype computes:

$$
EWMA_t = 0.2x_t + 0.8EWMA_{t-1}
$$

Every observed (x_t) was zero, so every router-side (EWMA_t) was also zero. Figure 3 contrasts the maximum value visible through metric scraping with the maximum recorded synchronously by vLLM transfer events during the pressure window. The event result is not a substitute metric available to EPP; it proves the queue impulse existed between scrapes.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3. Pending-load bytes visible by observation path","width":650,"height":340,"data":{"values":[{"observation":"100 ms requested scrape / unique-prefix pilot","bytes":0},{"observation":"50 ms requested scrape / shared-prefix pilot","bytes":0},{"observation":"vLLM event-time structured telemetry","bytes":1073741824}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"observation","type":"nominal","title":"Observation path","axis":{"labelAngle":-18}},"y":{"field":"bytes","type":"quantitative","title":"Maximum pending load bytes, rank 0 (bytes)","scale":{"zero":true}},"color":{"field":"observation","type":"nominal","title":"Observation path","scale":{"scheme":"category10"}},"tooltip":[{"field":"observation","type":"nominal"},{"field":"bytes","type":"quantitative","title":"Pending bytes"}]}}
```

Figure 4 explains the aliasing. Although the sampler requested 100 ms and 50 ms periods, the p50 completed-scrape intervals were 450.2 ms and 519.1 ms. The source resolution is the finest available completed-scrape cadence; no samples were thinned. The unique-prefix file contains 3,442 successful samples and one recorded scrape error. The shared-prefix file contains 628 successful samples and six recorded errors.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4. Effective metrics scrape cadence","width":620,"height":330,"data":{"values":[{"run":"Unique-prefix pilot","percentile":"p50","interval_ms":450.2},{"run":"Unique-prefix pilot","percentile":"p90","interval_ms":694.9},{"run":"Shared-prefix pilot","percentile":"p50","interval_ms":519.1},{"run":"Shared-prefix pilot","percentile":"p90","interval_ms":850.2}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"run","type":"nominal","title":"Pressure experiment"},"xOffset":{"field":"percentile"},"y":{"field":"interval_ms","type":"quantitative","title":"Observed interval between scrapes (ms)","scale":{"zero":true}},"color":{"field":"percentile","type":"nominal","title":"Percentile","scale":{"scheme":"category10"}},"tooltip":[{"field":"run","type":"nominal"},{"field":"percentile","type":"nominal"},{"field":"interval_ms","type":"quantitative","title":"Interval (ms)"}]}}
```

The current EPP metrics source normally scrapes much less frequently than this direct local experiment. Therefore, reducing the configured EPP interval alone is unlikely to make the instantaneous gauge reliable and would increase the cost of exporting the full vLLM metric registry.

## Shared-prefix diagnostic

At c32, vLLM coalesced the shared 65K-token prefix: the load-allowed cell completed two load operations totaling 66,576 tokens with 39.1 ms mean scheduler load wait. The forced-target cell completed one 65,536-token background load. The target wall times were 3,340.3 ms for load and 1,577.7 ms for recompute, while background requests took roughly 30–40 seconds.

This is not a transfer-saturation threshold. Coalescing prevented 32 independent CPU-to-GPU copies, and background latency was dominated by scheduling/execution. The result is useful because it rules out repeated identical prefixes as a calibration workload for pending-byte pressure.

## Validity and failure evidence

- Model pod: Ready, zero restarts after both experiments.
- Fixed model placement: `diadochos-hqxzk-gpu-h100-fx7c8`.
- Break-even mechanism checks: 0/100 load-allowed probes lacked external tokens; 0/100 forced-recompute probes loaded external tokens.
- Pressure matrix: only repetition 0 completed. The planned remaining repetitions were deliberately stopped after the paired pilot showed the pressure signal was not observable and the cells were not transfer-saturating.
- Structured telemetry over the pressure window: 24 load-direction transfer events, maximum rank-0 event-time pending load bytes 1,073,741,824, 211 resolutions, and maximum three pending scheduler jobs.
- Scrape telemetry: 4,070 successful samples and seven explicit scrape errors across both files; no missing samples were silently interpolated.
- CPU utilization, NUMA counters, and PCIe bandwidth were not available at request-level cadence. No claim about host CPU saturation is made.
- Direct targeted runs were not launched through BenchFlow and therefore have no MLflow run IDs.

## Decision

Keep the router token condition conceptually as:

$$
external\_reusable\_tokens \ge 1024
$$

for the next same-deployment validation. Do not configure a finite `maxPendingLoadBytesEWMA` from this dataset. With the current scraped gauge, any positive threshold would behave like “never veto,” while a zero threshold would veto everything.

Before activating the EWMA policy, expose one pressure signal whose information survives scrape latency. Candidate contracts are:

1. A time-aware pending-load-bytes EWMA computed inside vLLM and exported as a gauge.
2. A short-window maximum pending-load-bytes gauge with a documented hold interval longer than the metrics scrape interval.
3. Monotonic pending-byte-seconds and busy-time counters, with the router computing rates from consecutive endpoint snapshots.
4. A cumulative count of load-wait violations above calibrated latency buckets.

The first or second option is simplest for the existing scalar custom-metric extractor. The third is more composable but requires correct counter-delta and reset handling in EPP.

## Next experiment

1. Repeat 128, 256, 384, 512, 768, and 1,024 tokens with 20 paired observations to locate the low-pressure crossing.
2. Add a persistent vLLM pressure metric and verify that EPP observes the same pressure episode seen in structured telemetry.
3. Build a non-coalescing pressure corpus that fits within the CPU tier while keeping prefixes absent from HBM. Use several long prefixes and tune churn from measured HBM capacity rather than increasing request concurrency alone.
4. At every pressure level, collect at least 20 randomized target pairs and retain background load identically across treatments.
5. Choose the veto where paired target benefit becomes reliably negative, then validate always-load versus token-only versus token-plus-pressure policies on the same pinned-node AgentX trace.

## Artifact registry

| Artifact | Purpose | Acceptance |
|---|---|---|
| `.calibration-artifacts/2026-09-07-selective-load-ewma-calibration-batched.json` | 200 break-even probes, paired c1–c16 pressure pilot, native metric samples | Break-even accepted; pressure accepted as non-saturating pilot |
| `.calibration-artifacts/2026-09-07-selective-load-ewma-shared-prefix-pressure.json` | Shared 65K-prefix c32 pressure diagnostic | Diagnostic only |
| `.calibration-artifacts/2026-09-07-selective-load-ewma-calibration.json` | Superseded 40-probe schedule stopped during optimization | Rejected as incomplete; preserved |
| `.calibration-artifacts/run_selective_load_ewma_calibration.py` | Reproducible request, eviction, pressure, and sampling runner | Accepted tooling |
| `.calibration-artifacts/analyze_selective_load_ewma_calibration.py` | Reproducible summary calculation | Accepted tooling |

## Related

- [[Selective loading calibration test plan]]
- [[2026-09-07 selective loading calibration results]]
- [[2026-09-07 - Selective-load binary opt-out first AgentX run - diagnostic invalidation]]
- [[00 - Index]]

## Provenance

Direct execution against the Diadochos `benchflow` deployment on 2026-09-07; raw request and metric artifacts listed above; live deployment and image inspection; vLLM Prometheus metrics; structured `offload_telemetry` events; and pod-health checks. Figure 1 includes all paired request observations. No chart invents data or silently downsamples a source series.