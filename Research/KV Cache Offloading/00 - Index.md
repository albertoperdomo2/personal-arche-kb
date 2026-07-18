# KV Cache Offloading

## Research question

Under which cache sizes, reuse windows, and workload pressures do precise prefix routing with HBM-only, CPU offload, and CPU+NVMe offload show distinct and reproducible performance tiers for agentic workloads?

## Status

**Active.** The controlled `U=0.64`, CPU32, concurrency-128, seed-42 triplet is complete. It produces the desired performance and cache-reuse staircase, but the CPU and CPU+NVMe runs are rejected because vLLM 0.23.0 EngineCore workers restarted.

## Current working conclusion

- The experiment is correctly sized. Keep `gpu-memory-utilization=0.64`, CPU32, concurrency 128, and seed 42 for the reliability rerun.
- Precise no offload → CPU32 → CPU32+NVMe produced 5.417 → 5.928 → 6.499 RPS.
- CPU32 improved RPS 9.4%, average TTFT 29.5%, and average E2E 22.0% versus precise no offload.
- NVMe added another 9.6% RPS, 34.3% average TTFT, and 22.7% average E2E improvement over CPU.
- Prompt recomputation formed the desired 57.9% → 33.4% → 11.1% staircase.
- The clean precise no-offload control had zero errors, restarts, and preemptions. CPU-only differs only by `OffloadingConnector`, isolating worker deaths to vLLM's offload path rather than workload, precise EPP, saturation, or NVMe.
- The requested vLLM 0.24.0 image was silently ignored by Benchflow. Every run actually deployed vLLM 0.23.0 because the llm-d guide owns its image transformer in a Kustomize component and Benchflow only updates a nonempty top-level `images` list.
- NVMe was active at about 2.57 GB/s reads and 1.31 GB/s writes, below 47% device busy. Storage was not the bottleneck.
- EPP v0.9.0 dropped 3,814 parent-dependent block events in the NVMe run and had missing precise-prefix data on about 2.8% of requests.
- Fix the image renderer, assert the final image, actually test a newer vLLM, and collect `kubectl logs --previous` before changing workload sizing.

## Documents

- [[2026-07-17 - Initial AgentX offloading tier analysis]] — reconstruction of the initial report/MLflow deep dive and proposed clean experiment.
- [[2026-07-18 - U0.64 paired-seed AgentX rerun analysis]] — complete precise triplet, performance/cache staircase, offload crash isolation, EPP findings, and the Benchflow image-override bug.

## Source report and artifacts

- Local initial report: `/private/tmp/kv-cache-experiments/llm-d-qwen3.6-35b-a3b-agentx-report.html`
- Initial normalized artifacts: `/private/tmp/kv-cache-experiments/mlflow-report-runs/`
- Latest rerun artifacts: `/private/tmp/kv-cache-experiments/rerun-2026-07-18/`
- Latest analysis: `/private/tmp/kv-cache-experiments/rerun-2026-07-18/analysis.json`
- Analysis helper: `/private/tmp/kv-cache-experiments/analyze_rerun.py`
- Methodology reference: [KV-cache offloading experiments and math](https://www.albertoperdomo.me/posts/kv-cache-offloading-experiments-math)

## Initial MLflow run registry

| Variant | Run ID | Link | Disposition |
|---|---|---|---|
| Optimized baseline | `6ef87d95297842548d7c36eb02f3fcdf` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/6ef87d95297842548d7c36eb02f3fcdf?workspace=benchflow) | Reject: EPP CrashLoop and approximately 90% request errors |
| Precise, no offload | `b3d0fd333acb4b27b5ae2b68124495bd` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/b3d0fd333acb4b27b5ae2b68124495bd?workspace=benchflow) | Directionally usable; topology differs |
| Precise, CPU 32 GiB | `a455cc6580ad401ab37a96bffb6d9150` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/a455cc6580ad401ab37a96bffb6d9150?workspace=benchflow) | Directionally usable; unpaired seed/topology |
| Precise, CPU 32 GiB + NVMe | `e6fd24dc869e4bbbb434af9e653d2fbe` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/e6fd24dc869e4bbbb434af9e653d2fbe?workspace=benchflow) | Mechanism proven; reject because a model pod restarted |

## 2026-07-18 controlled rerun registry

| Variant | Run ID | Link | Disposition |
|---|---|---|---|
| Optimized approximate baseline | `d007e8f640354c509700f651bf6d2045` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/d007e8f640354c509700f651bf6d2045?workspace=benchflow) | Clean standalone approximate control |
| Precise, no offload | `20ce8f347172412a98c22fc6b1a3b30b` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/20ce8f347172412a98c22fc6b1a3b30b?workspace=benchflow) | Clean: zero errors, restarts, and preemptions |
| Precise, CPU 32 GiB | `c9d75053b9564393bfdf902331511588` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/c9d75053b9564393bfdf902331511588?workspace=benchflow) | Reject: four model workers restarted; 48 errors |
| Precise, CPU 32 GiB + NVMe | `29a50025c7bc421b802023cd31e22ff9` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/29a50025c7bc421b802023cd31e22ff9?workspace=benchflow) | Reject: one model worker restarted twice; 15 errors |

## Immediate next work

1. Fix Benchflow's llm-d recipe image override so component-owned image transformers can be overridden.
2. Add a pre-deployment assertion that run-plan, rendered Deployment, and startup-log vLLM versions agree.
3. Collect current and previous container logs for restarted pods.
4. Run CPU32 for 10 minutes using the intended vLLM 0.24.0 or selected newer build.
5. If clean, repeat the full precise triplet with the same U, CPU size, concurrency, topology, and seed.
6. Repair EPP parent-event indexing and configure verified per-tier weights.
7. Warm or scale renderers to eliminate measured-window tokenization deadlines.
8. Restore DCGM GPU telemetry.
9. After one clean triplet, repeat across three to five paired seeds. Do not sweep U or concurrency before reliability is solved.