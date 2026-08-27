---
title: "COSTAR Continuation Readiness A2 — bounded soft-TTL frontier"
date: "2026-08-27"
type: "research-experiment"
experiment: "COSTAR Continuation Readiness — A2"
status: "conditionally-valid-narrow-positive"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-costar-oracle-trace-v1@sha256:0e79705305e63b50ac80454641c4ca277014ae0c47a664df5784a46eb079e17f"
vllm_version: "v0.27.0 experimental trace build plus offline TTL replay"
tensor_parallelism: 8
replicas: 1
gpu: "8x H100"
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: "default / not explicitly set"
concurrency: [32, 64]
cpu_bytes_reference: 274877906944
cpu_capacity_gib_sweep: [192, 256, 320]
cpu_blocks_reference: 131072
kv_bytes_per_chunk: 2097152
ttl_seconds: [0, 1, 3, 10, 30, 60, 120, 300, 600, 1800]
offload_spec: "TieringOffloadingSpec"
secondary_tier: "filesystem on local NVMe"
secondary_tier_threads: "64 read / 64 write"
workload: "AIPerf AgentX/Weka agentic replay"
random_seed: 20260707
duration_seconds: 1800
cache_cleaning_state: "not established; unchanged from accepted corpus"
---

# COSTAR Continuation Readiness A2 — bounded soft-TTL frontier

## Executive summary

A2 asks whether an online-realizable, bounded continuation TTL can preserve useful CPU KV without reproducing A1's cache-wide false protection.

The answer is **narrowly positive against always-admit LRU, but negative as a standalone production policy**.

At the reference 256 GiB CPU capacity:

- c32's best safe point is 300 s: 29/42 requests ready, one net miss worse than recorded, and 21.65% gross oracle-service recovery;
- c64's best request-safety point is 10 s: 545/789 ready, 32 net misses worse than recorded, and 23.06% gross recovery;
- 30 s is the most robust common setting: it is identical to LRU at c32 and improves c64 by three complete requests, eight fewer net misses, and 23.97 additional gross service seconds avoided;
- no TTL beats recorded physical placement at the reference capacity.

The capacity sweep is the reason A2 is not a complete rejection. At 192, 256, and 320 GiB, 30 s is neutral versus no-TTL LRU on c32 and improves c64 by 6, 3, and 10 complete external requests. It therefore **weakly dominates LRU across the tested capacity range**, and the improvement is not a single knife-edge point.

Long TTLs are unsafe. At c64, 1,800 s protects 80.45% of time-weighted CPU capacity, completes only 75/789 external targets, and creates 502 net new misses.

**Decision:** keep 30 s as the bounded block-level TTL baseline for A3. Do not implement it live yet. A3 must determine whether request-readiness-aware allocation can turn the small robust signal into positive request-level outcomes versus recorded placement.

## Validity verdict

# Conditionally valid

The replay is valid for comparing static soft TTLs under the accepted exogenous-arrival, equal-capacity trace model:

- every exported turn is protected, including the last/right-censored turn;
- no oracle resume label is used;
- protection starts only at the recorded previous-turn finish;
- protection ends on observed demand, TTL expiry, or trace end;
- admission remains mandatory;
- protected state remains evictable when no unprotected victim exists;
- no proactive reads, bypass, reserve, lease, or pin are introduced;
- all policies receive the same CPU-ready arrivals at each capacity.

The result is not an end-to-end latency experiment. Counterfactual demand-fill feedback, live ref-count/in-flight protections, and the service duration of newly created reads are absent. Gross service avoided is therefore diagnostic; request-level net miss delta is the safety metric.

The 192/320 GiB cells change simulated CPU capacity while keeping the recorded arrival stream. They test policy pressure sensitivity, not a physically rerun deployment.

## Main takeaways

- **Measured:** 30 s weakly dominates no-TTL LRU over all six corpus/capacity cells: c32 is unchanged and c64 gains 6/3/10 complete requests at 192/256/320 GiB.
- **Measured:** at reference capacity, c64 30 s avoids 41 recorded reads but creates 78 new misses, for a +37 net regression versus recorded placement.
- **Measured:** the c64 request-optimal region is short: 10–30 s. At 60 s, net regression worsens to +57.
- **Measured:** c32 needs much longer protection before placement changes; 300 s improves LRU by two complete requests at reference capacity but is extremely inefficient and unsafe on c64.
- **Measured:** longer temporal coverage is not monotonic value. c64 300 s covers 95.74% of observed resume edges but completes only 416/789 requests.
- **Inference:** continuation survival time alone cannot decide which working set deserves residency. It needs pressure and request-value/readiness structure.
- **Decision:** A2 passes only as a baseline-selection gate; live retention remains gated on A3.

