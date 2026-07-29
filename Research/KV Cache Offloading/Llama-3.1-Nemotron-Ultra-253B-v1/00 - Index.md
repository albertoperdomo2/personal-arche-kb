---
title: "Llama-3.1-Nemotron-Ultra-253B-v1 KV Cache Offloading"
date: "2026-07-29"
type: "research-index"
status: "active"
model: "RedHatAI/Llama-3_1-Nemotron-Ultra-253B-v1"
reports: 1
---

# Llama-3.1-Nemotron-Ultra-253B-v1

## Reports

- [[2026-07-29 - Baseline vs CPU offload experiment|Baseline vs CPU offload experiment]]

## Current conclusion

The first AgentX comparison does not demonstrate an end-user benefit from a 64 GiB CPU tier. The tier was active and accepted about 4.85 TB of GPU-to-CPU writes, but only 3.78 GB returned to GPU and the external-KV prompt-token source remained zero. The workload stayed capacity-bound and only 2 of 32 root sessions completed in either run.

## Next experiment

Validate the CPU load path with an exact-prefix replay, then repeat baseline/CPU comparisons across lower concurrency and larger CPU capacities with three paired repetitions on matched nodes.

## Run registry

| Configuration | Run |
|---|---|
| Baseline | [MLflow `c2720531`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/c272053125704ddea1941213a8e84f5d?workspace=benchflow) |
| CPU offload 64 GiB | [MLflow `98586342`](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/325/runs/9858634252fa410da396f69a3eaf6816?workspace=benchflow) |