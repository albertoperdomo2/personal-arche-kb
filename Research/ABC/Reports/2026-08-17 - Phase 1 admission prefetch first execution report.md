---
title: Phase 1 admission prefetch first execution report
date: 2026-08-17
last_updated: 2026-08-18
type: research_report
experiment: ABC
status: invalid_inconclusive
model: nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
vllm_version: 0.27.0
images:
  official: docker.io/vllm/vllm-openai@sha256:07ea4e292adf3a26b05ac97114b28849cf4551a26beb1fbe7decd3842d752ed7
  custom: quay.io/rh-ee-aperdomo/vllm@sha256:32a580fe2005571b7800c8862a0c484ac3626dfb3f4d6da0c0cdf4de1c8ae054
tensor_parallelism: 8
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 8192
max_num_seqs: 1
concurrency: 8
cpu_bytes: 274877906944
offload_spec: TieringOffloadingSpec
secondary_tier: filesystem on local NVMe
secondary_tier_threads: 64 read / 64 write
shared_memory: 300Gi
workload: 4096 synthetic prompt tokens, 64 output tokens, 256 measured requests
warmup: 1024 requests with abc_admission_prefetch=false
random_seed: 20260814
cache_cleaning: hostPath cleanup enabled between deployments; no reset during measured phase
prometheus_source_cadence: 15 seconds
---

# Phase 1 admission prefetch first execution report

## Executive summary

This batch compared three Nemotron FP8 deployments under the same GuideLLM workload: official vLLM without the experimental code, the custom image with `admission_prefetch_chunks=0`, and the same custom image with `admission_prefetch_chunks=100`. Every measured request carried `kv_transfer_params.abc_admission_prefetch=true`; all warm-up requests explicitly carried `false`.

The batch does **not** test proactive prefetch performance. The configured treatment emitted no `prefetch_attempted`, `promoted`, `redundant`, `skipped`, `useful`, `wasted`, `untracked`, `load_failed`, or `late` series. Code inspection explains the absence: the manager stores the configured limit in `_admission_prefetch_chunks`, while the scheduler reads `manager.admission_prefetch_chunks` with a zero fallback. The treatment therefore executed as an ordinary reactive-offload run.

## Validity verdict — Invalid / inconclusive for admission prefetch

The mechanism gate failed before any performance comparison can be accepted. `N=100` was parsed correctly and appears in the model log, but the admission hook observed zero because the manager exposes no matching public attribute. The apparent treatment-versus-control differences are run variability between two effectively non-prefetching deployments.

The runs remain useful as a deployment and observability smoke test: all 768 measured requests succeeded, the custom image started cleanly at the same immutable digest in both custom cells, warm-up/measurement request gating was correct, NVMe-backed reactive offload was active, and no allocation failures or lookup-overflow series appeared.

## Main takeaways

- **Good — workload execution was clean:** each cell completed 256/256 measured requests with zero errors or cancellations at mean client concurrency 7.86–7.89.
- **Good — control/treatment image parity was correct:** both custom cells used `sha256:32a580...`, and they differed in the intended server knob (`0` versus `100`).
- **Good — the storage/replay workload exercised reactive offload:** external KV supplied approximately 44–48% of prompt-token rate over the measurement window, CPU→GPU KV traffic averaged about 484–502 MiB/s when present, and runtime-node NVMe reads averaged about 223–236 MiB/s.
- **Bad — proactive prefetch never started:** all nine prefetch metric queries returned zero series in the `N=100` cell. With 256 marked requests and at least 100 complete prompt chunks each, a working hook should have attempted 25,600 chunks.
- **Bad — the custom control was an outlier:** it had only 3 requests above 8 seconds TTFT, versus 33 in both the official control and configured treatment. Because neither custom cell prefetched, this tail difference cannot be credited to the feature.
- **Bad — hardware placement was not paired:** the three model pods landed on three different nodes and two NVMe device names. The MLflow tag says `H200`, while all selected node names contain `gpu-h100`; accelerator metadata must be corrected or verified.
- **Bad — all model logs report `_swap_blocks_kernel` JIT compilation during inference.** That adds a known latency disturbance and should be eliminated before the performance trial.

