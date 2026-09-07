---
title: "vLLM binary selective-load opt-out cluster validation"
date: 2026-09-04
type: experiment
experiment: "Selective KV loading and offloading"
status: conditionally-valid
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v1@sha256:4557d9e0bcf663562ba446064b35f258775cd4e7ba068eb32082a162f9cc1a41"
vllm_version: "0.27.0 with local selective-load patch"
tensor_parallelism: 8
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 131072
cpu_bytes: 274877906944
offload_spec: "TieringOffloadingSpec"
secondary_tier: "filesystem at /mnt/nvme-kv-cache"
secondary_tier_threads: "64 read, 64 write"
cache_cleaning_state: "HBM-only reset unavailable; /reset_prefix_cache returned HTTP 404"
namespace: benchflow
cluster_context: "benchflow/api-diadochos-ibm-rhperfscale-org:6443/system:admin"
---

# vLLM binary selective-load opt-out cluster validation

## Executive summary

This experiment tested the first vLLM selective-loading contract on the live
`benchflow` deployment: an explicit `kv_load_tiers=[]` must skip all
offload-source lookup, while `max_offload_tokens=0` must independently prevent
new KV storage. Local HBM prefix-cache reuse must remain available.

The deployed image was
`quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v1`, running one replica
with tensor parallelism 8 on eight H100 GPUs. Requests were sent from a temporary
in-cluster curl pod to the workload service over HTTPS. vLLM's own
`/metrics` endpoint supplied the mechanism evidence.

The combined opt-out request completed with HTTP 200, caused no offload-manager
lookup and no KV transfer, and recomputed a previously unseen prompt. Repeating
a prompt still resident in HBM with the same opt-out produced 1,024 cached
tokens without consulting the offload manager.

## Validity verdict: Conditionally valid

The experiment is valid for the binary scheduler behavior under test:
`kv_load_tiers=[]` bypassed offload lookup, `max_offload_tokens=0` prevented
storage, and local HBM reuse remained intact.

It is conditional for the narrower end-to-end CPU-resident-hit scenario. The
running server did not expose `/reset_prefix_cache`, so the test could not
evict only the HBM copy while preserving the CPU copy and then replay the same
prefix. No CPU-to-GPU transfer was observed or claimed.

## Main takeaways

- **Measured:** A normal cache-miss request increased the offload connector
  lookup counter by 2 and the tiering lookup counter by 1.
- **Measured:** A distinct cache-miss request with `kv_load_tiers=[]` changed
  neither lookup counter.
- **Measured:** Adding `max_offload_tokens=0` kept GPU-to-CPU/store bytes
  unchanged during both opt-out requests.
- **Measured:** Repeating an HBM-resident prefix under the same load opt-out
  returned `cached_tokens=1024` while offload lookup counters remained
  unchanged.
- **Inference:** The binary opt-out is enforced at the intended scheduler
  boundary: local computed-token discovery still runs, but offload-manager
  lookup does not.
- **Not established:** This run did not measure whether recomputation is faster
  than CPU loading under CPU pressure. That calibration belongs to the router
  policy experiment.

## Deployment and request contract

| Dimension | Value |
|---|---|
| Deployment | `vllm-lookahead-bda4e630e4-kserve` |
| Workload service | `vllm-lookahead-bda4e630e4-kserve-workload-svc:8000` |
| Image | `quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v1` |
| Image digest | `sha256:4557d9e0bcf663562ba446064b35f258775cd4e7ba068eb32082a162f9cc1a41` |
| Model | `nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8` |
| Replica / TP | 1 replica / TP 8 |
| GPU memory utilization | 0.8 |
| Maximum model length | 131,072 tokens |
| CPU offload allocation | 274,877,906,944 bytes (256 GiB) |
| Secondary tier | Filesystem, `/mnt/nvme-kv-cache` |
| Secondary-tier workers | 64 read / 64 write |
| Prefix caching | Enabled |
| Temporary client pod | `selective-load-validation-20260904` |

The validated request extension was:

```json
{
  "kv_transfer_params": {
    "kv_load_tiers": [],
    "max_offload_tokens": 0
  }
}
```

## Headline metrics

The pre-traffic state was the functional baseline: transfer bytes, prefix
queries/hits, and prompt-token source counters were all zero. This was not a
performance benchmark, so latency, throughput, concurrency, and deltas versus
an HBM-only deployment were not collected.

| Request stage | Prompt tokens | HTTP | Connector lookup delta | Tiering lookup delta | CPU→GPU byte delta | GPU→CPU/store byte delta | Cached tokens |
|---|---:|---:|---:|---:|---:|---:|---:|
| Seed prefix | 1,025 | 200 | +2 | +1 | 0 | +134,217,728 | 0 |
| Normal distinct miss | 2,049 | 200 | +2 | +1 | 0 | +268,435,456 | 0 |
| Combined opt-out, distinct miss | 3,073 | 200 | 0 | 0 | 0 | 0 | 0 |
| Combined opt-out, HBM-resident repeat | 1,025 | 200 | 0 | 0 | 0 | 0 | 1,024 |

