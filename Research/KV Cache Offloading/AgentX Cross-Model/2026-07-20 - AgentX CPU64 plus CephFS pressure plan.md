---
title: AgentX CPU64 plus CephFS pressure plan
date: 2026-07-20
type: research-plan
model: Multiple/unspecified
topic: KV Cache Offloading
---

# AgentX CPU64 + CephFS pressure plan

Date: 2026-07-20

## Decision

For one Qwen/Qwen3.6-35B-A3B replica at TP=2, `gpu_memory_utilization=0.9`, a 64 GiB CPU primary tier, and a CephFS secondary tier, use **AgentX MVP concurrency 128** as the first full run.

Do not use concurrency 32 for this purpose. It may cause eager offload writes, but it does not force the retained working set beyond HBM, much less beyond HBM plus CPU64.

Keep the 1,800-second AgentX profiling window initially. A longer run can accumulate more writes, but it does not repair a retention failure or prove useful CephFS readback.

## Required profile and deployment changes

Clone the canonical benchmark profile so the standard AgentX profile remains a stable control:

```yaml
apiVersion: benchflow.io/v1alpha1
kind: BenchmarkProfile
metadata:
  name: aiperf-agentx-inference-cephfs-pressure
spec:
  tool: aiperf
  env:
    AIPERF_HTTP_SSL_VERIFY: "false"
  requirements:
    min_max_model_len: 131072
  aiperf:
    scenario: inferencex-agentx-mvp
    public_dataset: weka_hf
    hf_weka_repo: semianalysisai/cc-traces-weka-with-subagents-060826
    endpoint_type: chat
    endpoint_path: /v1/chat/completions
    streaming: true
    use_server_token_count: true
    max_context_length: 131072
    concurrency: 128
    benchmark_duration: 1800
    max_seconds: 7200
    random_seed: 42
```

The workload change is necessary but not sufficient. The checked-in deployment/experiment also need:

- change `--max-model-len=8192` to `--max-model-len=131072`; 8,192 conflicts with AgentX's declared minimum and falsifies/rejects the long-context trace;
- change the Qwen override from `--gpu-memory-utilization=0.55` to `--gpu-memory-utilization=0.9`;
- use the vLLM 0.24 image that contains the tested multi-tier implementation rather than the checked-in v0.23.0 override;
- explicitly add `--mamba-cache-mode=align` for this hybrid model, even though recent vLLM builds may infer it when prefix caching is enabled;
- keep `cpu_bytes_to_use=68719476736` and `shared_memory_size: 128Gi`;
- keep `--max-num-seqs=256`; it is already above the nominal tree concurrency and should not be raised unless scheduler evidence calls for it.

## Capacity argument

The prior vLLM 0.24 run at utilization 0.64 reported 1,197,193 GPU KV tokens per TP=2 replica. Projecting the observed allocation to utilization 0.9 gives roughly 3.3M GPU tokens; the startup log from the new run is the ground truth.

For BF16 attention KV, Qwen's nominal growing-cache cost is:

$$
c_{KV}=2L_{attn}H_{kv}d_{head}b_{kv}
       =2\cdot10\cdot2\cdot256\cdot2
       =20{,}480\ \text{bytes/token}.
$$

CPU64 therefore has a theoretical upper bound:

$$
T_{CPU64}\le\frac{68{,}719{,}476{,}736}{20{,}480}
\approx3.36\ \text{M tokens}.
$$

The real tiering capacity may be lower because Qwen's aligned hybrid chunks and recurrent-state checkpoints add granularity/overhead. Using the upper bound is conservative for proving Ceph spill.

With mean AgentX input around 62.3k tokens and mean output around 0.68k, use approximately 63k retained tokens per trajectory as the simple lower-order estimate:

$$
T_{corpus}(C)\approx C\cdot63{,}000.
$$

That gives 2.02M tokens at concurrency 32, 4.03M at 64, 6.05M at 96, and 8.06M at 128. The conservative HBM+CPU ceiling is about 6.6M tokens, so concurrency 128 crosses:

$$
T_{corpus}>T_{GPU}+T_{CPU}.
$$

