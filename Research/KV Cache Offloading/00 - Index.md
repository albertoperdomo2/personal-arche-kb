# KV Cache Offloading

## Research question

Under which cache sizes, reuse windows, and workload pressures do precise prefix routing with HBM-only, CPU offload, and CPU+NVMe offload show distinct and reproducible performance tiers for agentic workloads?

## Status

**Active.** The U=0.55/C=32 vLLM 0.23 matrix now includes a second CPU64+CephFS observation. NVMe remains directionally promising (+10.5% request throughput over CPU64), while both CephFS runs are rejected healthy-performance cells and accepted failure experiments. The new Ceph run fell to 0.732 requests/s because external reuse collapsed and rare multi-minute stalls throttled AgentX's closed loop; post-run log collection was not causal.

## Current working conclusion

- In the U=0.55/C=32 matrix, CPU64 raised successful throughput from 0.578 to 1.174 requests/s (+103.1%), halved TTFT/E2E latency, and served 77.3% of prompt tokens through external KV transfer versus 0% without offload.
- The corrected CPU64+NVMe run `b06d550d0b314b028a93dd2cd43ecc79` reached 1.298 requests/s (+10.5% over CPU64), reduced mean TTFT 29.8% and mean E2E latency 22.2%, and completed 44 sessions versus 41. It read 459.8 GiB and wrote 381.5 GiB at 13.5% mean NVMe busy; CPU capacity is matched, but `/dev/shm` (200 versus 1 GiB), `cleanup: false`, unset `PYTHONHASHSEED`, and one run per cell keep the effect provisional.
- CPU64+CephFS reached 1.031 requests/s, 12.2% below matched CPU64. A fresh 3 TiB PVC accumulated 192.39 GiB, proving writes, but 102,325 `cannot store blocks` warnings began at 12.31 minutes across 812 request IDs.
- The second CephFS run `1cd063d289f1456da4507382fe284df7` used the same image, flags, CPU tier, shared memory, seed, workload, and a fresh 3 TiB PVC, but moved from node `…-6kl5z` to `…-fx7c8`. It reached only 0.732 requests/s: external prompt sourcing fell to 41.4%, local compute rose to 53.6%, 25 of 55 sessions completed, and the retained 24.87 MB log tail contains at least 153,743 store-refusal warnings.
- This second regression is not a workload-selection artifact: 1,316 of 1,338 successful requests pair to the first Ceph run by AgentX conversation/turn/depth with only a 3-token p95 prompt-length difference. Matched p99 E2E increased from 108.2 to 454.2 seconds and max reached 1,104.3 seconds. The tail stalls, not the better-looking successful-request median, explain the lower offered load.
- Artifact collection started after the benchmark job completed, so log capture did not slow inference. The new model log is tail-truncated; warning onset and totals are lower bounds.
- The vLLM 0.23 source path explains the Ceph failure: async secondary stores retain references on primary CPU blocks; when CephFS cannot drain quickly enough, those blocks are non-evictable, `prepare_store` returns `None`, external prompt sourcing falls 67.7%, and local compute grows 4.0×.
- U=0.55/C=32 is sufficient to expose offload behavior. The original cells kept GPU KV occupancy around 92–94%; the added Ceph run averaged 79.6% only after tail-stalled sessions reduced runnable request supply. Do not increase concurrency or lower U until secondary-tier draining is healthy.
- Repeat the CPU64 versus CPU64+NVMe pair with identical 200 GiB `/dev/shm`, clean run-scoped storage, `PYTHONHASHSEED=0`, identical non-tier flags, direct per-tier hit/byte/latency/queue telemetry, and at least three paired seeds.

