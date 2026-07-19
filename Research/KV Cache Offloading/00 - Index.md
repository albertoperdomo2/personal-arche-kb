# KV Cache Offloading

## Research question

Under which cache sizes, reuse windows, and workload pressures do precise prefix routing with HBM-only, CPU offload, and CPU+NVMe offload show distinct and reproducible performance tiers for agentic workloads?

## Status

**Active.** The experiment is correctly sized at `U=0.64`, CPU32, concurrency 128, and seed 42, but a clean publishable paired triplet is still blocked on component compatibility. The original rerun unknowingly used vLLM 0.23.0 and crashed model workers. The 2026-07-18/19 rerun correctly used vLLM 0.24.0 and eliminated model-worker crashes, but EPP v0.9.0 now CrashLoops on offloading KV events because of a known empty-slice indexing bug.

## Current working conclusion

- Keep `gpu-memory-utilization=0.64`, CPU32, concurrency 128, and seed 42 until reliability is clean. The earlier controlled triplet already produced the intended cache-reuse and performance staircase.
- The requested vLLM 0.24.0 image now deploys correctly: run plan, rendered Deployment, pod manifest, and startup logs all agree.
- vLLM 0.24.0 keeps all offloading model workers alive; the earlier v0.23 EngineCore crash is gone in this batch.
- The new offloading-run failures come from **EPP v0.9.0**, not vLLM workers, workload saturation, CPU memory, or NVMe.
- In both CPU and CPU+NVMe, both EPP replicas restarted 10 times and ended in `CrashLoopBackOff`.
- Exact panic: `runtime error: index out of range [0] with length 0` in `llm-d-kv-cache/pkg/kvcache/kvblock.(*InMemoryIndex).Add`, called from `kvevents.(*Pool).processEventBatch`.
- EPP v0.9.0 receives an offloading event with an empty-but-non-nil engine-key slice and one request key, checks only `engineKeys != nil`, and indexes element zero.
- The missing empty-slice guard was fixed after v0.9.0 in llm-d-kv-cache PR #670 / commit `c4e7265938985c455b177940815a99491f218ff6`, and is already in current llm-d-router source.
- EPP restarts sever active chunked streams (client `TransferEncodingError`) and make new requests return empty HTTP 500s. The wall-clock match between EPP exits and synchronized error bursts is exact.
- CPU had 729 failures (7.85%); CPU+NVMe had 699 (5.93%). Both are rejected for performance comparison.
- Model memory failures and CPU throttling are zero. GPU KV usage is not sustained at 100%. CPU-only reproduces the EPP panic, ruling out NVMe as the cause.
- NVMe was active at approximately 8.17 GB/s aggregate reads and 1.45 GB/s writes; the two node devices averaged about 78% and 81% busy.
- Replace EPP v0.9.0 with an immutable image built after the empty-slice fix, smoke-test CPU and NVMe unchanged, then rerun the paired batch.

## Documents

- [[2026-07-17 - Initial AgentX offloading tier analysis]] — reconstruction of the initial report/MLflow deep dive and proposed clean experiment.
- [[2026-07-18 - U0.64 paired-seed AgentX rerun analysis]] — controlled precise triplet, performance/cache staircase, vLLM 0.23 worker-crash isolation, EPP findings, and the Benchflow image-override bug.
- [[2026-07-19 - vLLM 0.24 offload EPP CrashLoop analysis]] — actual vLLM 0.24 rerun; exact EPP v0.9 empty-engine-key panic, request-error chain, saturation exclusion, and fix/validation plan.

## Source report and artifacts

- Local initial report: `/private/tmp/kv-cache-experiments/llm-d-qwen3.6-35b-a3b-agentx-report.html`
- Initial normalized artifacts: `/private/tmp/kv-cache-experiments/mlflow-report-runs/`
- v0.23 controlled rerun artifacts: `/private/tmp/kv-cache-experiments/rerun-2026-07-18/`
- v0.24 rerun artifacts: `/private/tmp/kv-cache-experiments/v024-rerun-2026-07-19/`
- v0.24 detailed local report: `/private/tmp/kv-cache-experiments/v024-rerun-failure-analysis.md`
- Analysis helpers: `/private/tmp/kv-cache-experiments/analyze_rerun.py` and `analyze_v024_rerun.py`
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

## Immediate next work

1. Override `spec.overrides.images.scheduler` with an EPP image built after commit `c4e7265938985c455b177940815a99491f218ff6`.
2. Use `:main` only for a short validation; pin a verified immutable digest for final experiments.
3. Keep vLLM 0.24.0, U=0.64, CPU32, concurrency 128, seed 42, and topology unchanged.
4. Run a 5–10 minute CPU32 smoke test and require zero EPP/model restart delta and zero request errors.
5. Repeat the smoke test with CPU32+NVMe.
6. Add a Benchflow pre-profile gate for EPP/model readiness and restart deltas; save current and previous logs.
7. Rerun the full four-way batch only after both smoke tests pass.
8. Then run three to five paired seeds and resume performance-gap analysis.
9. Separately fix EPP SSE comment parsing noise, parent-event/index loss, and explicit tier weights.