Final prompt-token source totals were:

| Source | Tokens |
|---|---:|
| Local compute | 6,148 |
| Local HBM cache hit | 1,024 |
| External KV transfer | 0 |
| Prefix-cache queries | 7,172 |

## Evidence

Figure 1 compares lookup calls caused by the normal miss and the two opt-out
requests. Values are counter deltas scraped directly from vLLM `/metrics`
immediately before and after each request.

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": "Figure 1 — Offload lookup calls per validation request",
  "width": 520,
  "height": 280,
  "data": {
    "values": [
      {"request": "Normal miss", "counter": "Connector lookup", "calls": 2},
      {"request": "Normal miss", "counter": "Tiering lookup", "calls": 1},
      {"request": "Opt-out miss", "counter": "Connector lookup", "calls": 0},
      {"request": "Opt-out miss", "counter": "Tiering lookup", "calls": 0},
      {"request": "Opt-out HBM hit", "counter": "Connector lookup", "calls": 0},
      {"request": "Opt-out HBM hit", "counter": "Tiering lookup", "calls": 0}
    ]
  },
  "mark": {"type": "bar"},
  "encoding": {
    "x": {
      "field": "request",
      "type": "nominal",
      "title": "Request scenario"
    },
    "xOffset": {
      "field": "counter"
    },
    "y": {
      "field": "calls",
      "type": "quantitative",
      "title": "Lookup calls (count)",
      "scale": {"zero": true}
    },
    "color": {
      "field": "counter",
      "type": "nominal",
      "title": "Metric",
      "scale": {"scheme": "category10"}
    }
  }
}
```

Figure 1 shows the expected control/experiment separation: the ordinary miss
entered both lookup layers, while both requests carrying an empty load-tier
list caused zero lookups. The HBM-hit request still reused 1,024 local tokens,
so zero offload lookups did not mean prefix caching was globally disabled.

Additional checks:

- The installed package contained `TierFilter.is_empty` and the corresponding
  scheduler early-return branch.
- CPU-to-GPU bytes remained zero for the entire validation.
- GPU-to-CPU/store bytes reached 402,653,184 after the two setup/control
  requests and did not change during either opt-out request.
- The vLLM pod remained ready with zero restarts.
- Recent logs contained no `ERROR`, `Traceback`, `AssertionError`,
  `kv_load_tiers`, or `max_offload_tokens` error matches.

## Validity and limitations

The attempted `POST /reset_prefix_cache` returned HTTP 404. Restarting or
mutating the live deployment solely to manufacture a CPU-resident-only prefix
was intentionally avoided. Consequently, unchanged lookup counters are the
direct proof that the offload manager was skipped; the run does not include a
paired normal CPU-hit promotion.

The normal distinct-miss control accidentally placed
`max_offload_tokens: 0` at the request top level rather than under
`kv_transfer_params`. vLLM ignored that field and stored the result. This does
not affect the control's intended evidence—normal requests invoke offload
lookup—but it explains the +268,435,456-byte store delta and must not be treated
as a test of the offload cap.

Prometheus was not required for this functional check because the authoritative
vLLM counters were directly scrapeable. CPU utilization/pressure, transfer
latency, request latency, and throughput were not collected.

## Conclusion

The deployed v0.27.0-based patch satisfies the initial binary contract needed
by router development:

1. `kv_load_tiers=[]` makes the request behave as though no external KV
   offload source is available.
2. It does not disable local HBM prefix-cache reuse.
3. `max_offload_tokens=0` independently prevents newly computed KV from being
   offloaded when correctly nested under `kv_transfer_params`.

This is enough to begin llm-d-router gating and payload-propagation tests behind
a backend capability gate. It is not yet evidence for non-empty source
selection such as CPU-only or STORAGE-only policy.

## Next experiment

Run a controlled deployment that can evict HBM state without deleting CPU-tier
state, or add a test-only/admin-safe cache reset mechanism. Then execute the
same prefix twice:

1. Seed and offload the prefix.
2. Evict only the HBM copy.
3. Replay normally and verify a CPU-to-GPU transfer.
4. Evict HBM again.
5. Replay with `kv_load_tiers=[]` and verify zero lookup/transfer plus local
   recomputation.

After that mechanism check, calibrate the router's static load threshold under
increasing CPU pressure using repeated workloads, direct latency/throughput
measurements, and CPU/memory/transfer telemetry.

## Cleanup

The temporary client pod `selective-load-validation-20260904` was deleted and
verified absent. The serving deployment remained `1/1` ready after cleanup.

## Related

- [[00 - Index]]
- [[01 - Initial implementation plan]]
- [[02 - vLLM primary-tier selective loading]]
- [[03 - llm-d-router selective load and offload policy and wiring]]
- [[04 - vLLM selective load - problem statement]]