- The C=256/U=0.85 CPU64+CephFS run `c3d2102abd41418082df29ae05814c4d` wrote 192.25 GiB to a fresh PVC but still did not expose direct CephFS reads; all Ceph/MDS/OSD and container-FS queries returned zero series.
- It is beyond the workload capacity knee: versus C=128/U=0.90, throughput fell 10.24%, mean TTFT rose 170.07%, waiting requests rose 159.84%, and the cancellation fraction rose from 7.43% to 16.91%.
- The hierarchy became store-heavy: 818.70 GiB GPU→CPU versus 59.80 GiB CPU→GPU (13.69:1), while actual prompt reuse fell from 7.58% to 2.33%. The retained final 86 seconds contain 18,054 `cannot store blocks` warnings.
- The C=256 run again used vLLM 0.23 and its EPP auto-created the approximate producer. Return to U=0.90/C=96–128, pin v0.24+, and use a measured fill/drain/replay test before further pressure sweeps.
- The CPU64 + CephFS mechanism run `d2c57cdc56084c4193d71bfb8e1cfdfb` proves CephFS writes through a 105.84 GiB retained PVC footprint, but direct Ceph/MDS/OSD and container-FS metrics returned no series, so CephFS read hits cannot be claimed.
- That CephFS run accidentally used vLLM 0.23.0. Its retained model log contains at least 107,670 `cannot store blocks` warnings across 398 requests; 103 of 1,386 requests were cancelled at the profile boundary. Treat it as a mechanism/debug run, not a v0.24 performance result.
- The fixed EPP image (digest `sha256:138d54c72f132da48574c6254d7d85d71e7d0186f9fdee6f31a46cc88e319234`, build commit `b41827163b35fa03460f4deecf2cc68bdc60c1a6`) eliminates the v0.9.0 offload-event CrashLoop: every EPP/model pod has zero restarts and no panic.
- The accepted performance staircase is approximate 5.461 req/s, precise no-offload 5.611, CPU32 6.529, and CPU32+NVMe 6.546.
- CPU32 versus precise no-offload improves request throughput 16.36%, mean TTFT 52.47%, and mean E2E latency 32.09%.
- NVMe versus CPU32 improves request throughput only 0.26%, but improves mean TTFT 6.04%, p95 TTFT 24.29%, and mean E2E latency 5.08%.
- Prompt-token reuse is 47.28% approximate, 51.51% precise no-offload, 85.71% CPU32, and 90.65% CPU32+NVMe. CPU32 already captures most useful reuse, leaving diminishing headroom for NVMe.
- NVMe is active, not starved: about 2.05 TB read and 2.04 TB written in 30 minutes at roughly 1.14 GB/s each direction. Device busy averages 24–29% per node and peaks below 35%, arguing against SSD saturation.
- Precise routing pays a large synchronous render/tokenization tax: roughly 139–149 ms p50 EPP dispatch versus 10.6 ms approximate, with 4–5 second p99/max outliers and 21–26 render timeouts.
- The renderer is three `vllm-openai-cpu:v0.23.0` pods requesting one CPU each, while the backend is v0.24.0. Align versions, reserve more CPU/replicas, instrument it, and eliminate timeouts.
- NVMe alone logs 1,116 missing-parent errors across 319 unique engine keys. Router `pkg/kvevents/pool.go` skips those complete `BlockStored` events, making the precise NVMe index incomplete; fix this before final CPU-versus-NVMe claims.
- For a wider tier gap, keep U=0.64 first, reduce CPU capacity equally to 16 GiB/replica in CPU-only and CPU+NVMe, then sweep concurrency 128/192/256. Test lower GPU memory only after that.
- Repeat every selected point at least three times. The current 0.26% NVMe RPS delta is below single-run resolution, while its p95 latency improvement is promising but unreplicated.

## Documents

- [[2026-07-21 - AgentX C32 U0.55 vLLM 0.23 tier matrix/00 - Report|2026-07-21 - AgentX C32 U0.55 vLLM 0.23 tier matrix]] — corrected tier matrix plus the second CephFS observation, paired-request tail analysis, post-run log-capture exclusion, equations, and eleven renderer-validated figures.

- [[2026-07-20 - AgentX C256 U0.85 CPU64 plus CephFS stress analysis]] — native-resolution stress-run audit: write proof, missing read proof, workload saturation, store/load imbalance, approximate-EPP and v0.23 mismatches, nine Vega-Lite figures, and corrected experiment design.
- [[2026-07-20 - AgentX CPU64 plus CephFS single-run analysis]] — CephFS write proof, missing read telemetry, v0.23 version mismatch, CPU-tier pinning/backpressure root cause, nine Vega-Lite figures, and corrected-run gates.
- [[2026-07-20 - AgentX CPU64 plus CephFS pressure plan]] — concurrency-128 capacity window, mandatory deployment corrections, retention-clock caveat, and Ceph readback acceptance gates.
- [[2026-07-17 - Initial AgentX offloading tier analysis]] — reconstruction of the initial report/MLflow deep dive and proposed clean experiment.
- [[2026-07-18 - U0.64 paired-seed AgentX rerun analysis]] — controlled precise triplet, performance/cache staircase, vLLM 0.23 worker-crash isolation, EPP findings, and the Benchflow image-override bug.
- [[2026-07-19 - vLLM 0.24 offload EPP CrashLoop analysis]] — actual vLLM 0.24 rerun; exact EPP v0.9 empty-engine-key panic, request-error chain, saturation exclusion, and fix/validation plan.
- [[2026-07-19 - vLLM 0.24 fixed-EPP clean rerun analysis]] — clean post-fix four-way comparison with Vega-Lite figures, cache-source accounting, NVMe traffic, renderer latency, router-index caveat, and next experiment matrix.

