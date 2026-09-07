---
title: "Selective-load binary opt-out first AgentX run — diagnostic invalidation"
date: "2026-09-07"
type: "experiment-report"
topic: "KV Cache Offloading"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
status: "invalid-for-calibration"
mlflow_run_id: "fba296914c214852b58bbfbf31230f1a"
---

# Selective-load binary opt-out first AgentX run — diagnostic invalidation

## Decision

MLflow run [fba296914c214852b58bbfbf31230f1a](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/fba296914c214852b58bbfbf31230f1a?workspace=benchflow) is valid as a mechanism and failure-mode observation, but invalid for deriving a selective-load threshold or attributing a vLLM image regression.

The deployed policy was an intentionally pathological steady state:

- `loadPolicy: disable` caused every request to carry an empty load-tier selection.
- `offloadPolicy: preserve` left GPU-to-CPU stores enabled.
- Telemetry was enabled with `offload_telemetry_sample_rate: 1.0`.

This combination paid the store and connector costs while allowing no CPU-to-GPU reuse.

## Workload

- AgentX MVP replay, Weka dataset
- Nemotron Ultra 253B FP8
- TP8, one H100 replica
- concurrency 64
- 1,800-second profiling interval
- first-turn-prefix cache bust
- prompts averaged 69,674 tokens; p50 was 74,930 tokens

## Client outcome

| Metric | Run |
|---|---:|
| Completed profiling requests | 38 |
| Cancelled in-flight requests at timeout | 54 |
| Mean request latency | 908.8 s |
| P95 request latency | 1,512.3 s |
| Mean TTFT | 325.1 s |
| P95 TTFT | 885.0 s |
| Mean ITL | 1.951 s |
| Output throughput | 5.99 token/s |
| Request throughput | 0.0207 request/s |

Warmup was already pathological: 66 one-token warmup requests required 2,473.7 seconds, with mean TTFT 392.0 seconds.

## Mechanism evidence

The selective-load behavior itself worked:

- 693 scheduler resolutions were observed.
- All 693 resolved as `disabled`.
- Scheduler resolution averaged 4.8 microseconds.
- There were zero load submissions, zero pending load bytes, and zero externally loaded prompt tokens.

The server remained healthy: the model pod had zero restarts, no CPU throttling, and no memory allocation failures.

The actual bottleneck was sustained engine capacity pressure:

- profiling server scrape: about 26.6 running and 20.7 waiting requests on average
- waiting reason was almost entirely `capacity`, not KV-transfer deferral
- GPU KV-cache utilization averaged 92.5% and reached 99% late in the run
- generation throughput repeatedly fell to roughly 2.5–5.4 token/s while 26–28 requests ran and 26–28 waited

Offload stores remained active:

- 920.7 GB was stored GPU-to-CPU over the full collected window
- mean active copy time was approximately 4.6 ms per rank, with roughly 24 GB/s active transfer bandwidth
- persistent store backlog was not observed; the pending-job gauge had p50/p95 zero and maximum two jobs per rank

## Telemetry assessment

The new metrics are partly useful:

- scheduler outcome and resolution time correctly prove that the binary opt-out was applied
- copy duration and payload size are useful for transfer-cost calibration
- pending load/store gauges correctly distinguish the directions
- request records correctly connect policy outcome, prompt size, and completion state

The `kv_offload_transfer_turnaround_seconds` and derived non-copy time must not be interpreted as transfer queueing. They measure submission until vLLM later observes completion, so they include engine-step polling delay. In this run the mean was about 4.89 seconds while the copy itself was about 4.6 ms; that gap primarily reflects delayed observation and cannot be used as a CPU-transfer queue estimate.

Full-sample logging produced 16,420 records and an 8.3 MB model log over approximately 73 minutes. This is not enough by itself to explain the multi-minute latency, but the implementation also emits and aggregates worker gauge metadata on every engine step. Its overhead remains unquantified and requires an A/B test.

## Comparison and limitation

A nearby same-model, TP8, concurrency-64 no-connector run, `65a2930350254d459ff8cadce67f22f0`, used the same AgentX scenario and random seed. It reported 266.8-second mean latency, 118.5-second mean TTFT, 336-ms mean ITL, and 54.8 output token/s. The current run was 3.41× slower in mean latency and 5.81× worse in ITL, with 89% lower output throughput.

This is directional, not causal: the control used a different deployment profile and the lookahead-v2 image. It cannot separate connector store overhead, telemetry overhead, image differences, and policy effects.

## Required follow-up

Run a controlled matrix with identical image, model, placement, workload seed, and router path:

1. Connector enabled, loads preserved, telemetry disabled.
2. Connector enabled, loads preserved, telemetry enabled with sample rate zero.
3. Connector enabled, loads disabled, stores disabled with `max_offload_tokens: 0`.
4. Connector enabled, loads disabled, stores preserved.
5. Optionally repeat cell 2 with sampled request logs at 1–5%.

Cells 1 and 2 isolate metric-export overhead. Cells 3 and 4 isolate the independent cost of preserving stores when loads are rejected. A stock/no-connector cell is useful as an external baseline but does not replace these same-image controls.

Do not calibrate a selective-load threshold from this run.