## Replay semantics

After every exported turn finishes, its known reusable working set receives a binary soft-priority bonus for TTL seconds. This includes turns that never visibly return, because an online policy does not know they are terminal.

When the same concrete continuation resumes, its prior protection is cleared after the first external-demand observation. The newly completed turn later starts a fresh TTL.

If an ordinary CPU arrival requires capacity:

1. evict ordinary LRU state first;
2. if all state is protected, evict protected state;
3. never reject the admission.

All blocks of one continuation receive the same TTL value. Shared resident keys are charged once even when they support multiple continuations.

This is deliberately simpler than A3. A2 tests whether elapsed time alone is enough.

## Reference-capacity frontier

### c32, 256 GiB

| TTL | Resume edges inside TTL | Complete external targets | Reads avoided | Gross service avoided | Oracle recovery | Net miss delta | Protected GiB-hours |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 0 s | 0.00% | 27/42 | 2 | 5.620 s | 15.42% | +3 | 0.00 |
| 1 s | 22.89% | 27/42 | 2 | 5.620 s | 15.42% | +3 | 1.76 |
| 3 s | 62.40% | 27/42 | 2 | 5.620 s | 15.42% | +3 | 3.94 |
| 10 s | 77.11% | 27/42 | 2 | 5.620 s | 15.42% | +3 | 8.30 |
| 30 s | 83.76% | 27/42 | 2 | 5.620 s | 15.42% | +3 | 18.04 |
| 60 s | 88.11% | 26/42 | 2 | 5.620 s | 15.42% | +4 | 30.60 |
| 120 s | 91.18% | 26/42 | 2 | 5.620 s | 15.42% | +4 | 50.55 |
| 300 s | 98.34% | 29/42 | 3 | 7.891 s | 21.65% | +1 | 88.11 |
| 600 s | 99.36% | 26/42 | 5 | 13.450 s | 36.91% | +4 | 112.85 |
| 1,800 s | 99.87% | 19/42 | 5 | 15.292 s | 41.97% | +11 | 132.13 |

### c64, 256 GiB

| TTL | Resume edges inside TTL | Complete external targets | Reads avoided | Gross service avoided | Oracle recovery | Net miss delta | Protected GiB-hours |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 0 s | 0.00% | 537/789 | 35 | 117.250 s | 21.70% | +40 | 0.00 |
| 1 s | 7.95% | 538/789 | 35 | 117.250 s | 21.70% | +39 | 2.25 |
| 3 s | 26.42% | 538/789 | 34 | 117.697 s | 21.78% | +39 | 6.21 |
| 10 s | 53.03% | **545/789** | 36 | 124.609 s | 23.06% | **+32** | 16.70 |
| 30 s | 79.73% | 540/789 | 41 | 141.215 s | 26.13% | +37 | 34.24 |
| 60 s | 84.28% | 520/789 | 40 | 135.244 s | 25.03% | +57 | 52.28 |
| 120 s | 88.64% | 497/789 | 48 | 157.215 s | 29.09% | +80 | 77.01 |
| 300 s | 95.74% | 416/789 | 57 | 209.655 s | 38.80% | +161 | 114.89 |
| 600 s | 98.11% | 278/789 | 51 | 174.343 s | 32.26% | +299 | 136.04 |
| 1,800 s | 99.53% | 75/789 | 28 | 95.387 s | 17.65% | +502 | 157.53 |

