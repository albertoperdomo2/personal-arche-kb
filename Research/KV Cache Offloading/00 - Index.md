# KV Cache Offloading

## Research question

Under which cache sizes, reuse windows, and workload pressures do precise prefix routing with HBM-only, CPU offload, and CPU+NVMe offload show distinct and reproducible performance tiers for agentic workloads?

## Status

**Active.** The initial AgentX results show a real offloading mechanism and meaningful TTFT/E2E improvement, but the source runs are not yet a clean paired tier comparison.

## Current working conclusion

- CPU offloading improves over HBM-only.
- Adding NVMe further reduces prompt recomputation and improves latency.
- NVMe was active and not saturated; storage performance was not the limiting factor.
- The incremental NVMe gap was constrained because the 32 GiB CPU tier nearly mirrored the TP2 HBM KV capacity and retained almost all reuse except the long tail.
- Precise routing did not fully track filesystem-resident blocks with the tested vLLM 0.23.0 / EPP v0.9.0 combination.
- A controlled experiment should move HBM closer to the working-set boundary while keeping CPU larger than HBM and smaller than the multi-minute reuse tail.

## Documents

- [[2026-07-17 - Initial AgentX offloading tier analysis]] — reconstruction of the initial report/MLflow deep dive and the proposed clean experiment design.

## Source report

- Local report: `/private/tmp/kv-cache-experiments/llm-d-qwen3.6-35b-a3b-agentx-report.html`
- Local normalized artifacts: `/private/tmp/kv-cache-experiments/mlflow-report-runs/`
- Methodology reference: [KV-cache offloading experiments and math](https://www.albertoperdomo.me/posts/kv-cache-offloading-experiments-math)

## Initial MLflow run registry

| Variant | Run ID | Link | Disposition |
|---|---|---|---|
| Optimized baseline | `6ef87d95297842548d7c36eb02f3fcdf` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/6ef87d95297842548d7c36eb02f3fcdf?workspace=benchflow) | Reject: EPP CrashLoop and approximately 90% request errors |
| Precise, no offload | `b3d0fd333acb4b27b5ae2b68124495bd` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/b3d0fd333acb4b27b5ae2b68124495bd?workspace=benchflow) | Directionally usable; topology differs |
| Precise, CPU 32 GiB | `a455cc6580ad401ab37a96bffb6d9150` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/a455cc6580ad401ab37a96bffb6d9150?workspace=benchflow) | Directionally usable; unpaired seed/topology |
| Precise, CPU 32 GiB + NVMe | `e6fd24dc869e4bbbb434af9e653d2fbe` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/e6fd24dc869e4bbbb434af9e653d2fbe?workspace=benchflow) | Mechanism proven; reject as clean comparison because a model pod restarted |

## Next experiment

Run a paired three-seed comparison on identical nodes with:

- eight replicas, TP2;
- concurrency 128 and duration 1,800 seconds;
- `gpu-memory-utilization=0.64`;
- CPU tier 32 GiB for both CPU variants;
- fixed `PYTHONHASHSEED=0`;
- filesystem KV events enabled with compatible vLLM/router builds;
- four tokenizer/render replicas;
- fresh deployments and clean caches for every run.

After the clean comparison, sweep concurrency 128, 160, and 192.