## Headline metrics

The official image is listed first as the broad baseline. The custom-image `N=0` cell is the correct direct mechanism control for the configured `N=100` cell.

| Configuration | Image / N | Requests | Errors | Request throughput (req/s) | Δ vs official | Mean TTFT (ms) | Δ vs official | p95 TTFT (ms) | Mean E2E (s) | Output rate (tok/s) |
|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| Official control | official / absent | 256 | 0 | 1.0259 | baseline | 6756.6 | baseline | 8158.2 | 7.6582 | 65.87 |
| Custom-image control | custom / 0 | 256 | 0 | 1.0603 | +3.35% | 6543.7 | -3.15% | 7189.2 | 7.4426 | 68.09 |
| Configured treatment; prefetch inactive | custom / 100 | 256 | 0 | 1.0205 | -0.53% | 6832.8 | +1.13% | 8152.8 | 7.7346 | 65.53 |

Directly against the custom-image control, the configured treatment had 3.75% lower request throughput, 4.42% higher mean TTFT, 0.72% higher median TTFT, 13.40% higher p95 TTFT, and 3.92% higher mean E2E latency. These are **observations between two non-prefetching runs**, not a feature result.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 \u2014 Configured N=100 result relative to custom-image N=0 control","width":740,"height":280,"data":{"values":[{"metric":"Request throughput","delta_pct":-3.75,"direction":"Rate"},{"metric":"Mean TTFT","delta_pct":4.42,"direction":"Latency"},{"metric":"Median TTFT","delta_pct":0.72,"direction":"Latency"},{"metric":"p95 TTFT","delta_pct":13.4,"direction":"Latency"},{"metric":"Mean E2E","delta_pct":3.92,"direction":"Latency"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"metric","type":"nominal","title":null,"sort":null,"axis":{"labelAngle":-20}},"y":{"field":"delta_pct","type":"quantitative","title":"Relative difference (%)"},"color":{"field":"direction","type":"nominal","title":"Metric type","scale":{"scheme":"category10"}},"tooltip":[{"field":"metric","type":"nominal"},{"field":"delta_pct","type":"quantitative","title":"Difference (%)"}]},"config":{"view":{"stroke":null}}}
```

Figure 1 uses MLflow GuideLLM aggregate metrics. Positive latency deltas are worse; positive rate deltas are better. It visualizes the observed run difference only—the failed mechanism gate forbids causal attribution.

## Run registry and configuration evidence

| Role | Run | Revision tag | Runtime image | `admission_prefetch_chunks` | Model node | Duration |
|---|---|---|---|---:|---|---:|
| Official control | [23b7f315…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/23b7f315a6a54c08b484b113037abccc?workspace=benchflow) | `official-no-prefetch` | official digest `07ea4e...` | absent | `...-gjfjh` | 249.5 s |
| Custom control | [3ee22e3a…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/3ee22e3ae07144039b83d9e6b8dfcbf0?workspace=benchflow) | `custom-no-prefetch` | custom digest `32a580...` | 0 | `...-mt46x` | 241.4 s |
| Configured treatment | [b6bce021…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/b6bce02143a0431baa9935731cbe8b23?workspace=benchflow) | `custom-prefetch` | custom digest `32a580...` | 100 | `...-6kl5z` | 250.9 s |

Controlled dimensions were model, TP=8, one replica, `gpu_memory_utilization=0.8`, `max_num_seqs=1`, 256 GiB CPU tier, 64 filesystem read and write threads, 300 GiB `/dev/shm`, 4,096/64 token workload, eight client streams, seed 20260814, 1,024-request pre-warm, and 256 measured requests. The exact 256 measured request bodies were the same set in all cells, although their admission order differed under concurrency.

## Prefetch mechanism evidence

The treatment model log confirms `admission_prefetch_chunks: 100`; the actual measured HTTP bodies confirm `abc_admission_prefetch: true`; pre-warm bodies confirm `false`. Despite that, every prefetch query returned an empty Prometheus matrix:

| Metric | Treatment series | Interpretation |
|---|---:|---|
| `prefetch_attempted` | 0 | Admission prefetch entry point did not attempt candidates |
| `prefetch_promoted` | 0 | No proactive NVMe→CPU promotion |
| `prefetch_redundant` | 0 | No candidates classified as already resident/pending |
| `prefetch_skipped` | 0 | No capacity/filter skips recorded |
| `prefetch_useful` | 0 | No proactive copy later consumed |
| `prefetch_wasted` | 0 | No proactive copy expired unused |
| `prefetch_untracked` | 0 | No tracking overflow activity |
| `prefetch_load_failed` | 0 | No proactive job existed to fail |
| `prefetch_late` | 0 | No proactive job was observed late |

These are **missing series, not measured zero-valued counters**. The distinction matters: the metric children are created only when increments occur. A working N=100 treatment should create at least `attempted` and one or more partition series with `tier="1:fs"`.

### Root cause

Current custom-image code has this mismatch:

```text
TieringOffloadingManager.__init__:
    self._admission_prefetch_chunks = admission_prefetch_chunks