Figure 1 uses every predefined TTL at the native 256 GiB capacity. Provenance: A2 finite replay over the accepted c32/c64 normalized databases; no smoothing or aggregation.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 1 — Net request-miss frontier at 256 GiB",
  "width": 700,
  "height": 310,
  "data": {
    "values": [
      {"ttl": "0", "order": 0, "corpus": "c32", "net_misses": 3},
      {"ttl": "1", "order": 1, "corpus": "c32", "net_misses": 3},
      {"ttl": "3", "order": 2, "corpus": "c32", "net_misses": 3},
      {"ttl": "10", "order": 3, "corpus": "c32", "net_misses": 3},
      {"ttl": "30", "order": 4, "corpus": "c32", "net_misses": 3},
      {"ttl": "60", "order": 5, "corpus": "c32", "net_misses": 4},
      {"ttl": "120", "order": 6, "corpus": "c32", "net_misses": 4},
      {"ttl": "300", "order": 7, "corpus": "c32", "net_misses": 1},
      {"ttl": "600", "order": 8, "corpus": "c32", "net_misses": 4},
      {"ttl": "1800", "order": 9, "corpus": "c32", "net_misses": 11},
      {"ttl": "0", "order": 0, "corpus": "c64", "net_misses": 40},
      {"ttl": "1", "order": 1, "corpus": "c64", "net_misses": 39},
      {"ttl": "3", "order": 2, "corpus": "c64", "net_misses": 39},
      {"ttl": "10", "order": 3, "corpus": "c64", "net_misses": 32},
      {"ttl": "30", "order": 4, "corpus": "c64", "net_misses": 37},
      {"ttl": "60", "order": 5, "corpus": "c64", "net_misses": 57},
      {"ttl": "120", "order": 6, "corpus": "c64", "net_misses": 80},
      {"ttl": "300", "order": 7, "corpus": "c64", "net_misses": 161},
      {"ttl": "600", "order": 8, "corpus": "c64", "net_misses": 299},
      {"ttl": "1800", "order": 9, "corpus": "c64", "net_misses": 502}
    ]
  },
  "mark": {"type": "line", "point": true, "strokeWidth": 2},
  "encoding": {
    "x": {"field": "ttl", "type": "ordinal", "sort": {"field": "order"}, "title": "Soft TTL (s)"},
    "y": {"field": "net_misses", "type": "quantitative", "title": "Net additional incomplete requests (count)", "scale": {"zero": true}},
    "color": {"field": "corpus", "type": "nominal", "title": "Corpus", "scale": {"domain": ["c32", "c64"], "range": ["#4c78a8", "#f58518"]}},
    "tooltip": [
      {"field": "corpus", "type": "nominal", "title": "Corpus"},
      {"field": "ttl", "type": "ordinal", "title": "TTL (s)"},
      {"field": "net_misses", "type": "quantitative", "title": "Net miss delta"}
    ]
  }
}
~~~

Figure 1 shows the failure mode directly: c64 has a small 10–30 s basin, then false protection rises rapidly. c32 has a different and weak 300 s optimum. A single TTL selected from average continuation gap would be unsafe.

Figure 2 plots gross known service recovery against time-weighted protected capacity. Provenance is the same reference-capacity replay. Values are exact TTL cells, not interpolated.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 2 — Gross service recovery versus retention cost",
  "width": 680,
  "height": 320,
  "data": {
    "values": [
      {"corpus": "c32", "ttl": "1 s", "gib_hours": 1.76, "recovery_pct": 15.42},
      {"corpus": "c32", "ttl": "3 s", "gib_hours": 3.94, "recovery_pct": 15.42},
      {"corpus": "c32", "ttl": "10 s", "gib_hours": 8.30, "recovery_pct": 15.42},
      {"corpus": "c32", "ttl": "30 s", "gib_hours": 18.04, "recovery_pct": 15.42},
      {"corpus": "c32", "ttl": "60 s", "gib_hours": 30.60, "recovery_pct": 15.42},
      {"corpus": "c32", "ttl": "120 s", "gib_hours": 50.55, "recovery_pct": 15.42},
      {"corpus": "c32", "ttl": "300 s", "gib_hours": 88.11, "recovery_pct": 21.65},
      {"corpus": "c32", "ttl": "600 s", "gib_hours": 112.85, "recovery_pct": 36.91},
      {"corpus": "c32", "ttl": "1800 s", "gib_hours": 132.13, "recovery_pct": 41.97},
      {"corpus": "c64", "ttl": "1 s", "gib_hours": 2.25, "recovery_pct": 21.70},
      {"corpus": "c64", "ttl": "3 s", "gib_hours": 6.21, "recovery_pct": 21.78},
      {"corpus": "c64", "ttl": "10 s", "gib_hours": 16.70, "recovery_pct": 23.06},
      {"corpus": "c64", "ttl": "30 s", "gib_hours": 34.24, "recovery_pct": 26.13},
      {"corpus": "c64", "ttl": "60 s", "gib_hours": 52.28, "recovery_pct": 25.03},
      {"corpus": "c64", "ttl": "120 s", "gib_hours": 77.01, "recovery_pct": 29.09},
      {"corpus": "c64", "ttl": "300 s", "gib_hours": 114.89, "recovery_pct": 38.80},
      {"corpus": "c64", "ttl": "600 s", "gib_hours": 136.04, "recovery_pct": 32.26},
      {"corpus": "c64", "ttl": "1800 s", "gib_hours": 157.53, "recovery_pct": 17.65}
    ]
  },
  "mark": {"type": "point", "filled": true, "size": 85},
  "encoding": {
    "x": {"field": "gib_hours", "type": "quantitative", "title": "Protected capacity (GiB-hours)", "scale": {"zero": true}},
    "y": {"field": "recovery_pct", "type": "quantitative", "title": "Gross oracle-service recovery (%)", "scale": {"zero": true}},
    "color": {"field": "corpus", "type": "nominal", "title": "Corpus", "scale": {"domain": ["c32", "c64"], "range": ["#4c78a8", "#f58518"]}},
    "tooltip": [
      {"field": "corpus", "type": "nominal", "title": "Corpus"},
      {"field": "ttl", "type": "nominal", "title": "TTL"},
      {"field": "gib_hours", "type": "quantitative", "title": "Protected GiB-hours", "format": ".2f"},
      {"field": "recovery_pct", "type": "quantitative", "title": "Recovery (%)", "format": ".2f"}
    ]
  }
}
~~~

