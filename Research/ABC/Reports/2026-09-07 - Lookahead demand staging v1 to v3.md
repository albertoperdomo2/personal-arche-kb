---
title: "Lookahead demand staging — v1 regression, diagnosis, fix, and the pivot to retrieval parallelism"
date: "2026-09-07"
type: "experiment-report"
experiment: "ABC"
status: "complete"
verdict: "Valid. Mechanism accepted as neutral; no end-to-end benefit on this workload. Retrieval parallelism identified as the larger lever and built but not yet measured end to end."
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
vllm_version: "v0.27.0"
images:
  control_stock: "vllm/vllm-openai:v0.27.0"
  lookahead_v1: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-lookahead-v1 (digest …fd0e3ec1ca284ffb2461)"
  lookahead_v2: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-lookahead-v2"
  lookahead_v3: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-lookahead-v3 (built, not yet run)"
tensor_parallelism: 8
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: "not explicitly set"
concurrency: 64
cpu_bytes: 274877906944
offload_spec: "TieringOffloadingSpec"
secondary_tier: "fs on node-local NVMe (/mnt/nvme-kv-cache)"
secondary_tier_threads: "64 read / 64 write"
shared_memory_size: "300Gi"
workload: "AIPerf inferencex-agentx-mvp, semianalysisai/cc-traces-weka-062126"
random_seed: 20260707
benchmark_duration_seconds: 1800
cache_bust: "first_turn_prefix"
cache_cleaning: "hostPath cleanup: true; verified 0.698% NVMe fill at start for the accepted pair"
knobs:
  lookahead_requests: 4
  lookahead_probe_ttl_steps: 8
  lookahead_max_probes: 3
  lookahead_probe_chunks: 1024
---

# Lookahead demand staging — v1 regression, diagnosis, fix, and the pivot to retrieval parallelism

> **Supersedes an earlier proposal of this same path.** The only substantive
> difference is the final bullet of §Related: the earlier draft claimed the
> retrieval-path learning needed correcting, which was itself unverified and
> wrong. Take this version.

## 1. Executive summary

**Research question.** vLLM's offloading connector retrieves offloaded KV
reactively: the lookup happens when the scheduler examines a request during
its admission loop. Can starting that lookup *earlier*, for requests still
queued, hide external-retrieval latency and reduce TTFT?

**Mechanism under test.** "Lookahead demand staging": after the admission loop
stops, the connector runs the ordinary `_lookup` for the next `K` waiting
requests in scheduling order, so their secondary-tier existence checks and
NVMe→CPU promotions begin while they queue. Probes allocate no GPU blocks. A
promotion started by a probe is refused rather than evicting CPU chunks that an
earlier-scheduled request will need — a "do no harm" gate in the sense of
[[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|the research audit]].
Deliberately *not* speculative: every key staged belongs to an already-admitted
request, which is what distinguishes this from the V1–V7 prefetch attempts.

**Result.** The mechanism works and is now nearly free, but it does not pay.
v1 cost −10.0% request throughput and +40.4% mean TTFT. Three defects in the
probe path were found, quantified, and fixed; the final paired run measured
−1.7% throughput with p95 TTFT flat — inside the noise band. The reason it
never converts is structural and is the report's main finding: **admission on
this workload is gated by batch capacity, not by metadata readiness**, so
removing retrieval stall frees time that was already being spent waiting.

**Pivot.** The same investigation located a larger, unconditional lever: the
filesystem tier's data path is single-threaded per job. That fix is
implemented (`v3`) and benchmarked on the target device at 2.33x, but has
**not** been measured end to end.

## 2. Validity verdict

**Valid**, for the accepted pair only (`0e982c4a` control versus `6c4bdb01`
treatment, 2026-09-07).