## Source report and artifacts

- Local initial report: `/private/tmp/kv-cache-experiments/llm-d-qwen3.6-35b-a3b-agentx-report.html`
- Initial normalized artifacts: `/private/tmp/kv-cache-experiments/mlflow-report-runs/`
- v0.23 controlled rerun artifacts: `/private/tmp/kv-cache-experiments/rerun-2026-07-18/`
- v0.24 rerun artifacts: `/private/tmp/kv-cache-experiments/v024-rerun-2026-07-19/`
- v0.24 detailed local report: `/private/tmp/kv-cache-experiments/v024-rerun-failure-analysis.md`
- Fixed-EPP rerun artifacts: `/private/tmp/kv-cache-experiments/v024-epp-fix-rerun-2026-07-19/`
- Fixed-EPP Vega-Lite report: `/private/tmp/kv-cache-experiments/v024-epp-fixed-rerun-analysis.md`
- Analysis helpers: `/private/tmp/kv-cache-experiments/analyze_rerun.py`, `analyze_v024_rerun.py`, and `analyze_v024_epp_fix_rerun.py`
- Methodology reference: [KV-cache offloading experiments and math](https://www.albertoperdomo.me/posts/kv-cache-offloading-experiments-math)

## Initial MLflow run registry

| Variant | Run ID | Link | Disposition |
|---|---|---|---|
| Optimized baseline | `6ef87d95297842548d7c36eb02f3fcdf` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/6ef87d95297842548d7c36eb02f3fcdf?workspace=benchflow) | Reject: EPP CrashLoop and approximately 90% request errors |
| Precise, no offload | `b3d0fd333acb4b27b5ae2b68124495bd` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/b3d0fd333acb4b27b5ae2b68124495bd?workspace=benchflow) | Directionally usable; topology differs |
| Precise, CPU 32 GiB | `a455cc6580ad401ab37a96bffb6d9150` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/a455cc6580ad401ab37a96bffb6d9150?workspace=benchflow) | Directionally usable; unpaired seed/topology |
| Precise, CPU 32 GiB + NVMe | `e6fd24dc869e4bbbb434af9e653d2fbe` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/e6fd24dc869e4bbbb434af9e653d2fbe?workspace=benchflow) | Mechanism proven; reject because a model pod restarted |

## vLLM 0.23 controlled rerun registry

| Variant | Run ID | Link | Disposition |
|---|---|---|---|
| Optimized approximate baseline | `d007e8f640354c509700f651bf6d2045` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/d007e8f640354c509700f651bf6d2045?workspace=benchflow) | Clean standalone approximate control |
| Precise, no offload | `20ce8f347172412a98c22fc6b1a3b30b` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/20ce8f347172412a98c22fc6b1a3b30b?workspace=benchflow) | Clean: zero errors, restarts, and preemptions |
| Precise, CPU 32 GiB | `c9d75053b9564393bfdf902331511588` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/c9d75053b9564393bfdf902331511588?workspace=benchflow) | Reject: four model workers restarted; 48 errors |
| Precise, CPU 32 GiB + NVMe | `29a50025c7bc421b802023cd31e22ff9` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/29a50025c7bc421b802023cd31e22ff9?workspace=benchflow) | Reject: one model worker restarted twice; 15 errors |

## vLLM 0.24 + EPP v0.9 rerun registry

| Variant | Run ID | Link | Disposition |
|---|---|---|---|
| Optimized approximate baseline | `c81791112ad94c6f9fb9ad53ef5db2c9` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/c81791112ad94c6f9fb9ad53ef5db2c9?workspace=benchflow) | Clean: zero errors |
| Precise, no offload | `ce34fc90549544519366152afc5768dd` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/ce34fc90549544519366152afc5768dd?workspace=benchflow) | Clean: zero errors and zero EPP/model restarts |
| Precise, CPU 32 GiB | `d55f0a79f89d493fbf2652cefedab382` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/d55f0a79f89d493fbf2652cefedab382?workspace=benchflow) | Reject: 729 errors; each EPP restarted 10 times |
| Precise, CPU 32 GiB + NVMe | `2857520dd3dd4762b0b241a4dfdbaafd` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/2857520dd3dd4762b0b241a4dfdbaafd?workspace=benchflow) | Reject: 699 errors; each EPP restarted 10 times |

