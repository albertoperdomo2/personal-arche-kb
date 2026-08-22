---
title: "2026-08-22 - Clean-prefetch v2 AgentX concurrency-64 comparison"
date: "2026-08-22"
type: "experiment-report"
experiment: "ABC"
status: "complete"
verdict: "valid negative policy result"
---

# Clean-prefetch v2 AgentX concurrency-64 comparison

## Executive conclusion

This is a **valid negative result for the current fixed-64 admission-prefetch policy**, not evidence that the v2 repair is broken.

The v2 demand cutoff and ordering repair worked: all 4,670 submitted chunks completed, no chunk was submitted after demand, admission-to-CPU-ready p90 fell from roughly 740 seconds in v1 to 1.70 seconds, and no load failures or late events were observed. Nevertheless, treatment did not improve the workload: request throughput fell 0.56%, total-token throughput fell 0.73%, mean TTFT rose 2.14%, and p95 TTFT rose 4.05%.

The fundamental mismatch is request-level coverage. The policy stages at most 64 chunks (1,024 tokens) for one request at a time, while the average external prefix was about 41,153 tokens, or roughly 2,572 chunks. A prefetched chunk is currently counted as “useful” when demand observes a CPU hit, even if thousands of later chunks remain only on NVMe and the request must still enter the ordinary full-prefix promotion path. Thus 4,550 useful chunk hits do **not** mean that 4,550 transfers were removed from the request critical path.

The next experiment should not be an N sweep. First measure request-level completeness and run a one-request full-working-set oracle. If making an entire next request CPU-resident before its first connector lookup does not reduce deferred lookup time and TTFT, admission-time NVMe→CPU prefetch at this layer should be stopped.

## Runs and comparability

| Role | MLflow run | Deployment profile |
|---|---|---|
| Treatment | [c03bf0c79d6844da8069162633bb3d94](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/c03bf0c79d6844da8069162633bb3d94?workspace=benchflow) | `clean-prefetch-cpu-kv-offload-nvme` |
| Control | [24df61e44ac34ede8b94d42b23a8cb58](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/359/runs/24df61e44ac34ede8b94d42b23a8cb58?workspace=benchflow) | `multi-tier-offloading-nvme` |

Both cells used:

- image `quay.io/rh-ee-aperdomo/vllm:v0.27.0-clean-prefetch-v2`;
- immutable digest `sha256:595bdff0726410bee8d3e50d4e5ee1e519223ba912dfca62d37a1fa0e2dcd611`;
- the same H100 model node, `diadochos-hqxzk-gpu-h100-mt46x`;
- TP=8, one replica, FP8 Nemotron, 256 GiB CPU KV tier, local NVMe, 64 read/write threads;
- AgentX/Weka trace, concurrency 64, seed 20260707, default scheduler concurrency;
- RHOAI 3.5.

Treatment used `admission_prefetch_chunks=64`, `max_candidate_chunks=1024`, `apply_chunks_per_step=64`, `load_batch_size=8`, `max_pending_intents=64`, `max_tracked_prefetches=64`, `eviction_history_capacity=4096`, and source tier 0. The control had no admission-prefetch configuration.

Warmup was exceptionally close: mean TTFT was 28,982.47 ms in treatment and 28,987.51 ms in control; durations were 827.895 and 827.825 seconds. This removes the node/configuration divergence that invalidated several earlier comparisons.

## End-to-end result