That pair is the only properly controlled comparison in the series: same batch
(13:45 / 13:46 start), same duration (57 / 58 min), both `FINISHED`, both at
`--gpu-memory-utilization=0.8`, both starting from an identical 0.698% NVMe
fill, and matched in shape throughout — running 17.65 vs 17.98, waiting 4.00 vs
3.93, GPU KV usage 61.2% vs 62.1%, external hit rate 0.5497 vs 0.5519, NVMe
busy 26% on both. The arms differ **only** in the three lookahead knobs,
verified by diffing the rendered `kv_connector_extra_config` of both profiles.

Earlier comparisons in the campaign are **conditionally valid or invalid** and
are preserved below with their defects, per §6.

One caveat stands even for the accepted pair: **one run per cell**. The
no-offload replicates measured 0.1304, 0.1315 and 0.1283 across two days
(±2.4%), but a fourth produced 0.0217 — a 6x outlier. A −1.7% delta is inside
what a single pair can wander, so the honest reading is "indistinguishable from
neutral", not "a measured 1.7% cost".

## 3. Main takeaways

- **Measured:** v1 lookahead cost −10.0% throughput and +40.4% mean TTFT
  against its own feature-off control.
- **Measured:** the A/A gate was clean — the lookahead *image* with the feature
  disabled matched the stock image within +0.6% throughput. The regression was
  the feature, not the build.
- **Inference, quantitatively confirmed:** the cost was scheduler-thread time
  in the probe scans. A microbenchmark of the deployed code predicted +2.92 ms
  per engine step; the runs showed +3.35 ms, and the implied throughput loss
  (−10.2%) matched the observed −10.0%.
- **Measured:** after the fix, probe volume fell 7x (11.50 → 1.61/s) yet the
  per-block synchronous lookup p99 did not move (0.6416 → 0.6395 s). Cost
  scales with **burst size per probe**, not probe rate. Bounding the scan to
  1024 chunks then halved the excess.
- **Measured:** the mechanism does deliver its intended effect — external
  retrieval stall fell 32.4% in the accepted pair (3.44 → 2.32 s mean).
- **Inference:** that saving does not convert. Running requests, waiting depth
  and GPU KV usage were unchanged between arms, so retrieval was not what
  requests were waiting on. Admission is capacity-bound.
- **Measured, and a correction to the campaign's working hypothesis:** the
  4.27 s tier metadata delay is **not** I/O. 4,000 `faccessat` calls take
  55.8 ms on the target NVMe; a request's ~2,600 keys take ~36 ms, about 1% of
  the reported delay. The rest is a request waiting to be examined again.
- **Measured:** the filesystem tier's *data* path is genuinely thread-bound.
  512 × 2 MiB blocks read serially took 372 ms (2.89 GB/s) versus 160 ms
  (6.72 GB/s) split across threads — 2.33x, on a device the runs left 74% idle.

## 4. Headline metrics

Baseline is the feature-off NVMe control in each comparison. Deltas are
relative.

| Configuration | Run | Throughput (req/s) | Δ | Mean TTFT (ms) | Δ | p95 TTFT (ms) | Δ | Mean ITL (ms) | Δ |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|
| Stock v0.27.0, NVMe | `65ccbf10` | 0.6739 | — | 8,321 | — | 23,671 | — | 56.04 | — |
| v1 image, feature OFF (A/A) | `1573078c` | 0.6777 | +0.6% | 8,109 | −2.5% | 23,764 | +0.4% | 55.38 | −1.2% |
| **v1 lookahead ON** | `ffe1170a` | 0.6098 | **−10.0%** | 11,384 | **+40.4%** | 31,259 | +31.5% | 62.54 | +12.9% |
| v2, full scan (cross-day) | `ca3ecd3c` | 0.6272 | −7.5% | 10,457 | +28.9% | 28,715 | +20.8% | 59.11 | +6.7% |
| **v2 control (paired)** | `0e982c4a` | 0.6745 | — | 8,213 | — | 24,257 | — | 55.30 | — |
| **v2 @1024 (paired)** | `6c4bdb01` | 0.6630 | **−1.7%** | 8,665 | **+5.5%** | 24,252 | **−0.02%** | 55.51 | +0.4% |

