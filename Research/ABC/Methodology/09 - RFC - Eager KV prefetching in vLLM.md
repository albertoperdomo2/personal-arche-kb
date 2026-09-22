---
title: "RFC: Eager KV prefetching in vLLM"
date: "2026-09-22"
type: "research-rfc"
experiment: "ABC — Activity-Based KV Cache Tier Placement"
status: "draft-for-review"
decision: "Proposed research and implementation gates; no new implementation authorized by this RFC"
authors:
  - "Alberto Perdomo — project owner"
  - "AI-assisted synthesis and code investigation"
code_reference:
  repository: "vllm-project/vllm"
  local_path: "/Users/aperdomo/workspace/redhat/vllm"
  commit: "1ea7c63f4af7bb4fd6f025c8db44434ab274cb51"
  branch: "main"
  worktree: "clean when inspected"
evidence:
  kind: "Synthesis of existing experiment reports, local handoff, code, and primary literature"
  new_benchmarks: false
  plots: "Published endpoint aggregates and cumulative counts; no reconstructed or downsampled time series"
historical_configuration:
  lookahead_and_working_set_common:
    model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
    tensor_parallelism: 8
    replicas: 1
    concurrency: 64
    cpu_bytes: 274877906944
    offload_spec: "TieringOffloadingSpec"
    secondary_tier: "fs on node-local NVMe"
    secondary_tier_threads: {read: 64, write: 64}
    max_model_len: 131072
    max_num_seqs: "Not explicitly set in cited reports"
    workload: "AgentX Weka; exact per-campaign fingerprints below"
  lookahead:
    vllm_version: "v0.27.0"
    gpu_memory_utilization: 0.8
    shared_memory: "300Gi"
    random_seed: 20260707
    duration_seconds: 1800
    cache_cleaning: "hostPath cleanup; report records matching 0.698% filesystem fill"
    knobs: {requests: 4, ttl_steps: 8, max_probes: 3, probe_chunks: 1024}
  working_set:
    image_digest: "sha256:1715fa537b2f754ebec797cb7a47ab9fe2d0ff061f407c2df531c4bf5b338b46"
    platform: "RHOAI 3.5.0"
    source_branch: "clean-prefetch v0.27.0 campaign"
    gpu_memory_utilization: "Not independently verified for this RFC"
    shared_memory: "Not independently verified for this RFC"
    random_seed: "Not independently verified for this RFC"
    duration_seconds: "Not independently verified for this RFC"
    cache_cleaning: "Not independently verified for this RFC"
    knobs: {candidate_ceiling: 8192, chunks_applied_per_step: 256, load_batch: 64, pending_load_limit: 64, owners: 1, eviction_budget: 8192}
  fs_job_split:
    model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
    tensor_parallelism: 8
    replicas: 1
    concurrency: 64
    cpu_bytes: 274877906944
    secondary_tier: "fs on node-local NVMe"
    secondary_tier_threads: {read: 64, write: 64}
    vllm_version: "v0.29.0"
    image: "v0.29.0-fs-job-split-v1; digest not reverified"
    source: "KV-OFFLOAD-JOB-SPLIT-HANDOFF.md, 2026-09-22"
    exact_other_knobs: "Not independently verified; do not infer from lookahead"
  llm_d_followup:
    model: "Qwen/Qwen3.6-35B-A3B"
    replicas: 8
    tensor_parallelism: 2
    cpu_bytes_per_replica: 34359738368
    secondary_tier_threads: {read: 16, write: 16}
    image: "v0.29.0-fs-job-split-v2"
    disposition: "First launch node-confounded; no accepted outcome incorporated"
---

# RFC: Eager KV prefetching in vLLM

## 1. Summary and requested decision

The project aims to make reusable KV cache available before it becomes an exposed dependency of inference. The desired outcome is lower time to first token (TTFT), shorter workflow completion time, or higher useful serving throughput, **without degrading ongoing generation or correctness**. A small repeatable benefit is sufficient; a known regression is not acceptable.

Eager prefetching includes both:

- **Non-speculative preparation:** moving exact KV for an already queued request, or loading the next layer whose execution is guaranteed.
- **Speculative preparation:** moving existing KV for a likely future continuation or workflow branch before its inference request arrives.

The project moves existing, identity-verified KV. It does not predict unknown token values or synthesize approximate KV.

**Requested decision:** review the design space and the staged validation program below. Prioritize bounded CPU→GPU staging for known waiting requests and earlier workflow/session advisories; evaluate layer-wise loading and tier pipelining when stage measurements establish their opportunity. Keep new behavior opt-in until correctness and non-regression gates pass.

This is an internal RFC and research plan, not an upstream submission, a promised speedup, or a declaration that any proposed mechanism has been implemented.

### Evidence validity verdict

**Conditionally valid evidence synthesis; no new performance proof.** Existing experiments establish that several admission-time NVMe→CPU policies did not demonstrate an end-to-end benefit. They do not establish that all eager KV prefetching is ineffective. The working-set experiment rarely achieved full readiness, and therefore was not a true perfect-residency oracle. Small performance changes from one pair must not be treated as equivalence or causality.

The September 22 filesystem handoff adds a reported serving regression to the September 7 KB, which still described that experiment as unmeasured. This RFC preserves that chronology and qualifies overstrong conclusions rather than overwriting historical reports.

## 2. Goal, scope, and expected behavior

### Goals

1. Hide exposed storage, network, and CPU→GPU loading latency using the earliest reliable signal.
2. Preserve useful KV until consumption without causing a larger miss or preemption elsewhere.
3. Make readiness explicit at the CPU, GPU, and, where supported, layer levels.
4. Bound scheduler work, outstanding bytes, destination reservations, retention time, and interference.
5. Retain the native reactive load/recompute path when preparation is late, unavailable, rejected, cancelled, or unsuccessful.
6. Produce a reproducible decision about where prefetch helps, including regimes where it should remain disabled.

### Scope

The implementation target is the native vLLM offloading and scheduler path. CPU DRAM is the primary offload tier; filesystem/NVMe/CephFS and remote tiers supply colder copies. llm-d or an application runtime may provide an early signal and select a destination, but vLLM owns allocator state and the authority to admit a transfer.

Initial correctness scope should be one explicitly supported full-attention configuration. Hybrid attention, Mamba, sliding windows, partial tails, speculative decoding, TP/PP, and cross-topology formats require their own compatibility gates before support is advertised.

### Outside the initial proposal

- A learned per-block temperature predictor.
- Model-weight prefetching or sparse/approximate attention.
- Automatic CPU-pool growth or a new GPU allocator.
- A blanket cache replacement-policy rewrite.
- Making faster filesystem I/O the project’s sole goal.
- Opening an upstream issue or PR before the relevant contribution and human-review requirements are satisfied.

## 3. Current vLLM architecture and available extension points

Code references are pinned to the inspected commit; current main may evolve independently of historical v0.27/v0.29 benchmarks.

The native retrieval sequence is:

1. Request registration creates connector state; it does not itself retrieve KV.
2. The scheduler examines a waiting request and checks the local GPU prefix.
3. The offloading connector scans exact chunk keys. CPU hits are immediately known. Secondary existence checks run asynchronously.
4. A secondary hit reserves a CPU destination; end-of-step batching submits secondary→CPU promotion.
5. On completed promotion, the CPU chunks become readable.
6. GPU allocation and CPU pinning create a worker load job. The request waits for asynchronous reception.
7. After reception completes, the request can enter computation and valid GPU cache content can be published.

The key observations are:

| Code fact | Consequence for eager loading |
|---|---|
| The prefix scan continues across RETRY and stops on a true MISS | Ordinary demand lookup already scans ahead over known keys; post-miss same-request read-ahead is not the missing mechanism |
| A deferred lookup causes the admission loop to continue; allocation failure causes it to break | Lookahead mainly adds coverage beyond admission barriers, not a universal removal of head-of-line blocking |
| CPU→GPU loading is asynchronous, but its admission is reached through compute-budget and capacity gates | There may be an opportunity to separate transfer admission from compute admission when HBM remains available |
| CPU chunks are pinned during transfers; GPU destinations are withheld from cache publication until safe | Early and partial preparation must preserve these lifetime rules |
| Native filesystem submission enqueues one task per job; the C batch loops serially with the GIL released | More pool threads currently parallelize different jobs, not blocks within a job |
| Whole-job completion controls when promoted chunks become readable | Job splitting alone does not pipeline completed chunks into the next tier |
| Native layer-load hooks are no-ops | Layer-wise prefetch needs actual per-layer completion and scheduling semantics |
| Python layer-transfer synchronization requires piecewise CUDA graphs | Graph-mode overhead may outweigh loading savings |
| Typed KV hints reach request/offload context; the KVCR adapter forwards hints | Transport exists, but native filesystem/HBM prefetch execution is not supplied by the envelope itself |
| Optional secondary-store backpressure exists on this main | Older “unconditional cascade” descriptions need qualification for current code |

CPU memory is shared between scheduler and workers through an mmap region. Substantial planning or I/O coordination can potentially run outside the scheduler’s Python execution path, but moving work to a different process does not eliminate memory, PCIe, or GPU contention.

The native filesystem tier has no ordinary capacity-based eviction. This must not be generalized to every secondary backend: the current tree also contains KVCR and backend-specific policies.

### Source map

- [Prefix lookup and request registration](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L671)
- [Waiting-request admission](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/vllm/v1/core/sched/scheduler.py#L859)
- [Tiering promotion and completion](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/vllm/v1/kv_offload/tiering/manager.py)
- [Filesystem job submission](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/vllm/v1/kv_offload/tiering/fs/manager.py#L223), [C batch loop](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/csrc/fs_io.cpp#L272)
- [CPU→GPU worker](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/vllm/v1/kv_offload/cpu/gpu_worker.py)
- [Layer hooks and graph-mode contract](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/vllm/distributed/kv_transfer/kv_connector/v1/base.py#L647)
- [KV hint envelope](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/vllm/v1/kv_hints/protocol.py), [KVCR hint forwarding](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/vllm/v1/kv_offload/tiering/kvcr/manager.py#L577)

## 4. Prior evidence and what it supports

This section uses the existing KB reports and the dated local handoff. It does not reanalyse raw MLflow telemetry. Run-level values and cumulative counters below retain the granularity at which those sources report them. No new temporal aggregation, smoothing, synthetic samples, or cross-campaign pooling is performed. The linked original reports remain the detailed evidence record.

### 4.1 Headline outcomes

The **mechanism baseline is native reactive offloading with the feature disabled**, not no-offload: this isolates eager movement from the benefit of offloading itself. No-offload is retained as a contextual reference.

| Campaign / arm | Request rate (req/s) | Output rate (tokens/s) | Mean / p95 TTFT (s) | Mean ITL (ms) | Mean E2E (s) | Disposition |
|---|---:|---:|---:|---:|---:|---|
| No-offload contextual replicates | 0.1304 / 0.1315 / 0.1283; outlier 0.0217 | Not reported here | Not reported here | Not reported here | Not reported here | Separate runs; no cross-campaign delta |
| Lookahead v2 reactive control | 0.6745 | Not reported here | 8.213 / 24.257 | 55.30 | Not reported here | Accepted source pair; n=1 |
| Lookahead v2, bounded 1024-chunk probe | 0.6630 | Not reported here | 8.665 / 24.252 | 55.51 | Not reported here | No demonstrated benefit; n=1 |
| Working-set reactive control | 0.6750 | 452.4 | 8.313 / 23.457 | 56.51 | 44.005 | Different-node comparison |
| Working-set treatment | 0.6761 | 457.1 | 8.158 / 23.716 | 54.31 | 43.638 | Mechanism active; full readiness rarely achieved |

Lookahead source: [[Research/ABC/Reports/2026-09-07 - Lookahead demand staging v1 to v3]]. Working-set source: [[Research/ABC/Reports/2026-08-23 - Working-set oracle AgentX first comparison]].

Lookahead runs were both FINISHED, with total recorded durations of roughly 57/58 minutes and a configured 1800-second benchmark phase. Completion of an MLflow run is not itself proof of complete workflow drain. Working-set client records contain 1242 control and 1244 treatment requests; both clients reported a known final cancelled-credit timeout despite profile records listing no cancelled requests. Completed-session counts and comparable error-rate aggregates are not available in this synthesis and must not be inferred as zero.

For the accepted lookahead pair, relative deltas are −1.70% request throughput, +5.50% mean TTFT, −0.02% p95 TTFT, and +0.38% mean ITL, calculated from the displayed rounded source values. No causal deltas are claimed for the different-node working-set pair. The earlier v1 lookahead result is preserved as a conditionally valid reported regression: −10.0% request throughput and +40.4% mean TTFT.

**Figure 1** compares the accepted lookahead pair’s request outcomes. Provenance: endpoint aggregates in the September 7 report, runs L0/L1 in the registry. These are point estimates from one pair, without confidence intervals. Positive latency changes are worse; positive throughput changes would be better.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 1 — Lookahead v2: request outcomes versus its paired control",
  "width": 640,
  "height": 180,
  "data": {
    "values": [
      {
        "metric": "Request throughput",
        "control": 0.6745,
        "treatment": 0.663,
        "unit": "requests/s",
        "delta_pct": -1.7049666419569953
      },
      {
        "metric": "Mean TTFT",
        "control": 8213,
        "treatment": 8665,
        "unit": "ms",
        "delta_pct": 5.503470108364783
      },
      {
        "metric": "p95 TTFT",
        "control": 24257,
        "treatment": 24252,
        "unit": "ms",
        "delta_pct": -0.020612606670245004
      },
      {
        "metric": "Mean ITL",
        "control": 55.3,
        "treatment": 55.51,
        "unit": "ms",
        "delta_pct": 0.37974683544304
      }
    ]
  },
  "mark": {
    "type": "bar"
  },
  "encoding": {
    "y": {
      "field": "metric",
      "type": "nominal",
      "sort": null,
      "title": "Request outcome"
    },
    "x": {
      "field": "delta_pct",
      "type": "quantitative",
      "title": "Treatment change relative to control (%)",
      "scale": {
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
    },
    "tooltip": [
      {
        "field": "metric"
      },
      {
        "field": "control"
      },
      {
        "field": "treatment"
      },
      {
        "field": "unit"
      },
      {
        "field": "delta_pct",
        "format": ".2f",
        "title": "Relative change (%)"
      }
    ]
  }
}
~~~

Figure 1 supports “no demonstrated serving improvement.” It does not prove statistical equivalence, and the higher mean TTFT prevents calling the treatment universally harmless.

### 4.2 Earlier lookup resolution was insufficient

In that same lookahead pair, the reported mean connector deferred-lookup interval fell from 3.4356 to 2.3211 seconds. Running requests changed from 17.65 to 17.98, waiting from 4.00 to 3.93, and GPU KV occupancy from 0.6117 to 0.6206. Meanwhile, synchronous tier-lookup p99 increased from 0.2657 to 0.4808 seconds.

**Figure 2** retains these distinct source-report statistics and compares relative changes. Provenance: September 7 report, mechanism table. The “means” are the report’s summaries; no new time-series average is computed here. GPU KV occupancy is not GPU SM utilization. The p99 is not a mean.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 2 — Earlier lookup resolution did not establish more serving capacity",
  "width": 640,
  "height": 215,
  "data": {
    "values": [
      {
        "metric": "Connector deferred-lookup delay",
        "control": 3.4356,
        "treatment": 2.3211,
        "unit": "s",
        "delta_pct": -32.439748515543144
      },
      {
        "metric": "Running requests",
        "control": 17.65,
        "treatment": 17.98,
        "unit": "requests",
        "delta_pct": 1.8696883852691304
      },
      {
        "metric": "Waiting requests",
        "control": 4,
        "treatment": 3.93,
        "unit": "requests",
        "delta_pct": -1.749999999999996
      },
      {
        "metric": "GPU KV occupancy",
        "control": 0.6117,
        "treatment": 0.6206,
        "unit": "fraction",
        "delta_pct": 1.4549615824750672
      },
      {
        "metric": "Synchronous tier lookup p99",
        "control": 0.2657,
        "treatment": 0.4808,
        "unit": "s",
        "delta_pct": 80.9559653744825
      }
    ]
  },
  "mark": {
    "type": "bar"
  },
  "encoding": {
    "y": {
      "field": "metric",
      "type": "nominal",
      "sort": null,
      "title": "Mechanism observation"
    },
    "x": {
      "field": "delta_pct",
      "type": "quantitative",
      "title": "Treatment change relative to control (%)",
      "scale": {
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
    },
    "tooltip": [
      {
        "field": "metric"
      },
      {
        "field": "control"
      },
      {
        "field": "treatment"
      },
      {
        "field": "unit"
      },
      {
        "field": "delta_pct",
        "format": ".2f",
        "title": "Relative change (%)"
      }
    ]
  }
}
~~~

Figure 2 is consistent with retrieval delay being overlapped with other admission constraints. It does not identify GPU capacity as the exclusive cause: running count alone cannot establish causality. Shortening demand I/O is also subject to this critical-path question.

The report says CPU occupancy was only 3–4% and the protection gate never fired. Safety under CPU pressure therefore remains untested. The remaining synchronous lookup tail also means “nearly free” is too strong.

### 4.3 Chunk usefulness is different from request readiness

The working-set treatment promoted 676,388 chunks, of which 673,320 were eventually useful. Only 20 of 2,638 request intents were fully ready at first connector lookup; 2,618 still deferred. Only 934 intents obtained the single speculative owner.

**Figure 3** contrasts two explicitly different populations. Provenance: exact cumulative counts in the August 23 report. The percentages are computed directly from the shown numerators and denominators; these bars are not a common partition.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 3 — Eventually useful chunks versus requests ready at first lookup",
  "width": 640,
  "height": 130,
  "data": {
    "values": [
      {
        "population": "Promoted chunks eventually useful",
        "numerator": 673320,
        "denominator": 676388,
        "percent": 99.54641418830612
      },
      {
        "population": "Request intents fully ready at first lookup",
        "numerator": 20,
        "denominator": 2638,
        "percent": 0.7581501137225171
      }
    ]
  },
  "mark": {
    "type": "bar"
  },
  "encoding": {
    "y": {
      "field": "population",
      "type": "nominal",
      "sort": null,
      "title": "Separate populations"
    },
    "x": {
      "field": "percent",
      "type": "quantitative",
      "title": "Share of the stated population (%)",
      "scale": {
        "domain": [
          0,
          100
        ],
        "zero": true
      }
    },
    "color": {
      "field": "population",
      "type": "nominal",
      "scale": {
        "scheme": "category10"
      },
      "legend": null
    },
    "tooltip": [
      {
        "field": "population"
      },
      {
        "field": "numerator",
        "title": "Observed count"
      },
      {
        "field": "denominator",
        "title": "Population count"
      },
      {
        "field": "percent",
        "format": ".2f",
        "title": "Share (%)"
      }
    ]
  }
}
~~~

Figure 3 explains why an accurate selection can still miss its deadline. The manager’s separate count of 754 completed plans is not authoritative full readiness: its target shrank after an admission-time source miss.

Every proactive promotion displaced an ordinary CPU block. At least 131,276 victims were later demanded; 438,312 victim outcomes were untracked. The reported 19.41% eviction-regret fraction is a lower bound, not a complete cost estimate.

This result rejects that admission-time, one-owner implementation as a successful readiness oracle. It does not reject a capacity-matched experiment that actually makes the selected working set ready before demand. Full readiness is one useful measure; valid partial-prefix reuse can also save recomputation and must be measured separately.

The preceding first-N campaign found that 98.44% of useful promotions were late at C32. Its admission horizon was too short to hide substantial transfers. See [[Research/ABC/00 - Index]] and the clean-prefetch reports linked there.

### 4.4 Filesystem parallelism: faster device work, unresolved serving economics

The handoff reports a standalone 512 × 2 MiB read taking 372 ms as one task and 160 ms when split across threads: a 2.33× batch speedup. The code history confirms that #49152 replaced per-block tasks with one batched task per job.

The later RHOAI serving comparison reportedly produced:

| Metric | Reported treatment change |
|---|---:|
| Prefill interval p50 / p90 / p99 | −2.5% / −2.7% / −2.3% |
| Decode interval p50 / p90 / p99 | +20.1% / +6.7% / +3.3% |
| Preemption rate | +181% |
| Output-token rate | −6.8% |
| Mean / p95 ITL | +13.8% / +13.5% |
| Request throughput | −2.8% |

Source: September 22 handoff, runs F0/F1. This is reported single-pair evidence, not independently reconstructed telemetry. It is a regression signal that prevents treating the implementation as ready to ship.

**Metric correction.** vLLM’s prefill interval is first SCHEDULED to first token; native asynchronous loading occurs before that scheduling event. Decode interval is first token to last token and includes preemptions. A smaller prefill interval is therefore not direct proof of faster filesystem reads causing a request-level gain. See [metric definitions](https://github.com/vllm-project/vllm/blob/1ea7c63f4af7bb4fd6f025c8db44434ab274cb51/vllm/v1/metrics/stats.py#L542); the v0.29.0 definitions were also checked.

Extra Python worker activity contending for the GIL is a plausible explanation for the regression, **not an isolated observation**. Load-only versus load-and-store splitting is an attribution experiment. If it fails, that rejects the tested configuration; it is not a mathematical proof that every thread-pool design is impossible.

The llm-d follow-up changes the model, replica count, CPU capacity, thread count, and node placement. Its first launch used different nodes across arms. No accepted result from that run is used here.

## 5. Related systems and applicability

These are mechanism references, not speedup forecasts for Nemotron or this vLLM revision.

| System | Relevant contribution | Applicability and limitation |
|---|---|---|
| [KVFlow](https://arxiv.org/html/2507.07400v1) | Uses workflow execution distance for retention and CPU→GPU prefetch; skips requests whose cache is still loading | Supports earlier intent and coordinated readiness. Its evaluation excludes regimes where active requests consume all reusable-cache capacity |
| [Symphony](https://arxiv.org/html/2412.16434v1) | Pre-arrival advisories coordinate destination selection and cache movement; layer-aware preparation prioritizes earlier dependencies | Supports workflow/session signals. Requires advance notice; lower-layer residency does not map directly onto vLLM’s existing allocation granularity |
| [CachedAttention](https://arxiv.org/html/2403.19708) | Overlaps loading later layers with computation and uses scheduler knowledge for disk→CPU fetching | Supports deterministic layer prefetch. Available computation on the new suffix limits overlap |
| [Strata](https://arxiv.org/html/2508.18572v1) | Couples efficient KV transfer/layouts with cache-aware batch formation and best-effort storage prefetch | Supports treating loading and computation jointly. GPU-assisted I/O consumes resources and its paper measures interference |
| [PBKV](https://arxiv.org/html/2605.06472v1) | Predicts workflow activity and bounds speculative preparation by space and bandwidth opportunity | Supports conservative speculation. Its retention and prefetch effects are distinct; it does not establish a generic no-regression guarantee |

The [session-hint RFC #52113](https://github.com/vllm-project/vllm/issues/52113), [programmatic management RFC #51428](https://github.com/vllm-project/vllm/issues/51428), and [programmable-policy RFC #57103](https://github.com/vllm-project/vllm/issues/57103) cover related interfaces. The existing typed envelope should be reused or extended through upstream coordination, not duplicated with another incompatible hint format.

LMCache’s [separate-process architecture](https://docs.lmcache.ai/mp/architecture.html) is a useful reference for isolating storage coordination. Its lookup-triggered prefetch terminology does not imply prediction of a future request.

## 6. Candidate mechanisms

### A. Early workflow/session preparation

**Trigger:** a predecessor starts, a tool wait begins, or a known continuation is scheduled.

Resolve an immutable, already-materialized prefix; verify its available source copies; prepare CPU residency early and GPU residency nearer to use. Unknown tool-output tokens do not prevent preparing the unchanged preceding prefix.

An advisory should identify logical content or an existing session association, intended destination, expected-use window, priority, expiry, and resource bounds. These are conceptual semantics, not a new public JSON schema.

The destination selected for preparation should also receive the eventual inference request, subject to load-balance constraints. A reroute must invalidate or reconcile the old plan. Without this coordination, llm-d may consume bandwidth on a replica that never serves the continuation.

**Why it might work:** the preparation horizon starts before inference admission.

**What could defeat it:** late/incorrect hints, unstable prefix identity, unavailable source data, wasted retention, or destination load imbalance.

### B. Bounded GPU staging for known waiting requests

**Trigger:** exact CPU-ready KV exists for an upcoming request, while compute admission is closed but sufficient HBM remains.

Separate transfer admission from compute admission. Reserve destinations, pin source chunks, and use the existing bulk CPU→GPU path without adding the request to RUNNING until ready. Count all reservations against normal capacity.

**Why it might work:** moves beyond CPU-only readiness while retaining coarse transfer granularity and the existing graph mode.

**What could defeat it:** active requests already consume HBM, forecast reservations prolong pressure, or the transfer slows generation. The previous lookahead result did not test this destination stage.

### C. Layer-wise deterministic prefetch

**Trigger:** execution order guarantees the next layer.

Begin when the initial required layer is ready; transfer later layers while earlier layers compute. This requires per-layer dependencies and completion rather than the native whole-request reception barrier.

**Why it might work:** obtains overlap even when there is little request-level lead time.

**What could defeat it:** short new suffixes, PCIe or memory contention, fragmented transfers, and piecewise-graph overhead. Loading fewer layers initially does not automatically reduce HBM allocation. Current native partial-tail/group semantics cannot be assumed compatible without verification.

### D. Pipelined secondary→CPU→GPU promotion

**Trigger:** an initial group of exact chunks has completed its storage read.

Publish bounded partial completion to the transfer planner, pin the completed CPU destinations, and start CPU→GPU transfer while subsequent storage reads proceed. Unfinished data remains unavailable to computation and unrelated prefix-cache consumers.

**Why it might work:** overlaps two transfer stages and removes a whole-job barrier. It is especially worth evaluating on a slower tier such as CephFS.

**What could defeat it:** excessive bookkeeping, small transfer inefficiency, scarce destinations, partial-failure handling, or a scheduler path that still waits for the complete lookup before allowing any progress.

Faster filesystem service is an enabling optimization for A–D. It supplies neither an earlier trigger nor a readiness contract by itself.

## 7. Proposed common contract

Keep mechanism selection separate from execution, while giving the engine authority over resources.

1. **Intent:** compact, immutable logical target and expected-use window, from a queued request or external advisory.
2. **Plan:** source residency, missing bytes, target tier, transfer ordering, deadline, and bounded resource reservation.
3. **Execution:** asynchronous, bounded transfers with ordinary demand priority.
4. **Readiness:** explicit CPU, GPU, and optionally per-layer completion.
5. **Consumption or expiry:** normal inference consumes valid content; unused work releases its reservations and retention protection.

A conceptual action lifecycle is:

~~~text
received → resolved → admitted → transferring → ready → consumed
     └──────────── rejected / expired / cancelled / failed
~~~

Cancellation stops unsubmitted work. In-flight destinations must remain protected until I/O completion makes release safe; cancellation is not permission to reuse memory immediately. Shared requests reference common work rather than cancelling each other’s data.

Required invariants:

- Exact hash/model/configuration identity and group-specific prefix rules remain authoritative.
- The original target never silently shrinks to make a completion metric look successful.
- Speculative preparation cannot evict active/pinned state.
- Capacity reserved for prefetch is visible to the normal allocator.
- Initial speculative policy uses a bounded explicit allowance; “evictable” is not synonymous with “free of opportunity cost.”
- No global cache publication before the relevant data is valid.
- Failures preserve the largest safe usable result and use reactive fallback.
- Source and destination lifetime tracking survives aborts, preemption, reset, shutdown, and multi-rank completion.
- Planning cannot synchronously scan unbounded keys in the scheduler.
- An idle engine must still receive control work and finish transfers; avoid indefinite busy polling as the default control mechanism.

A proposed admission heuristic is:

$$
\text{expected exposed stall saved}
>
\text{expected interference} + \text{eviction regret} + \text{planning cost}.
$$

This is a decision model, not a proven guarantee. Estimate it in time units and retain measured error bounds. Use byte- and duration-based transfer budgets; a fixed chunk count has different costs across models.

## 8. Experimental plan and falsification

Experiments are sequential gates. Do not build all four mechanisms at once.

| Gate | Comparison | Question and decision |
|---|---|---|
| E0: observability and A/A | Stock reactive vs same-build feature-off/no-op path | Is measurement stable and is the inactive mechanism negligible? |
| E1: readiness opportunity | Reactive vs actual CPU-ready vs actual GPU-ready controls | Which stage has exposed value at matched effective capacity? |
| E2: queued GPU staging | Reactive vs B, exact queued keys | Does earlier GPU readiness help when HBM headroom exists? |
| E3: advisory horizon | Reactive vs A with exact future prefix and controlled lead times | How early must intent arrive for a positive net result? |
| E4: execution overlap | Bulk transfer vs C, plus graph-mode-only control | Do layer savings exceed graph and synchronization costs? |
| E5: tier pipeline | Whole-job vs D on measured storage-sensitive workload | Is the inter-tier barrier material? |
| E6: realistic signals | Oracle advisories vs actual workflow events, then optional predictions | Does practical signal quality recover enough oracle value? |

For E1, make the oracle condition real: record the complete selected source target and verify its residency before demand. Charge prepositioned data to the same CPU/HBM capacities; do not compare an extra-capacity treatment against a constrained baseline. Prewarming can change interference and placement, so report it as an opportunity control with those limitations, not an implementable free operation.

For E3, a proposed lead-time sweep is 0, 0.25, 0.5, 1, 2, 4, and 8 seconds. These are proposed settings, not measured results. Use exact advisory content first to separate execution feasibility from prediction quality. Track false-positive advisories, expiry, and rerouting later.

Include low-queue and pressured regimes (C32/C64 are historical starting points, not universal operating points), CPU-ready and secondary-resident hits, short and long new suffixes, and a no-external-hit control. First fix one model/topology; use a second model and multi-replica llm-d only after the mechanism is understood. Explicitly verify all participating KV groups for hybrid models.

Use same-node, counterbalanced comparisons, identical immutable images except when a graph-mode ablation requires otherwise, and at least three paired repetitions per accepted arm. Continue repetitions when uncertainty is too wide for the claimed small benefit. Preserve the same seed, cache preparation, request mix, routing policy, drain policy, storage backend, and contention conditions. Record actual image digests and rendered arguments.

CephFS is a proposed discriminating regime, not a guaranteed winner. Low average NVMe utilization alone cannot prove that storage latency never affects a request; inspect event-aligned queue/service time and tails.

## 9. Telemetry and acceptance

### Required evidence

| Question | Measurements |
|---|---|
| Did the signal arrive early enough? | Intent timestamp, expected-use window, actual first demand, immutable target bytes |
| Where was time spent? | Lookup enqueue/start/end, storage submission/start/completion, CPU-ready, GPU allocation, GPU-copy start/end, first compute, first token |
| Did preparation remove exposed delay? | Per-request readiness at demand/admission, valid prefix coverage, per-layer waits where applicable |
| What did it cost? | Scheduler-step duration, Python/GIL or host profiles, transfer queue/service time, CPU memory bandwidth, PCIe traffic, GPU SM/HBM activity |
| Did capacity suffer? | Free/evictable/pinned/staging bytes, reservation age, running/waiting requests, preemption and recomputation |
| Was it useful overall? | TTFT, ITL, E2E/workflow completion, request/output-token throughput, completed sessions, failures and cancellations |
| Was staging wasteful? | Consumed, late, expired, failed, duplicate, restaged bytes and complete eviction outcomes |

Collect native temporal samples in subsequent experiment reports. Do not downsample merely to fit a chart; split comparison appendices if necessary. Separate request quantiles from averages of rolling histogram quantiles.

The filesystem tier’s own read/write time and byte counters must be collected. The CPU↔GPU transfer metrics describe a different hop. Job timing must have documented semantics; summing concurrent task durations is not job elapsed time. Even summed job spans are not device busy time.

For NVMe collect bytes, operations, queue depth, latency, busy time, and the actual KV directory occupancy. Whole-filesystem usage can include the model cache. For CephFS additionally collect client operation latency, throughput, MDS/OSD or filesystem health, capacity, and store-refusal warnings. State unavailable telemetry explicitly.

### Acceptance criteria

- Correctness passes the existing relevant suites plus behavior tests for new lifetime/failure cases and representative model evaluation.
- Feature-off/no-op behavior is validated independently.
- At least one supported regime shows a repeatable request/workflow benefit attributable to earlier useful readiness.
- No supported evaluated regime has an established throughput, ITL, preemption, or error regression.
- Statistical precision is sufficient to assess the claimed benefit and non-regression. “Not statistically significant” is not evidence of safety.
- Resource use remains bounded; expired work drains; no starvation or persistent speculative pressure appears.
- Retention-only, transfer-only, and prediction contributions are isolated where combined.
- Default enablement is a separate decision. A mechanism useful only in a known regime remains opt-in.

“No regression” is the objective, not a proof that can be obtained for every unseen workload. Before running, state the practical detection precision and confidence procedure; that measurement tolerance is not authorization to accept known harm. Previous 5%/3% or 10%/5% research gates were campaign-specific and are not silently adopted as this RFC’s minimum upside. A smaller demonstrable gain remains acceptable.

Stop or redirect a mechanism when actual readiness fails to improve outcomes, overhead exceeds savings, sufficient lead time cannot be obtained, or capacity cannot be preserved. A documented negative result is a valid research deliverable.

## 10. Expected outputs

1. **Reviewed RFC and decision record:** selected mechanism, supported regimes, explicitly rejected alternatives, and unresolved questions.
2. **Reproducible evidence package:** immutable run registry, exact configuration fingerprints, controls, native-resolution telemetry, outcome and mechanism plots, and acceptance decisions.
3. **Readiness instrumentation:** stable definitions for target coverage and each transfer/admission stage, including bounded-resource and failure accounting.
4. **Minimal opt-in prototype, after the research gate:** ideally B for the first engine-only test, with A as the broader workflow direction. C/D are conditional extensions.
5. **Integration contract:** reuse of typed hints, logical prefix identity, destination coordination, expiry, cancellation, completion, and reactive fallback.
6. **Upstream-ready change only if justified:** focused implementation, relevant correctness/model tests, measured serving impact, explicit limitations, duplicate-work checks, AI disclosure, and human review of every changed line.

No prototype, benchmark launch, deployment, or upstream submission is performed by creating this document. No numerical speedup is promised.

## 11. Risks and review questions

- Does this workload expose storage time, CPU→GPU time, GPU allocation time, or mostly compute capacity?
- Can application events identify a stable existing prefix early enough, and is the eventual destination stable?
- Should the initial delivery be B alone, or an advisory-driven A/B combination?
- What explicit staging allowance is affordable without displacing more valuable data?
- Can layer synchronization preserve the desired graph behavior, or does piecewise overhead reject C?
- Which attention groups and topology combinations can be supported without weakening validity/publication rules?
- Is a separate control process worthwhile after measuring actual scheduler/GIL cost?
- How should out-of-band hints be accepted while no inference request is active, and which upstream interface owns this?
- When is valid partial-prefix use preferable to waiting for a longer external hit?
- What experiment precision is needed to accept a small improvement under the no-regression objective?

## 12. Run registry, provenance, and code-state caveats

MLflow links point to the original evidence; they are not claims that raw telemetry was reprocessed for this RFC. A fresh read-only artifact inventory was attempted during drafting but failed because the MLflow hostname could not be resolved. The figures therefore reproduce documented report-level aggregates and counters, not newly downloaded raw samples.

| ID | Role | Run |
|---|---|---|
| L0 | Accepted lookahead v2 control | [0e982c4a8094475fb84dfd63b9b9da0b](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/0e982c4a8094475fb84dfd63b9b9da0b?workspace=benchflow) |
| L1 | Accepted lookahead v2 @1024 treatment | [6c4bdb0195524b0c9109b3075edf79cb](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/6c4bdb0195524b0c9109b3075edf79cb?workspace=benchflow) |
| V0 | Earlier lookahead image, feature off | [1573078c65f743c9a0bb3ce72be08237](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/1573078c65f743c9a0bb3ce72be08237?workspace=benchflow) |
| V1 | Earlier v1 lookahead regression | [ffe1170ac7d54fc5b0b40e28ae21afb7](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/ffe1170ac7d54fc5b0b40e28ae21afb7?workspace=benchflow) |
| W0 | Working-set reactive control | [a34cca262119453a9837a2531c79c3de](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/a34cca262119453a9837a2531c79c3de?workspace=benchflow) |
| W1 | Working-set treatment; not a perfect-residency oracle | [39a70a1b52e241bcb48abe5338d56110](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/39a70a1b52e241bcb48abe5338d56110?workspace=benchflow) |
| F0 | v0.29.0 filesystem-split control, handoff evidence | [53c79be66eaa45789a8b7e57ddf81b60](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/53c79be66eaa45789a8b7e57ddf81b60?workspace=benchflow) |
| F1 | Filesystem load-and-store split, reported regression | [b27ecd93003b4707bf911c9bb4d3e42d](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/b27ecd93003b4707bf911c9bb4d3e42d?workspace=benchflow) |
| N0–N2 | Contextual no-offload replicates | [d7982fea41bc4fdba7eec32ca99e8604](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/d7982fea41bc4fdba7eec32ca99e8604?workspace=benchflow), [f5bd52f08c17455897d740ebdf1f9ebf](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f5bd52f08c17455897d740ebdf1f9ebf?workspace=benchflow), [65a2930350254d459ff8cadce67f22f0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/65a2930350254d459ff8cadce67f22f0?workspace=benchflow) |
| NX | Identical-config no-offload outlier | [5cffd9d9c0654cf5bdbf4640f24a30dd](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5cffd9d9c0654cf5bdbf4640f24a30dd?workspace=benchflow) |

The local handoff was read at /Users/aperdomo/workspace/redhat/KV-OFFLOAD-JOB-SPLIT-HANDOFF.md, dated September 22. Its uncommitted job-split implementation is not present in the inspected clean checkout; the local job-split branch points to the v0.29.0 base. Historical test-pass counts apply to the handoff’s implementation, not to code created or validated by this RFC.

Related durable records:

- [[Research/ABC/00 - Index]]
- [[Research/ABC/Methodology/01 - Experiment Definition]]
- [[Research/ABC/Methodology/08 - Lookahead demand staging design investigation]]
- [[Research/ABC/2026-08-21 - Independent research audit and redirection for speculative KV prefetching]]
- [[Research/ABC/Reports/2026-09-07 - Lookahead demand staging v1 to v3]]
- [[Research/ABC/Reports/2026-08-23 - Working-set oracle AgentX first comparison]]
- [[Research/ABC/Future-Value Placement/00 - Index]]
- [[Research/ABC/Continuation Readiness/00 - Index]]
- [[Engineering/Learnings/vLLM KV offload retrieval path - lookup, promotion, and load]]

The placement and continuation studies establish interesting offline information value but no validated general live policy. They support investigating earlier lifecycle information; they do not justify reviving failed fixed-N, whole-bundle, or coarse-lineage policies without a new discriminating experiment.