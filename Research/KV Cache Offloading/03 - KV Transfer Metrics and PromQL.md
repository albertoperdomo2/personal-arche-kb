---
title: "KV Transfer Metrics and PromQL"
date: "2026-08-03"
type: "methodology"
topic: "KV Cache Offloading"
status: "active"
repo: "vllm-project/vllm"
commit: "4ee9702bee668a447e9983a6aefc16ebbc3ad32e"
---

# KV Transfer Metrics and PromQL

Working reference for measuring KV transfer and retrieval latency in offloading benchmarks: which metrics exist, the queries to use, how to interpret them, and where they mislead.

Every metric name below was verified against `vllm-project/vllm@4ee9702` — names, types, buckets, and labels read from source, not from docs. Companion to [[Experiment Methodology]] and [[01 - Calibration Protocol]]. For the mechanism these metrics describe, see [[vLLM KV offload retrieval path - lookup, promotion, and load]].

All `vllm:kv_offload_*` metrics carry the labels **`model_name`** and **`engine`**.

---

## 1. Metric inventory

### 1.1 Stall / wait — the primary signals

Emitted from the scheduler process. These measure how long requests wait on retrieval.

| Metric | Type | Unit | Meaning |
|---|---|---|---|
| `vllm:kv_offload_tiering_lookup_async_delay_seconds` | histogram | s | **Promotion stall.** First deferred secondary-tier lookup → allocation or request finish |
| `vllm:kv_offload_tiering_lookup_sync_delay_seconds` | histogram | s | Accumulated *blocking* time querying secondary tiers, per request |
| `vllm:kv_offload_lookup_async_delay_seconds` | histogram | s | Connector-level equivalent; includes CPU-tier deferrals |
| `vllm:kv_offload_lookup_sync_delay_seconds` | histogram | s | Duration of a single `_lookup()` call on the scheduler thread |

Buckets — **note the ceilings**:

- async delay (both): `0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5, 10`
- sync delay (both): `0.00001, 0.00005, 0.0001, 0.0005, 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1`

Definitions: [`tiering/spec.py:86-129`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/spec.py#L86-L129), [`metrics.py:85-123`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/metrics.py#L85-L123).

### 1.2 Transfer — CPU↔GPU only

Measured from CUDA start/end events ([`gpu_worker.py:419-431`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/gpu_worker.py#L419-L431)), so this is **pure DMA time**, excluding queueing.

| Metric | Type | Unit |
|---|---|---|
| `vllm:kv_offload_load_bytes` | counter | bytes (CPU→GPU) |
| `vllm:kv_offload_load_time` | counter | seconds |
| `vllm:kv_offload_load_size` | histogram | bytes/op |
| `vllm:kv_offload_store_bytes` | counter | bytes (GPU→CPU) |
| `vllm:kv_offload_store_time` | counter | seconds |
| `vllm:kv_offload_store_size` | histogram | bytes/op |

Size buckets: `1e6, 5e6, 10e6, 20e6, 40e6, 60e6, 80e6, 100e6, 150e6, 200e6`.

Deprecated equivalents still emitted for `CPUOffloadingSpec`: `vllm:kv_offload_total_bytes`, `_total_time`, `_size`, all labelled `transfer_type={CPU_to_GPU,GPU_to_CPU}`. **Prefer the flat names**; the labelled ones exist only for dashboard compatibility.

### 1.3 CPU tier capacity

| Metric | Type | Meaning |
|---|---|---|
| `vllm:kv_offload_cpu_cache_usage_perc` | gauge | fraction of CPU tier occupied by non-evictable blocks |
| `vllm:kv_offload_cpu_cache_read_usage_perc` | gauge | portion pinned by in-flight reads |
| `vllm:kv_offload_cpu_cache_write_usage_perc` | gauge | portion pinned by in-flight writes |
| `vllm:kv_offload_cpu_allocation_size` | histogram | blocks per allocation |
| `vllm:kv_offload_allocation_failure` | counter | store allocations that failed |
| `vllm:kv_offload_stores_skipped` | counter | only when `store_threshold >= 2` |

Source: [`cpu/manager.py:293-327`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/manager.py#L293-L327), names in [`cpu/common.py`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/cpu/common.py).

### 1.4 Core engine metrics that matter here

| Metric | Type | Note |
|---|---|---|
| `vllm:prompt_tokens_by_source` | counter | label `source` ∈ `local_compute`, `local_cache_hit`, `external_kv_transfer` |
| `vllm:external_prefix_cache_queries` / `_hits` | counter | external (connector) hit rate — **exposed name is `external_*`, not `connector_*`** |
| `vllm:prefix_cache_queries` / `_hits` | counter | local GPU prefix cache |
| `vllm:num_requests_waiting_by_reason` | gauge | label `reason` ∈ `capacity`, `deferred` |
| `vllm:num_requests_running` / `_waiting` | gauge | |
| `vllm:kv_cache_usage_perc` | gauge | GPU KV occupancy |
| `vllm:num_preemptions` | counter | |
| `vllm:time_to_first_token_seconds` | histogram | |
| `vllm:request_queue_time_seconds` | histogram | arrival → first scheduled |
| `vllm:request_prefill_time_seconds` | histogram | |
| `vllm:e2e_request_latency_seconds` | histogram | |
| `vllm:request_success` | counter | |

---

## 2. Queries by question

### Q1 — How much are requests stalling on retrieval?

The primary measurement. `_count` is the number of stalls; `_sum` is total stalled request-seconds.

```promql
# Stalls per second
rate(vllm:kv_offload_tiering_lookup_async_delay_seconds_count[1m])

# Average number of requests concurrently blocked on retrieval
rate(vllm:kv_offload_tiering_lookup_async_delay_seconds_sum[1m])

# Mean stall duration (s)
rate(vllm:kv_offload_tiering_lookup_async_delay_seconds_sum[1m])
  / rate(vllm:kv_offload_tiering_lookup_async_delay_seconds_count[1m])

# Distribution
histogram_quantile(0.5,  sum by (le) (rate(vllm:kv_offload_tiering_lookup_async_delay_seconds_bucket[1m])))
histogram_quantile(0.9,  sum by (le) (rate(vllm:kv_offload_tiering_lookup_async_delay_seconds_bucket[1m])))
histogram_quantile(0.99, sum by (le) (rate(vllm:kv_offload_tiering_lookup_async_delay_seconds_bucket[1m])))
```

For a bounded benchmark window, totals are usually more meaningful than rates:

```promql
increase(vllm:kv_offload_tiering_lookup_async_delay_seconds_sum[$__range])    # total seconds lost to stalls
increase(vllm:kv_offload_tiering_lookup_async_delay_seconds_count[$__range])  # total stall events
```

Isolate the secondary-tier contribution by differencing against the connector-level metric:

```promql
increase(vllm:kv_offload_tiering_lookup_async_delay_seconds_sum[$__range])
  - increase(vllm:kv_offload_lookup_async_delay_seconds_sum[$__range])
```

### Q2 — Is the scheduler thread being blocked?

Should be microseconds. If not, the async lookup path is not working as designed.

```promql
histogram_quantile(0.99, sum by (le) (rate(vllm:kv_offload_tiering_lookup_sync_delay_seconds_bucket[1m])))
histogram_quantile(0.99, sum by (le) (rate(vllm:kv_offload_lookup_sync_delay_seconds_bucket[1m])))
```

### Q3 — How many requests are blocked right now?

```promql
vllm:num_requests_waiting_by_reason{reason="deferred"}   # includes KV transfer waits
vllm:num_requests_waiting_by_reason{reason="capacity"}   # waiting on GPU capacity
vllm:num_requests_running
```

**`deferred` is a mixed bucket** — it is the size of the `skipped_waiting` queue and bundles KV transfer with LoRA budget and blocked status ([`loggers.py:1117-1122`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/metrics/loggers.py#L1117-L1122)). With no LoRA in the run it is effectively the KV-transfer blocked-request gauge. Plot against `capacity` to separate retrieval stalls from GPU pressure.

### Q4 — What is CPU↔GPU transfer costing?

```promql
# Bandwidth, bytes/s
rate(vllm:kv_offload_load_bytes[1m])  / rate(vllm:kv_offload_load_time[1m])
rate(vllm:kv_offload_store_bytes[1m]) / rate(vllm:kv_offload_store_time[1m])

# Mean duration per load op (load_size is observed once per transfer)
rate(vllm:kv_offload_load_time[1m]) / rate(vllm:kv_offload_load_size_count[1m])

# Op rate and size
rate(vllm:kv_offload_load_size_count[1m])
histogram_quantile(0.9, sum by (le) (rate(vllm:kv_offload_load_size_bucket[1m])))

# Totals for the run
increase(vllm:kv_offload_load_bytes[$__range])
increase(vllm:kv_offload_store_bytes[$__range])
```

The **load/store byte ratio** is the single best sanity check on whether offloading is paying off: a tier that absorbs terabytes of writes and returns gigabytes of reads is not earning its keep.

```promql
increase(vllm:kv_offload_load_bytes[$__range]) / increase(vllm:kv_offload_store_bytes[$__range])
```

### Q5 — Is offload actually supplying prompt tokens?

```promql
rate(vllm:prompt_tokens_by_source{source="external_kv_transfer"}[1m])
rate(vllm:prompt_tokens_by_source{source="local_cache_hit"}[1m])
rate(vllm:prompt_tokens_by_source{source="local_compute"}[1m])

# External supply share
sum(rate(vllm:prompt_tokens_by_source{source="external_kv_transfer"}[1m]))
  / sum(rate(vllm:prompt_tokens_by_source[1m]))

# External hit rate
rate(vllm:external_prefix_cache_hits[1m]) / rate(vllm:external_prefix_cache_queries[1m])
```

### Q6 — Did it help?

```promql
histogram_quantile(0.9, sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[1m])))
histogram_quantile(0.9, sum by (le) (rate(vllm:request_queue_time_seconds_bucket[1m])))
histogram_quantile(0.9, sum by (le) (rate(vllm:request_prefill_time_seconds_bucket[1m])))
rate(vllm:generation_tokens[1m])
```

`request_queue_time_seconds` is where retrieval wait lands. Verified: `QUEUED` fires once at `add_request` ([`scheduler.py:2235`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/core/sched/scheduler.py#L2235)) and is **not** re-recorded when a request returns from `WAITING_FOR_REMOTE_KVS`; `SCHEDULED` fires only when the request actually runs ([`scheduler.py:1056-1059`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/core/sched/scheduler.py#L1056-L1059)). So the full retrieval wait is inside `queued_time` — conflated with ordinary queueing.

### Q7 — Guardrails (check before trusting anything above)

```promql
vllm:kv_offload_cpu_cache_usage_perc            # → 1.0 means promotions are failing silently
vllm:kv_offload_cpu_cache_read_usage_perc
vllm:kv_offload_cpu_cache_write_usage_perc
rate(vllm:kv_offload_allocation_failure[1m])    # should be 0
rate(vllm:num_preemptions[1m])                  # non-zero invalidates latency comparisons
vllm:kv_cache_usage_perc
rate(vllm:request_success[1m])
```

---

## 3. Interpretation identities

**Bandwidth.** Both are counters over the same set of transfers, so the ratio of rates is a true weighted mean:

$$
B = \frac{\Delta(\text{load\_bytes})}{\Delta(\text{load\_time})}
$$

**Mean stall.** Standard histogram mean:

$$
\bar{t}_{\text{stall}} = \frac{\Delta(\text{async\_delay\_sum})}{\Delta(\text{async\_delay\_count})}
$$

**Concurrently stalled requests.** `rate(_sum)` has units of seconds-of-stall per second, i.e. dimensionless — by Little's law this is the average number of requests blocked on retrieval at any instant:

$$
\bar{N}_{\text{blocked}} = \frac{d}{dt}\,\text{async\_delay\_sum} = \lambda \cdot \bar{t}_{\text{stall}}
$$

This is the most useful single scalar for comparing configurations: it converts a latency histogram into "how many requests is this costing me."

**Step-quantization check.** From [[vLLM KV offload retrieval path - lookup, promotion, and load]], a cold secondary hit costs at least ~3 engine steps plus tier I/O. If

$$
\bar{t}_{\text{stall}} \approx n \cdot t_{\text{step}}, \quad n \in \mathbb{Z},\ n \gtrsim 3
$$

then the stall is dominated by scheduler step granularity, not by device latency — and faster storage will not help.

---

## 4. Traps

1. **Survivorship bias in the async-delay histogram.** It is only observed when a lookup *defers*. Anything served from the CPU tier short-circuits and records nothing. So the histogram samples only the slow path. If a change increases CPU-tier hit rate, `_count` falls while the surviving p90 may *rise* — that is a success reading as a regression. **Always report `_count` and `_sum` alongside any quantile.** This becomes acute when evaluating prefetching.
2. **Bucket ceilings.** Async delay tops out at **10 s**, transfer size at **200 MB**. Values above land in `+Inf` and quantiles saturate — a p99 pinned at the top bucket means "≥10 s," not "10 s." Verify before quoting a tail:
   ```promql
   vllm:kv_offload_tiering_lookup_async_delay_seconds_bucket{le="+Inf"}
     - ignoring(le) vllm:kv_offload_tiering_lookup_async_delay_seconds_bucket{le="10"}
   ```
3. **Quantiles need `by (le)`.** Aggregating without it silently produces nonsense.
4. **Per-engine series under DP.** Each engine emits its own series. `sum by (le)` merges them; keep the `engine` label when checking for replica skew.
5. **`deferred` conflates causes** (Q3). Only clean without LoRA.
6. **`external_*` vs `connector_*`.** The internal variable is `connector_prefix_cache_*` but the **exposed metric name is `vllm:external_prefix_cache_*`**. Querying the internal name returns nothing.
7. **Counters reset on engine restart.** Always `rate`/`increase`, never raw deltas across a restart.
8. **Load/store time is DMA-only.** It excludes queueing and step latency, so it will look fast even when requests are stalling badly. Never use it as a proxy for retrieval latency.

---

## 5. Not instrumented

Confirmed absent at this commit:

- **Per-tier I/O time and bytes.** No secondary tier overrides `get_stats()` — the base returns `None` ([`tiering/base.py:313`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/base.py#L313)). Structural cause: `JobResult` carries only `job_id` and `success` ([`tiering/base.py:56-60`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/v1/kv_offload/tiering/base.py#L56-L60)) — there is nowhere to put a duration, unlike `TransferResult` on the CPU path.
- **Which tier served a hit.** No attribution in multi-tier configs.
- **Per-request attribution.** All of the above are aggregate histograms.
- **Prefetch/promotion accuracy.** Promoted blocks carry no provenance, so "was this promotion used before eviction" is unanswerable.

**In flight upstream:** PR [#48798](https://github.com/vllm-project/vllm/pull/48798) ("Add tiering offloading metrics") extends `JobResult` with optional transfer size/time and adds per-tier labelled read/write/failure/hit counters, under umbrella RFC [#44008](https://github.com/vllm-project/vllm/issues/44008). Under review by `orozery`, stalled on unaddressed review comments as of 2026-08-03. Re-check before designing around the gap — it may close.

**Workarounds meanwhile:**

- Node-level device telemetry (NVMe IOPS/throughput, CephFS client metrics) for tier I/O — attributable only when vLLM is the sole consumer.
- `VLLM_LOGGING_LEVEL=DEBUG` gives per-request tracing with request IDs at every retrieval decision point: `"Offloading manager delayed request %s"` ([`scheduler.py:763`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L763)), `"Delaying request %s since some of its chunks are already being loaded"` ([`:788`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L788)), `"Request %s hit %s offloaded tokens after %s GPU hit tokens"` ([`:795`](https://github.com/vllm-project/vllm/blob/4ee9702bee668a447e9983a6aefc16ebbc3ad32e/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L795)). Counting delay lines per request ID yields the promotion step count directly.
- Concurrency 1 removes queue pressure so `request_queue_time_seconds` ≈ retrieval time.
- Differential against a matched no-offload baseline attributes the `queued_time` delta to retrieval.

---

## 6. Minimum reporting set

For any offload run, record at least:

| Quantity | Query |
|---|---|
| Total stalled request-seconds | `increase(...tiering_lookup_async_delay_seconds_sum[$__range])` |
| Total stall events | `increase(...tiering_lookup_async_delay_seconds_count[$__range])` |
| Mean / p90 stall | ratio, and `histogram_quantile(0.9, ...)` |
| External prompt-token share | §Q5 |
| Load:store byte ratio | §Q4 |
| CPU tier peak usage | `max_over_time(vllm:kv_offload_cpu_cache_usage_perc[$__range])` |
| Allocation failures | `increase(vllm:kv_offload_allocation_failure[$__range])` |
| Preemptions | `increase(vllm:num_preemptions[$__range])` |
| TTFT p90, throughput | §Q6 |

A run with non-zero allocation failures or CPU usage pinned near 1.0 should be marked **conditionally valid at best** — promotions were failing, so tier comparisons are not meaningful.

---

## Provenance

Metric names, types, buckets, labels, and emission sites verified by direct source reading of `vllm-project/vllm` at `4ee9702bee668a447e9983a6aefc16ebbc3ad32e` (2026-08-03). Upstream PR/issue status checked the same day via `gh`.