No-offload baselines (same software, offloading disabled), used as a
cross-day cluster check: 0.1304, 0.1315 (09-04), 0.1283 (09-05). A fourth,
`5cffd9d9`, returned 0.0217 and is treated as an outlier.

## 5. Evidence

### Figure 1 — The regression shrinks across versions but never turns positive

Provenance: AIPerf aggregates from the runs in §4; each version compared to
its own feature-off control. Negative is better for latency, positive is
better for throughput.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 1 — Lookahead delta versus its own feature-off control, by version",
  "width": 520,
  "height": 300,
  "data": {
    "values": [
      {"version": "v1 (full scan)", "metric": "Request throughput", "delta": -10.0},
      {"version": "v1 (full scan)", "metric": "Mean TTFT", "delta": 40.4},
      {"version": "v1 (full scan)", "metric": "p95 TTFT", "delta": 31.5},
      {"version": "v1 (full scan)", "metric": "Mean ITL", "delta": 12.9},
      {"version": "v2 (full scan)", "metric": "Request throughput", "delta": -7.5},
      {"version": "v2 (full scan)", "metric": "Mean TTFT", "delta": 28.9},
      {"version": "v2 (full scan)", "metric": "p95 TTFT", "delta": 20.8},
      {"version": "v2 (full scan)", "metric": "Mean ITL", "delta": 6.7},
      {"version": "v2 (1024-chunk bound)", "metric": "Request throughput", "delta": -1.7},
      {"version": "v2 (1024-chunk bound)", "metric": "Mean TTFT", "delta": 5.5},
      {"version": "v2 (1024-chunk bound)", "metric": "p95 TTFT", "delta": -0.02},
      {"version": "v2 (1024-chunk bound)", "metric": "Mean ITL", "delta": 0.4}
    ]
  },
  "mark": {"type": "bar"},
  "encoding": {
    "y": {"field": "metric", "type": "nominal", "title": null, "sort": ["Request throughput", "Mean TTFT", "p95 TTFT", "Mean ITL"]},
    "x": {"field": "delta", "type": "quantitative", "title": "Relative delta versus control (%)"},
    "yOffset": {"field": "version"},
    "color": {"field": "version", "type": "nominal", "title": "Version", "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "version", "type": "nominal"},
      {"field": "metric", "type": "nominal"},
      {"field": "delta", "type": "quantitative", "format": ".2f", "title": "Delta (%)"}
    ]
  }
}
```

The v1 → v2 step removed the re-scanning and the touch; the v2 → v2@1024 step
bounded the burst. Only the latter brought p95 TTFT and mean ITL back to
control.

### Figure 2 — Cost scales with burst size, not probe rate

This is the decisive mechanism view. Excess is the treatment's per-block
synchronous tier-lookup p99 minus its own control's. Provenance:
`kv_offload_tiering_lookup_sync_delay_seconds` p99, from each run's
`metrics_summary.json`.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 2 — Excess synchronous lookup p99 over control, against probe rate and burst size",
  "width": 520,
  "height": 280,
  "data": {
    "values": [
      {"config": "v1: 11.50 probes/s, ~2628 chunks/probe", "excess_s": 0.404, "probes_per_s": 11.5, "chunks": 2628},
      {"config": "v2: 1.61 probes/s, ~2628 chunks/probe", "excess_s": 0.402, "probes_per_s": 1.61, "chunks": 2628},
      {"config": "v2: 1.61 probes/s, 1024 chunks/probe", "excess_s": 0.215, "probes_per_s": 1.61, "chunks": 1024}
    ]
  },
  "mark": {"type": "bar"},
  "encoding": {
    "y": {"field": "config", "type": "nominal", "title": null, "sort": null},
    "x": {"field": "excess_s", "type": "quantitative", "title": "Excess lookup p99 over control (seconds)"},
    "color": {"field": "chunks", "type": "nominal", "title": "Chunks per probe", "scale": {"scheme": "category10"}},
    "tooltip": [
      {"field": "config", "type": "nominal"},
      {"field": "excess_s", "type": "quantitative", "title": "Excess (s)"},
      {"field": "probes_per_s", "type": "quantitative", "title": "Probes/s"}
    ]
  }
}
```

Cutting the probe *rate* 7.1x left the excess unchanged (0.404 → 0.402 s).
Cutting the *burst* 2.6x nearly halved it (→ 0.215 s). The cost is dominated by
the per-probe batch: each probe seeds its whole key list into the tier's single
async-lookup thread, whose results are then drained on the scheduler thread.

### Figure 3 — Where the control arm's TTFT goes, and why the device is not the constraint

Provenance: run `0e982c4a` (reactive control, no lookahead). Device capability
measured directly on node `diadochos-hqxzk-gpu-h100-gjfjh` via `oc debug`.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 3 — NVMe throughput: what the run used versus what the device delivers",
  "width": 520,
  "height": 240,
  "data": {
    "values": [
      {"case": "Observed during run (26% busy)", "gbps": 0.54, "kind": "Measured in run"},
      {"case": "Serial read, 1 task/job (today's code)", "gbps": 2.89, "kind": "Device benchmark"},
      {"case": "Parallel read, 32 blocks/task", "gbps": 6.72, "kind": "Device benchmark"}
    ]
  },
  "mark": {"type": "bar"},
  "encoding": {
    "y": {"field": "case", "type": "nominal", "title": null, "sort": null},
    "x": {"field": "gbps", "type": "quantitative", "title": "Throughput (GB/s)"},
    "color": {"field": "kind", "type": "nominal", "title": null, "scale": {"scheme": "category10"}},
    "tooltip": [{"field": "case", "type": "nominal"}, {"field": "gbps", "type": "quantitative", "title": "GB/s"}]
  }
}
```

The run used 540 MB/s at 26% busy on a device that sustains 2.89 GB/s even
*serially*. Storage bandwidth was never the constraint; the serialization was.

## 6. Validity and failure evidence

Rejected and conditionally valid runs, preserved:

| Run | Defect | Disposition |
|---|---|---|
| `5cffd9d9` | No-offload replicate returning 0.0217 req/s against 0.1304 / 0.1315 for identical config | Outlier; establishes that a single run on this cluster can be 6x off |
| `5786964` | v2 treatment with **no contemporaneous control** — its m1 sibling failed after 17 min, and an earlier run failed at 12 min. 84 min duration versus ~59 | **Invalid.** Tier async lookup reached 55.4 s mean / 194.8 s max; not interpretable |
| `ca3ecd3c` | v2 treatment compared cross-day against a control from the previous day on a different node | **Conditionally valid.** Shape matched the control closely; used only as a trend point |
| `ffe1170a`, `1573078c`, `65ccbf10` | Each cell on a different node; single runs | **Conditionally valid.** The A/A gate and the size of the v1 effect carry the conclusion |

**Three corrections made during the investigation**, recorded because each was
a wrong inference that survived for a while:

1. *"Run `5786964` started on a dirty KV cache."* Wrong. The 25.1% starting
   fill was a 1.8 TB **model cache** at `/var/mnt/benchflow-nvme/models` on the
   same XFS filesystem. The KV cache directory was verified empty on all four
   GPU nodes. `storage_nvme_filesystem_usage_percent_by_node_mount` measures
   the whole filesystem, not the tier. **Lesson: that metric is not a tier
   occupancy signal and must not be read as one.**
2. *"Each tier evicts independently by LRU."* Wrong for secondary tiers. The
   `SecondaryTierManager` interface permits eviction, but `FileSystemTierManager`
   implements none — no capacity bound, no LRU, and `touch()` is the base
   no-op. The only `os.remove` calls are error paths. Secondary tiers are
   cumulative; reclamation is the operator's job via the hostPath wipe. (The CPU
   primary tier *does* evict by LRU/ARC; the error was extending that to
   secondary tiers.)
3. *"The 4.27 s metadata latency is an artifact of one-thread-per-job."* Wrong.
   Measured, `faccessat` is ~1% of it. The parallel-lookup knob therefore
   ships defaulted **off**.

A fourth, smaller correction: the campaign initially recorded "zero upside"
for lookahead on the grounds that the external hit *rate* was unchanged. The
hit rate is the wrong measure — the *timing* metrics improved materially
(−32.4% stall). The hit rate is unchanged because the same data is served
either way; what changes is when.

## 7. Mechanism telemetry

Accepted pair, `0e982c4a` (control) versus `6c4bdb01` (treatment):

| Metric | Control | Lookahead | Δ |
|---|---:|---:|---:|
| `kv_offload_lookahead_probes` (per s) | — | 1.61 | new |
| `kv_offload_lookahead_probes_deferred_share` | — | 0.331 | new |
| `kv_offload_tiering_promotions_gated` | absent | absent | gate never fired |
| `kv_offload_tiering_lookup_sync_delay` p99 (s) | 0.2657 | 0.4808 | +81.0% |
| `kv_offload_lookup_async_delay` mean (s) | 3.4356 | 2.3211 | **−32.4%** |
| `kv_offload_lookup_async_delay` sum rate (s/s) | 3.4472 | 2.7307 | −20.8% |
| `num_requests_running` | 17.65 | 17.98 | +1.9% |
| `num_requests_waiting` | 4.00 | 3.93 | −1.8% |
| `queue_time_p50` (s) | 4.544 | 4.918 | +8.2% |
| `kv_cache_usage_perc` | 0.6117 | 0.6206 | +1.5% |

The `promotions_gated` counter never fired because the CPU tier sat at 3–4%
occupancy throughout: the do-no-harm gate was never the binding constraint,
and CPU capacity was never scarce. This is consistent with the write-through
store model — the CPU pool is populated by production rate as chunks complete,
not by memory pressure.

**The decisive row is `num_requests_running`.** A 32.4% cut in retrieval stall
moved it by 1.9% and moved waiting depth by −1.8%. If retrieval had been
gating admission, that saving would have surfaced as more requests running or
fewer waiting. It did not.

## 8. Why it does not convert

vLLM's reactive path already overlaps retrieval with queue wait: a deferred
request returns to the queue and retries, so its lookup proceeds while it
waits. The admission loop `continue`s on a deferred lookup and only `break`s
on an allocation failure. Lookahead's marginal population is therefore just
the requests behind a head-of-line allocation break — and on this workload
their retrieval was largely hidden by the time they were admitted anyway.

Expressed as the gain model from the original design: the predicted gain was
`min(W, P)` for lead time `W` and promotion time `P`, which assumed a request
runs as soon as its data is ready. It does not. It runs when capacity frees.

## 9. Conclusions and next steps

**Established.**

- Lookahead demand staging is correct, costs ~nothing when properly bounded,
  and removes about a third of external retrieval stall.
- It produces no end-to-end benefit on AgentX at concurrency 64 with a local
  NVMe tier, because admission is capacity-bound.
- Cost in this class of mechanism scales with per-probe burst size, not probe
  frequency. This generalizes to any future probing policy.
- The filesystem tier's data path is serialized per job, on a device left 74%
  idle. Parallelizing it is worth ~2.33x on the read, extrapolating to roughly
  1.1 s off a 3.4 s stall.

**Uncertain.**

- Whether the retrieval parallelization converts to TTFT. Unlike lookahead it
  shortens work on the **demand** path for every request with an external hit,
  so it is not subject to the capacity-bound argument — but that is a
  prediction, not a measurement.
- Whether lookahead would pay in a regime where retrieval genuinely gates
  admission: a slower tier (CephFS, object store), or working sets large
  enough that retrieval dominates queueing.

**Disposition.** Keep lookahead, defaulted off. Do not sweep its knobs
further. It becomes relevant again only if retrieval stops overlapping with
queueing.

**Next experiment.** A same-batch A/B of the retrieval fix alone, lookahead
off in both arms, varying `blocks_per_task` in the `secondary_tiers` entry:
`0` reproduces the old one-task-per-job behaviour, the default `32` is the
split. Both on `v0.27.0-lookahead-v3`. Requires ≥2 repetitions per cell given
the outlier history, and a verified clean NVMe start.

**Standing acceptance gate for this line of work.** Read
`kv_offload_tiering_lookup_sync_delay_seconds` p99 and `num_requests_running`
before throughput. A mechanism that reduces stall without moving the running
count is not on the critical path, whatever its hit counters say.

## 10. Implementation record

Branch `feat/kv-offload-lookahead-probe-v0.27.0`, worktree
`/Users/aperdomo/workspace/redhat/vllm-lookahead-v0.27.0`, uncommitted.

| Version | Change | Config knobs |
|---|---|---|
| v1 | Lookahead probe, do-no-harm gate, metrics | `lookahead_requests`, `lookahead_probe_ttl_steps` |
| v2 | TTL applies to deferred probes; probe cap; no `_touch` on probe; bounded scan | `+ lookahead_max_probes` (3), `lookahead_probe_chunks` (4096) |
| v3 | FS tier job split across pool threads; optional parallel existence checks | `+ blocks_per_task` (32), `n_lookup_threads` (1) — both in the tier entry |

The Containerfile fails the build if any v2 or v3 fix is missing, verified by
building from a deliberately stale tree and observing the expected
`AssertionError`. Tests: 193 in the lookahead set, 346 in the tiering suite,
run inside the built image against the overlaid package.

## 11. Run registry

| Role | Run | Link |
|---|---|---|
| NVMe, stock image | `65ccbf10c4354ab6b35e6e486b8b23a1` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/65ccbf10c4354ab6b35e6e486b8b23a1?workspace=benchflow) |
| NVMe, v1 image, feature off (A/A) | `1573078c65f743c9a0bb3ce72be08237` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/1573078c65f743c9a0bb3ce72be08237?workspace=benchflow) |
| NVMe, v1 lookahead ON | `ffe1170ac7d54fc5b0b40e28ae21afb7` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/ffe1170ac7d54fc5b0b40e28ae21afb7?workspace=benchflow) |
| NVMe, v2 ON, invalid (no control) | `5786964561f441c183f0bc303c355c98` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5786964561f441c183f0bc303c355c98?workspace=benchflow) |
| NVMe, v2 ON, full scan | `ca3ecd3cbcae4ef8bbb1e4514cb9469c` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/ca3ecd3cbcae4ef8bbb1e4514cb9469c?workspace=benchflow) |
| **NVMe, v2 control (accepted pair)** | `0e982c4a8094475fb84dfd63b9b9da0b` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/0e982c4a8094475fb84dfd63b9b9da0b?workspace=benchflow) |
| **NVMe, v2 @1024 (accepted pair)** | `6c4bdb0195524b0c9109b3075edf79cb` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/6c4bdb0195524b0c9109b3075edf79cb?workspace=benchflow) |
| No-offload baselines | `d7982fea…`, `f5bd52f0…`, `65a29303…`, `5cffd9d9…` (outlier) | experiment 328 |

## Related

- [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|Independent research audit]] — killed V7; this work follows its "deadline-aware staging" branch and confirms its warning that `useful` is not a performance metric.
- [[2026-08-23 - Working-set oracle AgentX first comparison|Working-set oracle]] — the prior attempt at staging; same capacity-bound wall reached from a different direction.
- [[../Future-Value Placement/00 - Index|Future-Value Placement]] — retention-side headroom, complementary to this movement-side result.
- [[../../../Engineering/Learnings/vLLM KV offload retrieval path - lookup, promotion, and load|vLLM KV offload retrieval path]] — the reference for the read path. Checked against this work and still accurate: it already records that `complete_store` cascades unconditionally to all secondary tiers, and its eviction discussion is correctly scoped to the CPU primary tier.