# KV Cache Offloading

## Research question

Under which cache sizes, reuse windows, and workload pressures do precise prefix routing with HBM-only, CPU offload, and CPU+NVMe offload show distinct and reproducible performance tiers for agentic workloads?

## Status

**Active.** The controlled `U=0.64`, seed-42 rerun proves strong CPU and NVMe reuse, but the two offload runs are rejected because vLLM EngineCore workers restarted. The precise no-offload member of the triplet is still pending/rerunning.

## Current working conclusion

- CPU offloading improves prompt reuse; adding NVMe reduces recomputation substantially again.
- In the latest rerun, CPU+NVMe versus CPU produced +9.6% RPS, -34.3% average TTFT, -45.4% p95 TTFT, and -22.7% average E2E, but the effect size is contaminated by worker restarts.
- Prompt-token accounting is the strongest mechanism evidence: recomputation fell from 55.7% in the optimized baseline to 33.4% with CPU and 11.1% with CPU+NVMe.
- NVMe was active at about 2.57 GB/s reads and 1.31 GB/s writes, below 47% device busy. Storage was not the bottleneck.
- The workload was not saturating the system: queueing was low, offload variants had zero preemptions, CPU throttling was negligible, and memory-failure counters were zero.
- Seed 42 was already identical. Changing seed or increasing GPU-memory utilization is not a fix.
- The offload runs lost model workers. The artifact collector omitted previous-container logs, so the fatal EngineCore traceback is unavailable.
- EPP v0.9.0 dropped 3,814 parent-dependent block events in the NVMe run and had missing precise-prefix data on about 2.8% of requests, leaving routing performance on the table.
- Keep `gpu-memory-utilization=0.64`, CPU32, concurrency 128, and seed 42 while fixing observability and reliability. Consider `U=0.62`/`0.60` only after a clean zero-restart triplet.

## Documents

- [[2026-07-17 - Initial AgentX offloading tier analysis]] — reconstruction of the initial report/MLflow deep dive and the proposed clean experiment design.
- [[2026-07-18 - U0.64 paired-seed AgentX rerun analysis]] — controlled rerun metrics, worker-failure correlation, EPP multi-tier index defects, and the next isolation matrix.

## Source report and artifacts

- Local initial report: `/private/tmp/kv-cache-experiments/llm-d-qwen3.6-35b-a3b-agentx-report.html`
- Initial normalized artifacts: `/private/tmp/kv-cache-experiments/mlflow-report-runs/`
- Latest rerun artifacts: `/private/tmp/kv-cache-experiments/rerun-2026-07-18/`
- Latest analysis helper: `/private/tmp/kv-cache-experiments/analyze_rerun.py`
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
| Optimized baseline | `d007e8f640354c509700f651bf6d2045` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/d007e8f640354c509700f651bf6d2045?workspace=benchflow) | Clean standalone baseline; approximate routing so not the no-offload control |
| Precise, CPU 32 GiB | `c9d75053b9564393bfdf902331511588` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/c9d75053b9564393bfdf902331511588?workspace=benchflow) | Reject: four model workers restarted; 48 errors |
| Precise, CPU 32 GiB + NVMe | `29a50025c7bc421b802023cd31e22ff9` | [MLflow](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/29a50025c7bc421b802023cd31e22ff9?workspace=benchflow) | Reject: one model worker restarted twice; 15 errors |
| Precise, no offload | pending | — | Rerunning after initial failure |

## Immediate next experiment

Before another full benchmark, run a 10-minute isolation triplet with identical seed/topology:

1. precise, no offload;
2. precise + CPU32;
3. precise + CPU32 + NVMe.

Collect `kubectl logs --previous` for every restarted container. If only offload variants die, repeat one offload case with approximate/no precise EPP to distinguish the vLLM offloader from its EPP event stream.

After the crash and EPP event loss are fixed, run the paired three-seed comparison with:

- eight replicas, TP2;
- concurrency 128 and duration 1,800 seconds;
- `gpu-memory-utilization=0.64`;
- CPU tier 32 GiB in both CPU variants;
- fixed `PYTHONHASHSEED=0`;
- identical EPP and renderer topology;
- fresh deployments and clean caches;
- zero-restarter/zero-EngineCore-error acceptance gates.

Only after the clean comparison should HBM be swept to 0.62/0.60 or concurrency to 160/192.