Figure 2 also demonstrates why gross service is unsafe alone. High-cost c32/c64 points can show more recorded service saved while creating many more new misses.

## Capacity-sensitivity gate

The capacity sweep tests 192, 256, and 320 GiB with 0, 10, 30, and 300 s. The table reports the change in complete external requests relative to no-TTL LRU at the same simulated capacity.

| Corpus | TTL | 192 GiB | 256 GiB | 320 GiB |
|---|---:|---:|---:|---:|
| c32 | 10 s | 0 | 0 | 0 |
| c32 | 30 s | 0 | 0 | 0 |
| c32 | 300 s | +3 | +2 | +2 |
| c64 | 10 s | -7 | +8 | +1 |
| c64 | 30 s | **+6** | **+3** | **+10** |
| c64 | 300 s | -122 | -121 | -66 |

Figure 3 shows the capacity result at request granularity. Provenance: matched offline replay at each capacity; the no-TTL result at that same capacity is subtracted.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 3 — TTL request-readiness gain over no-TTL LRU by capacity",
  "width": 700,
  "height": 320,
  "data": {
    "values": [
      {"capacity_gib": 192, "series": "c32 · 10 s", "delta_ready": 0},
      {"capacity_gib": 256, "series": "c32 · 10 s", "delta_ready": 0},
      {"capacity_gib": 320, "series": "c32 · 10 s", "delta_ready": 0},
      {"capacity_gib": 192, "series": "c32 · 30 s", "delta_ready": 0},
      {"capacity_gib": 256, "series": "c32 · 30 s", "delta_ready": 0},
      {"capacity_gib": 320, "series": "c32 · 30 s", "delta_ready": 0},
      {"capacity_gib": 192, "series": "c32 · 300 s", "delta_ready": 3},
      {"capacity_gib": 256, "series": "c32 · 300 s", "delta_ready": 2},
      {"capacity_gib": 320, "series": "c32 · 300 s", "delta_ready": 2},
      {"capacity_gib": 192, "series": "c64 · 10 s", "delta_ready": -7},
      {"capacity_gib": 256, "series": "c64 · 10 s", "delta_ready": 8},
      {"capacity_gib": 320, "series": "c64 · 10 s", "delta_ready": 1},
      {"capacity_gib": 192, "series": "c64 · 30 s", "delta_ready": 6},
      {"capacity_gib": 256, "series": "c64 · 30 s", "delta_ready": 3},
      {"capacity_gib": 320, "series": "c64 · 30 s", "delta_ready": 10},
      {"capacity_gib": 192, "series": "c64 · 300 s", "delta_ready": -122},
      {"capacity_gib": 256, "series": "c64 · 300 s", "delta_ready": -121},
      {"capacity_gib": 320, "series": "c64 · 300 s", "delta_ready": -66}
    ]
  },
  "layer": [
    {
      "mark": {"type": "line", "point": true, "strokeWidth": 2},
      "encoding": {
        "x": {"field": "capacity_gib", "type": "quantitative", "title": "Simulated CPU capacity (GiB)"},
        "y": {"field": "delta_ready", "type": "quantitative", "title": "Additional complete external requests (count)"},
        "color": {"field": "series", "type": "nominal", "title": "Corpus and TTL", "scale": {"scheme": "category10"}},
        "tooltip": [
          {"field": "series", "type": "nominal", "title": "Policy"},
          {"field": "capacity_gib", "type": "quantitative", "title": "CPU capacity (GiB)"},
          {"field": "delta_ready", "type": "quantitative", "title": "Ready-request delta"}
        ]
      }
    },
    {
      "mark": {"type": "rule", "color": "#222", "strokeDash": [5, 4]},
      "encoding": {"y": {"datum": 0}}
    }
  ]
}
~~~

