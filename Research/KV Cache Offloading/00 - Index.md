# KV Cache Offloading

## Research question

Under which cache sizes, reuse windows, and workload pressures do precise prefix routing with HBM-only, CPU offload, and CPU+NVMe offload show distinct and reproducible performance tiers for agentic workloads?

## Status

**Active.** The first post-fix vLLM 0.24 four-way batch is clean enough for performance analysis: all EPP/model pods stayed Ready with zero restarts. In this accepted batch, CPU32 creates a large gain over precise no-offload; CPU32+NVMe is active and improves latency tails but adds almost no request throughput beyond CPU32. Remaining publication blockers are precise-renderer latency/timeouts, NVMe-only missing-parent index events, and lack of repeated runs.

## Current working conclusion

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

## Immediate next work

1. Pin the verified fixed EPP digest and keep vLLM 0.24.0.
2. Align render pods to vLLM 0.24.0, reserve more CPU, scale/balance them, and capture render CPU/request-latency telemetry.
3. Require zero precise tokenization timeouts.
4. Diagnose/fix NVMe missing-parent event ordering or make the index recover instead of dropping the complete `BlockStored` event.
5. Repeat the current U0.64/CPU32/concurrency-128 point at least three times to estimate variance.
6. Run paired CPU16 and CPU16+NVMe at concurrency 128, then 192 and 256.
7. Select the point where CPU-only recomputation is 20–35% and NVMe recomputation is below 10–15%, without sustained queue/storage saturation.
8. Only then add a U0.56 sensitivity point.
9. Keep the realistic AgentX trace as primary; optionally add a shorter-output cache-sensitive agentic profile to expose throughput rather than decode dominance.