OffloadingConnectorScheduler._maybe_prefetch_on_admission:
    n = getattr(self.manager, "admission_prefetch_chunks", 0)
```

Because no `admission_prefetch_chunks` property exists, `getattr(..., 0)` returns zero and the scheduler exits before key generation or manager prefetch. The focused scheduler unit tests used a mock manager with a public `admission_prefetch_chunks` attribute, so they did not exercise real spec→manager wiring and missed the defect.

## Reactive offload and resource evidence

Reactive correctness remained intact. Over native 15-second samples restricted to each measured phase:

| Configuration | External prompt-token share mean | Reactive async lookup mean | Waiting requests mean | NVMe read mean | CPU→GPU KV rate mean* |
|---|---:|---:|---:|---:|---:|
| Official N=0 | 46.6% | 4.657 s | 6.88 | 226 MiB/s | 484 MiB/s |
| Custom N=0 | 43.6% | 4.436 s | 6.56 | 223 MiB/s | 500 MiB/s |
| Custom N=100, inactive | 47.9% | 4.429 s | 6.56 | 236 MiB/s | 502 MiB/s |

\*CPU→GPU means use the available rate samples; the first measurement samples can be absent before the rate series becomes defined.

This similarity is consistent with all three cells running the normal reactive path. It also shows that the queue-cover prerequisite existed: one request ran while approximately 6–7 waited. The missing ingredient was not overlap opportunity but admission-hook activation.

Native evidence is preserved in:

- [[Reports/2026-08-17 - Phase 1 admission prefetch first execution report/Request TTFT comparison - requests 1-128]]
- [[Reports/2026-08-17 - Phase 1 admission prefetch first execution report/Request TTFT comparison - requests 129-256]]
- [[Reports/2026-08-17 - Phase 1 admission prefetch first execution report/Request E2E latency comparison]]
- [[Reports/2026-08-17 - Phase 1 admission prefetch first execution report/Offload and storage telemetry comparison]]

## What went well

1. The multi-arch custom image deployed and served Nemotron FP8 without functional failures.
2. Both custom cells resolved to the same immutable digest, eliminating accidental tag drift within the direct comparison.
3. The HTTP gate was correctly separated by phase: warm-up false, measurement true.
4. The server-side N knob was rendered as intended in both custom cells and visible in startup logs.
5. Queue pressure was sufficient for the intended overlap experiment.
6. Reactive NVMe offload remained the correctness fallback; no request failed when proactive work was absent.
7. BenchFlow captured the new prefetch queries even though the implementation never instantiated their series, making the failure diagnosable.

## What went poorly

1. Configuration plumbing stopped at a private manager field, so the central mechanism never ran.
2. The unit-test boundary was too mocked and reproduced the scheduler’s expectation instead of the real manager interface.
3. A full 1,024+256 request run was launched before a small live mechanism smoke test proved `prefetch_attempted > 0`.
4. One repetition per cell is insufficient, especially with a large 3-versus-33 difference in TTFT tail counts between two no-prefetch runs.
5. Each cell landed on a different model node, exposing the comparison to GPU, CPU, and local-NVMe node variance.
6. Accelerator metadata is internally inconsistent (`H200` tag versus `gpu-h100` node names).
7. Triton compiled `_swap_blocks_kernel` during the measured phase in every cell, creating avoidable latency spikes.

## Conclusions

### Measured observation

The configured treatment was slower than the custom control in this single execution, but emitted no proactive-prefetch telemetry. The official control and configured treatment were nearly identical on median and p95 TTFT, while the custom control had an unusually small slow tail.

### Inference

The treatment behaved as `N=0`. The performance spread is ordinary execution, admission-order, node, storage, and JIT variability. It neither supports nor refutes the proactive-prefetch hypothesis.

### Working conclusion

Phase 1 remains open. The benchmark and observability design are largely usable, but the code must expose the configured admission limit through the interface the scheduler reads, and a live mechanism smoke test must pass before repeating the performance matrix.

## Next steps

1. Add a read-only `TieringOffloadingManager.admission_prefetch_chunks` property returning `_admission_prefetch_chunks` (or establish one consistently named public field). Do not make the scheduler read a private attribute.
2. Add a real-wiring regression test that constructs the manager through `TieringOffloadingSpec`, passes it to the scheduler, sets `N=100`, and verifies the public manager method receives 100 ordered keys. Retain the strict request gate tests.
3. Rebuild under a new immutable tag/digest. Record the digest in the deployment profile or MLflow tags.
4. Run a tiny live smoke test before the full benchmark: warm storage, issue 8–16 marked measured requests, and require nonzero `attempted{tier="1:fs"}` plus the accounting identity `attempted = promoted + redundant + skipped`.
5. Require zero `load_failed`, no old `tier="prefetch"` label, useful/effective-promoted ≥0.9, and a low late rate before examining TTFT.
6. Eliminate measured-phase JIT by extending or shaping warm-up until `_swap_blocks_kernel` compiles before measurement.
7. Rerun the direct matrix with the same custom digest: request flag true in both measured cells, `N=0` versus `N=100`, warm-up flag false in both. Use at least five paired repetitions.
8. Pin both cells to the same model node when possible; otherwise randomize/interleave cell order and balance repetitions by node. Record actual GPU SKU and NVMe device identity rather than relying on the current tag.
9. Do not tune N yet. Once the N=100 mechanism and performance gates pass, sweep `N ∈ {25, 50, 100, 200}`.

## Post-analysis repair checkpoint — 2026-08-17

The configuration-wiring defect is fixed in the local vLLM working tree, without a commit:

- `TieringOffloadingManager` now exposes a read-only `admission_prefetch_chunks` property backed by `_admission_prefetch_chunks`;
- the existing manager configuration test now asserts the public interface;
- a scheduler regression test uses a real `TieringOffloadingManager` and proves that configured `N=100` passes the first 100 ordered request keys to `prefetch_assume_resident()`;
- `Containerfile.vllm-prefetch` now fails its overlay verification if the copied manager lacks the public property.

Validation completed:

```text
69 passed — tests/v1/kv_offload/tiering/test_tiering_offloading.py
41 passed — TestAdmissionPrefetch + TestMaximalPrefixLookup + TestSlidingWindowLookup
ruff format --check: 3 files already formatted
ruff check: all checks passed
git diff --check: passed
```

The broader scheduler file's network-sensitive fixtures exceeded the available execution window after requiring Hugging Face metadata; the focused scheduler sections covering this change completed successfully. No image has yet been rebuilt from the repaired working tree. The next checkpoint is a new immutable image digest followed by the 8–16 request live mechanism smoke test.

## Repeat execution checkpoint — 2026-08-18

Two nominal custom-image cells were executed after the local repair checkpoint:

| Role | Run | Configured N | Completed requests | Request throughput | Mean TTFT | p95 TTFT | Resolved image |
|---|---|---:|---:|---:|---:|---:|---|
| Nominal control | [048fa430…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/048fa4300c4c4b878941f72395c1258e?workspace=benchflow) | 0 | 255/256 | 1.0403 req/s | 6688.7 ms | 8123.3 ms | old custom digest `32a580...` |
| Nominal treatment | [eddf9874…](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/358/runs/eddf9874c8304cf79fe3231b722be21c?workspace=benchflow) | 100 | 256/256 | 1.0449 req/s | 6652.0 ms | 7241.1 ms | old custom digest `32a580...` |

### Verdict

This repeat is also **invalid/inconclusive** for proactive prefetch. Both pods pulled `quay.io/rh-ee-aperdomo/vllm:v0.27.0-prefetch-p1` and resolved to `sha256:32a580fe2005571b7800c8862a0c484ac3626dfb3f4d6da0c0cdf4de1c8ae054`—the same pre-repair image used by the first execution. The local manager-property fix was therefore absent.

The N=100 model log parsed `admission_prefetch_chunks: 100`, but all nine archived prefetch queries again returned empty result matrices: attempted, promoted, redundant, skipped, useful, wasted, untracked, load_failed, and late. These are missing series, not zero-valued activity. The mechanism gate failed exactly as in the first execution.

The nominal control has an additional validity failure. GuideLLM created 256 requests but processed and marked successful only 255; one remained in `processing_requests`, `end_processing_time` was null, and the max-request constraint still reported one remaining request even though the client emitted “Benchmarking complete” and BenchFlow marked the run FINISHED.

### Observed aggregate differences, without causal attribution

Relative to the incomplete N=0 cell, the nominal N=100 cell showed:

- request throughput: +0.44%;
- mean TTFT: −0.55%;
- median TTFT: +0.37%;
- p95 TTFT: −10.86%;
- p99 TTFT: −9.89%;
- mean E2E latency: −0.44%;
- p95 E2E latency: −9.73%.

These numbers are not evidence that admission prefetch helped: proactive work did not run, one control request did not complete, the model pods used different nodes, and there is only one repetition. The tail improvement is retained only as an observed no-prefetch run-to-run difference.

Both logs again show `_swap_blocks_kernel` JIT compilation during the measured phase. The deployment tag still says H200 while the selected node names contain `gpu-h100`, so the accelerator metadata inconsistency also remains.

### Required correction before another run

Do not launch another full matrix from the mutable `v0.27.0-prefetch-p1` tag. Build and publish the repaired tree under a new tag, resolve and record its immutable digest, update the BenchFlow experiment to that tag or digest, and verify the live pod `imageID` differs from `32a580...` before sending benchmark traffic. Then run the small mechanism smoke test and require nonzero `attempted{tier="1:fs"}` before proceeding to a 256-request comparison.

## Data provenance and limitations

Client aggregates and per-request values come from each run’s `results/benchmark_output.json`. Mechanism, queue, transfer, and storage series come from the archived Prometheus artifacts at their native 15-second cadence and are restricted—not resampled—to each measured-phase interval. Model logs, rendered run plans, pod manifests, and pod descriptions establish arguments, image digests, placement, and JIT warnings.

No GPU utilization series was available in these artifacts, so this report does not substitute GPU KV-cache occupancy or GPU memory values for GPU core utilization. Device-level NVMe telemetry is node-wide and may include activity outside the model process; temporal alignment supports diagnosis but not exclusive attribution.