| Metric | Control | Treatment | Treatment delta |
|---|---:|---:|---:|
| Completed requests | 1,242 | 1,235 | -7 |
| Request throughput | 0.6750 req/s | 0.6712 req/s | -0.56% |
| Total-token throughput | 43,083.6 tok/s | 42,771.1 tok/s | -0.73% |
| Mean TTFT | 8,299.2 ms | 8,476.7 ms | +2.14% |
| Median TTFT | 6,297.2 ms | 5,997.4 ms | -4.76% |
| p95 TTFT | 23,940.6 ms | 24,909.2 ms | +4.05% |
| Mean request latency | 43,709.8 ms | 44,269.9 ms | +1.28% |
| p95 request latency | 134,054.3 ms | 138,064.3 ms | +2.99% |
| Mean ITL | 54.881 ms | 55.095 ms | +0.39% |
| p95 ITL | 90.885 ms | 86.351 ms | -4.99% |

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Treatment change versus control (lower is better except throughput)","width":700,"height":300,"data":{"values":[{"metric":"Request throughput","delta":-0.56,"class":"Throughput"},{"metric":"Total-token throughput","delta":-0.73,"class":"Throughput"},{"metric":"Mean TTFT","delta":2.14,"class":"Latency"},{"metric":"Median TTFT","delta":-4.76,"class":"Latency"},{"metric":"p95 TTFT","delta":4.05,"class":"Latency"},{"metric":"Mean request latency","delta":1.28,"class":"Latency"},{"metric":"p95 request latency","delta":2.99,"class":"Latency"},{"metric":"Mean ITL","delta":0.39,"class":"Latency"},{"metric":"p95 ITL","delta":-4.99,"class":"Latency"}]},"mark":{"type":"bar"},"encoding":{"y":{"field":"metric","type":"nominal","sort":null,"title":null},"x":{"field":"delta","type":"quantitative","title":"Treatment delta (%)"},"color":{"field":"class","type":"nominal","scale":{"domain":["Latency","Throughput"],"range":["#d73027","#4575b4"]}},"tooltip":[{"field":"metric","type":"nominal"},{"field":"delta","type":"quantitative","format":".2f","title":"Delta (%)"}]},"config":{"view":{"stroke":null}}}
~~~

The mixed medians and tails do not establish benefit. The throughput and mean/tail TTFT outcomes are negative, while both runs are a single pair.

## Did v2 mechanically work?

Yes.

| Mechanism counter/timing | Observed |
|---|---:|
| Admission intents | 1,248 |
| Candidate chunks | 1,198,515 |
| Source-resident candidates | 1,122,638 |
| Redundant candidates | 273,771 |
| Cancelled candidates | 827,288 |
| Cancelled at first lookup | 326,947 |
| Expired before submit | 497,717 |
| Source miss | 45 |
| Submitted | 4,670 |
| Promoted | 4,670 |
| Useful | 4,550 |
| Wasted | 64 |
| Late | 0 |
| Submitted after demand | 0 |
| Evicted for prefetch | 4,670 |
| Eviction regret | 2,684 |
| Admission-to-ready p50 / p90 / p95 | 0.094 / 1.700 / 3.835 s |
| Source probe mean / p90 | 12.97 / 22.35 ms |
| Scheduler apply mean | 0.048 ms |

The terminal counters straddle the measurement boundary, so a small number of promoted chunks remained tracked. They are not evidence of an accounting failure.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"V2 mechanism outcomes and costs","width":700,"height":280,"data":{"values":[{"outcome":"Submitted/promoted","chunks":4670,"kind":"work"},{"outcome":"Useful CPU hit","chunks":4550,"kind":"benefit proxy"},{"outcome":"Wasted","chunks":64,"kind":"cost"},{"outcome":"Eviction regret","chunks":2684,"kind":"cost"},{"outcome":"Late","chunks":0,"kind":"failure"},{"outcome":"Post-demand submit","chunks":0,"kind":"failure"}]},"mark":{"type":"bar"},"encoding":{"y":{"field":"outcome","type":"nominal","sort":null,"title":null},"x":{"field":"chunks","type":"quantitative","title":"Chunks"},"color":{"field":"kind","type":"nominal","scale":{"domain":["work","benefit proxy","cost","failure"],"range":["#4575b4","#1a9850","#fdae61","#d73027"]}},"tooltip":[{"field":"outcome","type":"nominal"},{"field":"chunks","type":"quantitative"}]},"config":{"view":{"stroke":null}}}
~~~