Figure 1 compares the estimated retained corpus against the two capacity landmarks. Provenance: the previous vLLM 0.24 startup log, the previous AgentX request-length distribution, and the architecture formula above; values are sizing estimates, not measured Ceph traffic.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 1. Estimated AgentX corpus versus KV tier capacity",
  "width": 680,
  "height": 320,
  "data": {
    "values": [
      {"item": "HBM at U=0.9", "kind": "Capacity", "tokens_m": 3.30},
      {"item": "HBM + CPU64", "kind": "Capacity", "tokens_m": 6.66},
      {"item": "AgentX C=32", "kind": "Workload", "tokens_m": 2.02},
      {"item": "AgentX C=64", "kind": "Workload", "tokens_m": 4.03},
      {"item": "AgentX C=96", "kind": "Workload", "tokens_m": 6.05},
      {"item": "AgentX C=128", "kind": "Workload", "tokens_m": 8.06}
    ]
  },
  "mark": {"type": "bar", "tooltip": true},
  "encoding": {
    "x": {
      "field": "item",
      "type": "nominal",
      "sort": ["HBM at U=0.9", "HBM + CPU64", "AgentX C=32", "AgentX C=64", "AgentX C=96", "AgentX C=128"],
      "axis": {"title": "Capacity or workload configuration"}
    },
    "y": {
      "field": "tokens_m",
      "type": "quantitative",
      "scale": {"zero": true},
      "axis": {"title": "Estimated retained KV tokens (millions)"}
    },
    "color": {
      "field": "kind",
      "type": "nominal",
      "scale": {"scheme": "category10"},
      "legend": {"title": "Estimate type"}
    }
  }
}
```

Figure 1 explains why 128 is the first conservative choice. AgentX session-tree concurrency is not identical to simultaneous model requests. The earlier concurrency-128 run measured about 43.4 active model requests in the CPU+secondary-tier case. At roughly 63k tokens each, that is a live set near 2.7M tokens, below the projected 3.3M-token HBM capacity. This suggests the desired utilization window: the live set fits HBM while the retained corpus exceeds HBM+CPU.

That effective-concurrency ratio came from an eight-replica run and is a prior, not a guarantee for a single replica. Validate it.

## Ceph writes are not Ceph hits

Tiered offload stores completed blocks eagerly. Nonzero Ceph writes prove spill, not benefit. Useful Ceph readback requires a prefix to leave CPU before it is referenced again while surviving on CephFS.

Measure the CPU retention clock:

$$
t_{CPU,retain}=\frac{64\ \text{GiB}}{\dot B_{GPU\rightarrow CPU}}.
$$

For a block to be served from CephFS rather than CPU, at least part of the reuse-gap distribution must exceed this CPU retention time. At the same time, CephFS retention must exceed those reuse gaps. AgentX caps recorded request-start idle gaps at 60 seconds, although subagent gating and queue time can make some block reuse gaps much longer.

This means concurrency 128 guarantees the capacity-side spill estimate, but no workload-only setting can guarantee healthy Ceph read hits without observing churn and reuse gaps. If CPU64 retains every useful prefix until its next use, CephFS will be a write sink even though the shelf inequality is satisfied.

## Calibration and acceptance gates

Use the full concurrency-128 run first. If it queues excessively, run a short valid calibration sweep at 96, 112, and 128 and choose the lowest point that produces sustained Ceph reads.

Accept the point only when all of the following hold during the second half of profiling:

- CephFS **read** bytes/s and read IOPS are sustained; write-only activity is insufficient;
- CPU-to-GPU connector bytes are of the same order as GPU-to-CPU bytes, not near zero;
- `external_kv_transfer` prompt tokens are material and local-compute share falls relative to a paired CPU64-only control;
- GPU KV usage is active but not pinned at 100%;
- waiting requests are not persistently high and preemptions remain near zero;
- no context overflows, model restarts, memory failures, or meaningful request-error rate;
- compute `64 GiB / GPU→CPU write rate` and compare it with measured block reuse gaps / `think time + TTFT`.

If concurrency 128 yields Ceph writes but negligible reads, the honest next lever is CPU capacity, not benchmark duration. CPU32 shortens the primary-tier retention clock and increases the chance that reusable prefixes reach CephFS. If CPU64 is a fixed requirement, concurrency 160 is the upper diagnostic point; it is near the projected HBM live-set boundary and must be stopped if queueing or preemption begins to dominate.

## Metrics to preserve

Use the detailed metrics profile plus the CPU offload report metrics. The decisive series are:

- `prompt_tokens_rate_by_source`;
- `kv_offload_bytes_rate_by_pod` and `kv_offload_time_rate_by_pod`;
- `vllm:kv_offload_cpu_cache_usage_perc` (add this query to Benchflow if it is not exported);
- `storage_ceph_pool_bytes_per_second_by_pool`;
- `storage_ceph_pool_iops_by_pool`;
- Ceph OSD latency and health;
- GPU KV-cache utilization, running/waiting requests, waiting reasons, and preemptions.

## Sources

- Workload: `/Users/aperdomo/workspace/redhat/benchflow/profiles/benchmark/aiperf-agentx-inference.yaml`
- Deployment: `/Users/aperdomo/workspace/redhat/benchflow/profiles/deployment/rhoai/multi-tier-offloading-cephfs.yaml`
- Experiment: `/Users/aperdomo/workspace/redhat/benchflow/experiments/rhoai/cpu-offloading.yaml`
- Methodology: [Two Equations for Forcing KV-Cache Offload](https://www.albertoperdomo.me/posts/kv-cache-offloading-experiments-math)
- Prior measurements: [[2026-07-19 - vLLM 0.24 fixed-EPP clean rerun analysis]]