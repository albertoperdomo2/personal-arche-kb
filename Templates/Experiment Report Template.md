---
title: Experiment Report Template
type: template
topic: Research experiments
---

# YYYY-MM-DD — <Model / mechanism / workload> experiment

## Executive summary

State the research question, configurations and MLflow runs, model/workload shape, deployment fingerprint, mechanism under test, and the headline measured result.

## Validity verdict

**Verdict: Valid / Conditionally valid / Invalid / Inconclusive**

Explain the decision using evidence: saturation, request/session completion, errors, configuration drift, repetitions, cutoff behavior, or missing telemetry. Do not rank invalid runs.

## Main takeaways

- **Observation:** <measured result with units and provenance>
- **Observation:** <measured result with units and provenance>
- **Inference:** <carefully bounded interpretation>
- **Failure or limitation:** <what cannot be concluded>
- **Next decision:** <what this changes>

## Headline metrics

Declare the baseline explicitly (normally No offload/HBM-only). Use absolute values and deltas versus baseline. Use `N/A` for invalid or non-comparable deltas.

| Configuration | Run | Throughput (req/s) | Δ throughput vs baseline | Output (tokens/s) | Mean TTFT (ms) | Mean E2E (s) | Error/cancel rate (%) | Completed sessions | Validity |
|---|---|---:|---:|---:|---:|---:|---:|---:|---|
| Baseline | [MLflow](<url>) |  |  |  |  |  |  |  |  |
| <Configuration> | [MLflow](<url>) |  |  |  |  |  |  |  |  |

## Experiment and deployment fingerprint

- Model and revision:
- vLLM / runtime version:
- GPU type/count, TP/PP:
- GPU memory utilization and context length:
- CPU tier and secondary tier:
- `/dev/shm`, mounts, PVC/hostPath, offload spec:
- Workload profile, concurrency, duration, seed, trace/dataset:
- Cache-cleaning and warmup state:

## Outcome evidence

Choose representations by question: grouped bars for categorical outcomes; aligned time-series or small multiples for run behavior; stacked bars/areas for composition; points for relationships; rules for thresholds; error bars/bands for uncertainty. Use white backgrounds and Category10 colors.

Figure 1 shows <headline outcome>. Provenance: <artifact/metric and native cadence>.

```vega-lite
<validated inline Vega-Lite JSON>
```

Interpretation: <what the figure establishes and what it does not establish>.

## Request, session, latency, and saturation evidence

Include request/session completion and cancellation, running/waiting requests, queueing, TTFT/ITL/E2E latency distributions, KV occupancy, preemptions, and warning/error onset. Plot the finest source grain available; avoid aggregation or downsampling unless required by metric semantics or an explicit renderer limit, and document any aggregation.

## Mechanism and secondary-tier evidence

Proactively plot the mechanism metrics that can explain the result. For CPU offload consider KV occupancy, CPU↔GPU transfer rates, prompt-token sources, CPU pressure, and working set. For NVMe add read/write bandwidth, IOPS, latency, queue depth, busy time, capacity, and filesystem usage. For CephFS add client reads/writes, operation latency, PVC usage, MDS/OSD health, mount health, and store-refusal warnings. Align mechanism signals with throughput, queueing, prompt sources, and latency where readable. State unavailable telemetry explicitly.

## Conclusions

### Established by the data

- 

### Hypotheses / uncertainty

- 

## Next experiment

Specify the smallest discriminating experiment: controls, repetitions, concurrency/pressure steps, deployment changes, required telemetry, and acceptance criteria.

## Run registry and provenance

| Configuration | MLflow run | Artifact path | Disposition |
|---|---|---|---|
|  |  |  |  |