This is an important success of the repair: v1's stale FIFO tail is gone. It is also the result that falsifies the expectation that simply making those 64 promotions timely would materially improve AgentX.

## The request-level coverage problem

The block size was 16 tokens. Treatment processed 77,872,458 input tokens, of which 50,823,552 were served through the external KV path.

- Average external prefix per completed request: about 41,153 tokens.
- Average external prefix: about 2,572 chunks.
- Maximum fixed prefetch bundle: 64 chunks = 1,024 tokens.
- A full bundle therefore covers only about 2.49% of the average external prefix.
- Useful proactive chunks covered 72,800 tokens, only 0.0935% of all input tokens and 0.143% of external chunks.
- 4,670 promotions are only about 73 full 64-chunk bundles across 1,235 requests, roughly 5.9% of one bundle per request.

The global tracked limit is also 64, so the mechanism can effectively build only one request's fixed-size bundle at a time.

### Why “useful” does not imply TTFT benefit

The manager classifies a proactive key as useful when a later demand lookup sees that key as a CPU hit. But the connector performs a maximal-prefix lookup over the request's complete external prefix. If the first 64 chunks hit CPU while chunk 65 and thousands after it still require secondary-tier promotion, the request remains deferred.

```text
admission
  └─ proactive: first 64 chunks NVMe → CPU
       └─ first connector lookup scans the full external prefix
            ├─ first 64: CPU HIT → counted “useful”
            └─ remaining ~2,508 chunks: secondary-only
                 └─ ordinary lookup promotes them and request still waits
```

Relevant current code paths:

- `vllm/v1/kv_offload/tiering/manager.py`: proactive demand observation counts a CPU `HIT` as useful.
- `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py::_maximal_prefix_lookup()`: scans the complete external prefix and defers while any required chunk is pending.
- `vllm/v1/kv_offload/tiering/manager.py::lookup()`: already performs exact secondary-tier lookup and initiates promotion of every available missing CPU chunk.

The ordinary lookup is therefore already a scheduler-triggered full-working-set staging operation. The separate admission policy only races it with a very small prefix.

## What “first demand” really means

In v2 the proactive window closes at the manager's first lookup. That lookup is not GPU consumption, but it is when the exact reactive full-prefix promotion begins. Closing the proactive plan there is correct for avoiding duplicate work; continuing afterward was the v1 defect.

The mistaken assumption was treating total vLLM queue time as fully available proactive lead time. The observed queue metric includes both capacity waiting and time deferred for remote KV readiness. Some of that interval occurs **after** first lookup, when the normal exact promotion already owns the transfer. The present telemetry does not directly measure admission-to-first-lookup, so the real proactive horizon remains unknown.

## Bottlenecks and non-bottlenecks

| Signal | Control | Treatment | Interpretation |
|---|---:|---:|---|
| Tiering async lookup mean | 4.448 s | 4.553 s | +104 ms; no hidden-stall reduction |
| Scheduler queue mean | 7.807 s | 7.966 s | +159 ms |
| Waiting depth, native 15 s mean | 5.919 | 6.557 | modestly worse |
| Running depth | 26.695 | 26.701 | unchanged |
| Preemption rate | 0.0537/s | 0.0877/s | +63%; likely secondary pressure signal |
| NVMe reads | 918.3 MB/s | 929.4 MB/s | treatment read more despite less work |
| NVMe busy | 32.1% | 31.9% | device not saturated |
| CPU→GPU transfer | 6.849 GB/s | 6.904 GB/s | essentially unchanged |
| Aggregate CPU usage | 465.7% | 464.9% | planner is not CPU-bound |

There were no model errors, AIPerf errors, cancellations, tracebacks, OOMs, proactive load failures, or storage failure signatures. Both logs contained the same ordinary startup warnings. GPU/DCGM and PCIe artifacts were empty, so device-level utilization and overlap cannot be concluded from this pair.