## vLLM 0.24 + fixed-EPP clean rerun registry

| Variant | Run ID | Link | Disposition |
|---|---|---|---|
| Optimized approximate baseline | `e6f54f37a63549cbb4feaa1b4dd78c00` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/e6f54f37a63549cbb4feaa1b4dd78c00?workspace=benchflow) | Accept; zero restarts, one isolated HTTP 503 |
| Precise, no offload | `186e43cb9b4a4a7e82c3bca930a5cb35` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/186e43cb9b4a4a7e82c3bca930a5cb35?workspace=benchflow) | Accept; zero errors and zero restarts |
| Precise, CPU 32 GiB | `49e9185c2a5f4cc4951dfec020d67bd9` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/49e9185c2a5f4cc4951dfec020d67bd9?workspace=benchflow) | Accept; 16.36% RPS over no-offload, one isolated HTTP 503 |
| Precise, CPU 32 GiB + NVMe | `d37d113709e5426f92a3b1ac271c9c92` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/d37d113709e5426f92a3b1ac271c9c92?workspace=benchflow) | Accept with router-index caveat; active NVMe, one isolated HTTP 503 |

## CephFS mechanism audit registry

| Variant | Run ID | Link | Disposition |
|---|---|---|---|
| AgentX, CPU 64 GiB + CephFS, U=0.9, C=128 | `d2c57cdc56084c4193d71bfb8e1cfdfb` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/d2c57cdc56084c4193d71bfb8e1cfdfb?workspace=benchflow) | Reject for performance: actual vLLM 0.23.0; writes proven, read hits unproven; store-admission warning flood and 103 boundary cancellations |
| AgentX, CPU 64 GiB + CephFS, U=0.85, C=256 | `c3d2102abd41418082df29ae05814c4d` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/c3d2102abd41418082df29ae05814c4d?workspace=benchflow) | Reject: actual vLLM 0.23.0 and approximate EPP; writes proven (192.25 GiB), reads unproven; 16.91% boundary cancellations and store-heavy cache thrash |

## vLLM 0.23 U0.55/C32 tier-matrix registry

| Variant | Run ID | Link | Disposition |
|---|---|---|---|
| No offload | `6ded92329a4844c5a4c6f11f5cab764c` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/6ded92329a4844c5a4c6f11f5cab764c?workspace=benchflow) | Directionally accept; 19 boundary cancellations censor the latency tail |
| CPU 64 GiB | `aad824ce1e8b47699e869fd9fcf86625` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/aad824ce1e8b47699e869fd9fcf86625?workspace=benchflow) | Accept as the matched CPU-offload comparison |
| CPU 64 GiB + NVMe | `b06d550d0b314b028a93dd2cd43ecc79` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/b06d550d0b314b028a93dd2cd43ecc79?workspace=benchflow) | Directionally accept: capacity matched, active readback, no errors/warnings; repeat with shared-memory, clean-state, hash-seed, and multi-seed controls |
| CPU 64 GiB + CephFS | `4adc495ff3a841c580d2cbbe5d8a01eb` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/4adc495ff3a841c580d2cbbe5d8a01eb?workspace=benchflow) | Reject as healthy performance; accept as a filesystem-drain failure experiment |
| CPU 64 GiB + CephFS (log capture) | `1cd063d289f1456da4507382fe284df7` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/1cd063d289f1456da4507382fe284df7?workspace=benchflow) | Reject: repeated drain failure, severe p99 tail; log collection was post-run and the retained model log is incomplete |

## Immediate next work

1. Repeat CPU64 and CPU64+NVMe with identical U=0.55/C=32/TP=2, 200 GiB shared memory, image digest, release topology, and all non-tier flags.
2. Repeat CPU64+CephFS on `…-6kl5z` and `…-fx7c8` with a unique fresh PVC per run; require zero sustained `cannot store blocks` and no p99 tail explosion.
3. Start every filesystem tier empty, set `PYTHONHASHSEED=0`, retain the complete model log, and run at least three paired repetitions per node.
4. Instrument secondary-tier hits/misses, submitted/completed bytes, latency, queue depth, in-flight blocks/jobs, and failures; fix Ceph pool/MDS/OSD metric collection.
5. Sweep CephFS write threads only after queue telemetry exists; accept a setting only if completion rate remains at or above submission rate.
6. Repeat every selected point with at least three seeds and add a fixed-request/reuse replay beside the realistic AgentX run.
7. Keep the verified fixed EPP digest for v0.24 routing work; separately finish renderer capacity/version alignment and NVMe missing-parent event-ordering fixes.