The 30 s policy is the only tested setting that never loses request readiness relative to LRU across both traces and all three capacities. Its advantage is modest, but it is not a single-capacity artifact.

## False and harmful protection

At 30 s:

- c32 observes 655/782 resumes before expiry; 242/897 protected turn windows receive no observed reuse before expiry or are right-censored;
- c64 observes 842/1,056 resumes before expiry; 420/1,262 protected turn windows receive no observed reuse before expiry or are right-censored;
- average protected capacity is 10.41% at c32 and 17.49% at c64;
- no protected victim is forcibly evicted at the reference capacity;
- nevertheless, c32 creates five new misses and c64 creates 78 because preserving one set changes which unprotected LRU state survives.

“Protected eviction” is therefore not the complete regret measure. A TTL can harm competing state without ever evicting a protected block. Net request substitution remains the decisive diagnostic.

## Stop/go decision

### A2 gate: conditional pass as a baseline, not a live policy

The formal criterion asks whether at least one bounded TTL dominates ordinary eviction across a meaningful capacity range without being a knife-edge setting.

Thirty seconds satisfies the narrow form:

- c32: equal to LRU at all three capacities;
- c64: +6/+3/+10 complete requests at 192/256/320 GiB;
- nearby 10 s is also beneficial at the reference point, so the c64 effect is not one isolated timestamp.

But this is not a strong engineering go:

- reference-capacity net miss delta remains +3 at c32 and +37 at c64;
- recovery remains only 15.42% and 26.13%;
- the best per-trace TTL differs by 30×;
- terminal and low-value continuation pollution remains substantial.

Therefore 30 s becomes **BASELINE ONLY** for A3. It should not be patched into live vLLM yet.

## Conclusions

### Supported

1. A short bounded TTL carries some robust signal beyond LRU.
2. TTL must remain soft; long retention can destroy readiness even without hard pins.
3. Pressure changes the useful horizon materially.
4. Temporal survival alone is insufficient to select valuable continuations.
5. Request-level substitution, not gross service or resume coverage, must select the policy.

### Rejected

1. One long TTL derived from p95 continuation time.
2. Unconditional retention until return.
3. Selecting TTL by resume coverage.
4. Treating more gross read avoidance as net performance benefit.
5. Proceeding directly to a live TTL implementation.

## Next experiment

Run A3 request-readiness-aware allocation using:

- 30 s static TTL as the robust block-level baseline;
- no-TTL LRU;
- A1 perfect-until-next-use continuation oracle;
- matched finite next-use placement;
- whole-continuation value density;
- prefix-frontier and marginal-readiness variants;
- shared-prefix incremental capacity accounting.

A3 must beat 30 s on complete external requests and net miss delta at equal capacity. If grouping improves hit rate without request readiness, reject the added complexity.

State-conditioned TTL cannot be evaluated faithfully on this corpus because explicit tool/lifecycle state is absent. It remains gated on A4 instrumentation rather than being fabricated from generic timing.

## Reproducibility

Implementation:

- tools/costar/continuation_ttl.py
- tools/costar/run_continuation_ttl.py
- tools/costar/run_continuation_ttl_capacity.py
- tests/tools/test_costar_continuation_ttl.py

Verification:

- Ruff clean.
- Combined A1/A2/finite-retention/oracle-decomposition suite: 14 passed.
- All ten reference TTLs ran on c32 and c64.
- Capacity gate ran at 0.75x, 1.0x, and 1.25x with 0/10/30/300 s.
- Nothing was committed.

## Sources

- [[Research/ABC/Continuation Readiness/02 - A1 continuation-retention oracle|A1 continuation-retention oracle]]
- [[Research/ABC/Continuation Readiness/01 - A0 semantic trace certification|A0 semantic trace certification]]
- [[Research/ABC/Future-Value Placement/02 - Experiment 2 practical forced-admit policy benchmark|C32 practical-policy baseline]]
- [[Research/ABC/Future-Value Placement/06 - Experiment 6 C64 independent pressure validation|C64 pressure validation]]
- [c32 MLflow run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- [c64 MLflow run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow)