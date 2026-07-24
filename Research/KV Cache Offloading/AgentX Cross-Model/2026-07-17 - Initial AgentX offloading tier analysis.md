---
title: Initial AgentX offloading tier analysis
date: 2026-07-17
type: research-note
model: Multiple/unspecified
topic: KV Cache Offloading
---

# 2026-07-17 — Initial AgentX offloading tier analysis

**Experiment:** [[00 - Index|KV Cache Offloading]]  
**Related projects:** [[Engineering/Projects/llm-d|llm-d]], [[Engineering/Projects/vLLM|vLLM]]  
**Analysis status:** Complete; source runs are directional, not a clean paired benchmark.

## Question

Why did the precise-prefix-cache variants in `llm-d-qwen3.6-35b-a3b-agentx-report.html` show smaller-than-expected gaps between no offloading, CPU offloading, and CPU+NVMe offloading? What deployment and workload design would produce a clean, meaningful tier comparison for the AgentX trace?

## Executive conclusion

The results are directionally correct. NVMe was working and was not saturated. The modest incremental gap came primarily from cache sizing, reuse timing, incomplete NVMe-aware precise routing, and run-to-run confounders.

The offloading mechanism is visible in both prompt-token source metrics and latency. CPU reduced recomputation relative to HBM-only; NVMe reduced it further. RPS moved less than TTFT because AgentX is a closed-loop paced workload: faster requests finish earlier, but recorded inter-turn gaps prevent all saved compute from turning into additional request throughput.

## Source runs


| Variant                    | Run ID                             | MLflow                                                                                                                                          |
| -------------------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Optimized baseline         | `6ef87d95297842548d7c36eb02f3fcdf` | [Open run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/6ef87d95297842548d7c36eb02f3fcdf?workspace=benchflow) |
| Precise, no offload        | `b3d0fd333acb4b27b5ae2b68124495bd` | [Open run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/b3d0fd333acb4b27b5ae2b68124495bd?workspace=benchflow) |
| Precise, CPU 32 GiB        | `a455cc6580ad401ab37a96bffb6d9150` | [Open run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/a455cc6580ad401ab37a96bffb6d9150?workspace=benchflow) |
| Precise, CPU 32 GiB + NVMe | `e6fd24dc869e4bbbb434af9e653d2fbe` | [Open run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/256/runs/e6fd24dc869e4bbbb434af9e653d2fbe?workspace=benchflow) |


## Observed results


| Precise variant   | RPS  | Avg TTFT | p95 TTFT | Avg E2E  | Local recompute | External KV | Errors |
| ----------------- | ---- | -------- | -------- | -------- | --------------- | ----------- | ------ |
| No offload        | 7.83 | 1,584 ms | 4,619 ms | 6,899 ms | 28.7%           | 0%          | 1      |
| CPU 32 GiB        | 8.48 | 1,248 ms | 3,610 ms | 5,702 ms | 20.4%           | 3.7%        | 1      |
| CPU 32 GiB + NVMe | 8.85 | 1,090 ms | 2,903 ms | 5,062 ms | 12.0%           | 10.5%       | 5      |


Relative changes:

- CPU versus no offload: **+8.3% RPS, -21.3% TTFT, -17.4% E2E**.
- NVMe versus CPU: **+4.4% RPS, -12.6% TTFT, -11.2% E2E**.
- NVMe versus no offload: **+13.1% RPS, -31.2% TTFT, -26.6% E2E**.

## Breakthroughs

### 1. NVMe was active and was not the bottleneck

The NVMe variant reduced recomputed prompt tokens from 20.4% to 12.0% and supplied 10.5% of prompt tokens through external KV transfer.

Per NVMe device during the run:

- read bandwidth: 0.80–0.82 GB/s;
- write bandwidth: 1.17–1.18 GB/s;
- average busy: 38–44%;
- maximum busy: 48–60%;
- model-container CPU: approximately 4.6–5.1 cores out of 16;
- essentially no CPU throttling or memory failures.

Conclusion: increasing I/O threads or replacing the NVMe was not the next optimization.

### 2. CPU32 nearly mirrored the HBM KV tier

At `gpu-memory-utilization=0.7`, each TP2 replica reported about 1.67 million logical KV tokens, approximately 33.6 GiB of aggregate TP2 HBM KV storage.

The 32 GiB CPU tier held approximately 1.68 million logical tokens. CPU and HBM therefore had nearly identical reach, so CPU mostly mirrored blocks still represented in HBM instead of creating a clearly deeper tier.

### 3. CPU32 retained almost the entire normal reuse window

Measured cluster-wide CPU offload churn was 3.64 GB/s, about 455 MB/s per replica:

\[ T_{CPU} = \frac{32\ GiB}{455\ MB/s} \approx 75\ s \]


Observed positive conversation-idle gaps:

- p50: 4.4 s;
- p90: 34.5 s;
- p95: 60.0 s;
- p99: approximately 411–440 s;
- only about 2.8–3.2% exceeded 75 s.

CPU32 therefore captured nearly all normal reuse. NVMe could differentiate itself only on the multi-minute long tail. Those tail requests had long prefixes, explaining why a small fraction of reuse events still represented 10.5% of prompt tokens.

### 4. HBM was not under cache-capacity pressure

Allocated HBM KV utilization averaged 23–27%, peaked around 72%, and produced zero vLLM preemptions. The no-offload run was compute-busy at about 96% average GPU utilization, but compute saturation is not the same as KV-capacity pressure.

Increasing concurrency alone would mainly add queueing unless HBM capacity is deliberately moved toward the active working-set boundary.

### 5. Precise routing was not fully filesystem-aware

The tested combination was vLLM 0.23.0 and EPP v0.9.0. The FS tier did not emit secondary-tier `BlockStored` events, so EPP precisely tracked GPU and CPU placement but could lose knowledge of filesystem placement after eviction from CPU.

Across the two EPP replicas, the log message `Failed to get request key for parent block` appeared:

- no offload: 48 times;
- CPU: 178 times;
- CPU+NVMe: 2,753 times.

This indicated a substantially less complete precise index for the tiered event stream.

### 6. Cross-process filesystem hashing was not deterministic

`PYTHONHASHSEED` was not set. All vLLM instances sharing an FS root need the same fixed value so identical block content produces identical chain-hash filenames.

Use:

```yaml
env:
  PYTHONHASHSEED: "0"
```

## Failed or confounded runs

These failures must remain part of the research record rather than being silently discarded:

- **Optimized baseline:** reject. One EPP was CrashLooping with ten restarts and the run had approximately 90% request errors.
- **CPU+NVMe run:** a model pod exited cleanly and restarted during measurement, causing five request errors and a temporarily missing replica.
- **Unpaired random seeds:** no-offload, CPU, and NVMe used different seeds, so a roughly 4% incremental RPS gap cannot be cleanly attributed from one run each.
- **Different topology:** no-offload spread replicas 3/3/2 over three nodes, while CPU and NVMe used two nodes with four replicas each; the exact node pair also differed.
- **Render capacity:** all runs saw initial render timeouts; about 36 NVMe requests began without precise token data.
- **Telemetry:** CPU and NVMe runs lacked DCGM GPU metrics.
- **Artifact hygiene:** a collected pod description exposed an unredacted Hugging Face token. Rotate it and redact secret-derived environment values in future artifact collection.

## Recommended controlled experiment

Use this primary three-way comparison:

```text
gpu-memory-utilization: 0.64
cpu_bytes_to_use:       34359738368  # 32 GiB
concurrency:            128
duration:               1800 seconds
replicas:               8
tensor parallelism:     2
```

Apply CPU32 identically to CPU-only and CPU+NVMe.

At `gpu-memory-utilization=0.64`, predicted HBM KV capacity is about 1.17 million logical tokens, or about 23–24 GiB per TP2 replica. This places the observed peak working set near the HBM boundary while retaining enough active-sequence capacity to avoid preemption.

Expected tier separation:

```text
HBM-only: misses reuse around the 60-second boundary
CPU32:    catches most reuse through that boundary
NVMe:     catches the multi-minute long tail
```

Do not use CPU16 for this comparison because it would be smaller than TP2 HBM KV capacity. Do not use CPU64 in the main comparison because it would absorb more of the long tail and hide NVMe. If CPU32 and NVMe remain too close, try 28 GiB (`30064771072`) next.

## Router and filesystem requirements

Use a vLLM version newer than 0.23 that supports filesystem KV events and a compatible router build:

```json
{
  "type": "fs",
  "root_dir": "/mnt/nvme-kv-cache",
  "n_read_threads": 16,
  "n_write_threads": 16,
  "enable_kv_events": true
}
```

Use tier scoring approximately like:

```yaml
indexerConfig:
  backendConfigs:
    - name: gpu
      weight: 1.0
    - name: cpu
      weight: 0.8
    - name: fs
      weight: 0.4
```

This should prefer an equally long GPU prefix over CPU and CPU over FS, while still routing FS-only long matches to the correct replica.

Scale tokenizer/render from three to four replicas.

## Workload procedure

- Pin one dataset revision.
- Pin three random seeds.
- Run every variant once per seed.
- Randomize order to reduce temporal bias:

```text
Seed A: No offload -> CPU -> NVMe
Seed B: NVMe -> No offload -> CPU
Seed C: CPU -> NVMe -> No offload
```

- Use identical two-node topology for all variants.
- Use fresh deployments and cleaned cache directories for each run.
- After the clean concurrency-128 comparison, sweep 128, 160, and 192. Concurrency 160 is the likely useful capacity point; use 192 only if GPU utilization remains below about 90% and preemptions remain zero.

## Run acceptance gates

Reject any run with:

- any model or EPP restart;
- request error rate above 0.1%;
- missing required GPU metrics;
- any vLLM preemption;
- a NotReady node or model replica;
- NVMe busy above 80–85%;
- filesystem free space below 20%;
- persistent precise-index parent-key failures materially above the no-offload rate;
- mismatched random seed, dataset revision, topology, image, renderer count, or cache-cleaning procedure.

## Next decisions

1. Upgrade the filesystem-event and router combination.
2. Run the paired `U=0.64`, CPU32, concurrency-128 matrix across three seeds.
3. Compare TTFT, E2E, recomputed prompt tokens, external-KV tokens, queueing, and cache-source distributions before emphasizing RPS.
4. Only tune NVMe threads or hardware if device busy approaches the acceptance threshold.
5. Update [[00 - Index|the experiment index]] after every accepted/rejected run batch and whenever the working conclusion changes.