The scheduler-side policy work itself is too small to explain the regression: intent publication averaged 0.354 ms and scheduler apply averaged 0.048 ms. The likely costs are the speculative I/O and cache perturbation, not Python decision overhead.

## Transfer economics

Using the observed external-token count and CPU→GPU bytes gives an approximate payload density of 135 KB/token, or about 2.17 MB per 16-token chunk.

- Estimated useful prefetched bytes: 9.85 GB.
- Estimated bytes associated with later-regretted victims: 5.81 GB.
- Net useful-minus-regret footprint: about 4.04 GB.
- At the observed 0.929 GB/s NVMe read rate, this is only about 4.35 seconds of aggregate idealized transfer time across the entire profiling phase, or an upper-bound average of roughly 3.5 ms per request.

This is an order-of-magnitude inference, not a direct stall measurement. It nevertheless explains why a timely block-level success ratio can coexist with no measurable request-level benefit.

Eviction regret is substantial: 2,684 of 4,670 proactive allocations evicted an ordinary CPU block that was later demanded, a 57.5% regret rate. This does not mean every regret added a full NVMe miss, but it confirms that the policy is exchanging already useful cache state for a tiny incomplete prefix.

## Root-cause ranking

1. **Primary: insufficient request-level coverage.** Sixty-four chunks cannot make the average 2,572-chunk external working set ready.
2. **Primary: metric/goal mismatch.** A chunk-level CPU hit is labeled useful even when the request remains deferred and sees no critical-path improvement.
3. **Primary: the existing reactive path already has exact knowledge.** At first lookup it stages the whole external working set; admission prefetch has only the earlier arrival-to-first-lookup interval in which to outperform it.
4. **Material cost: eviction regret.** The policy evicted 4,670 ordinary blocks and later regretted 2,684 of those decisions.
5. **Secondary: global width of one 64-chunk bundle.** The cap prevents broad request coverage under concurrency 64.
6. **Not supported: storage saturation, CPU planner overhead, load failure, or stale post-demand work.**

## What not to do next

- Do not interpret 97% useful/promoted as 97% TTFT-effective.
- Do not simply sweep N on the current chunk-level outcome metric.
- Do not restore post-first-lookup submissions; that recreates stale duplicate work.
- Do not add a more elaborate predictor until an oracle proves that full request-level readiness is valuable and physically achievable.
- Do not optimize source-probe or scheduler-apply microseconds; they are not the bottleneck.

## Discriminating next experiment

### Step 1: request-level observability

Add metrics before changing policy:

- admission-to-first-lookup latency;
- proactive chunks / complete external chunks for each request;
- request fully CPU-ready at first lookup;
- lookup deferred despite proactive hits;
- reactive promotions and bytes avoided;
- victim-regret bytes and estimated added stall;
- first-lookup-to-full-CPU-ready latency.

The success metric must become **requests whose external working set is fully ready before exact lookup**, not individual CPU hits.

### Step 2: one-request full-working-set oracle

For the earliest queued FCFS request only, use known keys to stage 25%, 50%, and 100% of its complete external working set, preserving the v2 demand cutoff. This is an oracle feasibility test, not the production policy.

A typical request needs roughly 2,572 chunks, approximately 5.6 GB by the observed payload estimate. At 0.93 GB/s this is about six seconds of NVMe transfer, which is comparable to the observed queue/lookup window. That makes the result genuinely discriminating.

Compare:

1. control;
2. current N=64;
3. full-working-set oracle with normal eviction;
4. full-working-set oracle with controlled reserved CPU capacity, if needed to separate transfer value from eviction damage.

### Go criteria

Continue admission-time CPU staging only if the oracle produces all of:

- at least 50% fewer requests deferred for external KV;
- lower tiering async lookup mean and p95;
- at least 5% lower mean/p95 TTFT or at least 3% higher throughput across replicated same-node pairs;
- positive bytes or stall-time benefit after eviction regret;
- no post-demand submissions or load failures.

If a 100%-coverage oracle cannot meet those gates, stop admission-time NVMe→CPU prefetch for this workload. The stronger next direction is scheduler/data-readiness co-design around the existing exact promotion path, or earlier workflow-triggered staging during real tool/suspension windows. Those signals can provide lead time before the continuation request even reaches vLLM, unlike a predictor starting at HTTP admission.

## Working-set oracle implementation checkpoint — 2026-08-22

The next diagnostic image is now implemented, but not yet built or run, in `/Users/aperdomo/workspace/redhat/vllm-clean-prefetch` on branch `experimental/clean-prefetch-poc`. No commit was created.

The implementation preserves the v2 off-thread secondary probe, batched speculative loads, first-lookup cutoff, request gate, and reactive fallback. It adds:

- `admission_prefetch_mode: working_set`, which targets every complete prompt chunk up to a general safety ceiling rather than a fixed N;
- one non-detached scheduler-ordered owner at a time, using native priority/arrival order;
- `admission_prefetch_max_evictions_per_request`, an explicit bound on ordinary CPU blocks displaced for one request;
- request-level first-lookup outcomes, including target size, confirmed-ready size, bounded coverage, admission-to-first-lookup horizon, actual deferred versus ready result, time from first defer to ready, and eventual external hit chunks;
- a retained `fixed` mode so v2 remains available as a negative baseline;
- container verification and examples for image tag `v0.27.0-clean-prefetch-oracle-v1`.

The initial Nemotron configuration is intentionally model-aware but not hard-coded:

```json
{
  "admission_prefetch_mode": "working_set",
  "admission_prefetch_max_candidate_chunks": 8192,
  "admission_prefetch_apply_chunks_per_step": 256,
  "admission_prefetch_load_batch_chunks": 64,
  "admission_prefetch_max_pending_intents": 64,
  "admission_prefetch_max_tracked_chunks": 8192,
  "admission_prefetch_max_evictions_per_request": 8192,
  "admission_prefetch_max_eviction_history_chunks": 16384,
  "admission_prefetch_tier_idx": 0
}
```

At 16 tokens/chunk, 8,192 chunks cover the configured 131,072-token maximum context. Other models derive the ceiling from their own context and KV chunk size. The request still requires strict Boolean `kv_transfer_params.abc_admission_prefetch=true`.

The eviction control is a bounded budget, **not a carved physical reserve**. A real reserve would require partitioning the CPU allocator and would alter effective control capacity. The first oracle should measure normal production-style displacement with a known maximum; add a symmetric reserved-pool experiment only if full coverage shows benefit but eviction regret masks it.

Verification completed:

- CPU, filesystem, and tiering suites: 121 passed, 7 skipped;
- focused admission and maximal-prefix scheduler suites: 26 passed;
- expanded oracle/configuration/manager set: 47 passed;
- ruff check and format: passed;
- mypy Python 3.12 hook: passed;
- `git diff --check`: passed.

The complete scheduler file could not be validated in the shared local environment. With network disabled, its existing tiny-model fixtures could not resolve Hugging Face metadata. With network enabled, the v0.27.0 worktree still inherited a newer shared virtualenv and failed its baseline `VllmConfig` validation before constructing the connector. These are environment/baseline failures, not failures in the focused oracle tests, but the full file should be rerun inside the v0.27.0 build container or a matching clean virtualenv before treating the image as release-quality.

## Verdict

The clean v2 implementation did what it was designed to do. The current policy objective is the problem: it optimizes timely individual CPU hits, while vLLM's request becomes runnable only when the required external working set is ready. This pair therefore rejects fixed-N=64 admission prefetch as a performance mechanism for realistic AgentX concurrency, while leaving the broader full-working-set or earlier workflow-aware prefetch hypothesis open.