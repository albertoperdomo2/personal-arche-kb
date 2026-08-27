---
title: "COSTAR Continuation Readiness A3 — request-readiness allocation"
date: "2026-08-27"
type: "research-experiment"
experiment: "COSTAR Continuation Readiness — A3"
status: "conditionally-valid-negative-result"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-costar-oracle-trace-v1@sha256:0e79705305e63b50ac80454641c4ca277014ae0c47a664df5784a46eb079e17f"
vllm_version: "v0.27.0 experimental trace build plus offline readiness replay"
tensor_parallelism: 8
replicas: 1
gpu: "8x H100"
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: "default / not explicitly set"
concurrency: [32, 64]
cpu_capacity_gib_sweep: [192, 256, 320]
cpu_blocks_reference: 131072
kv_bytes_per_chunk: 2097152
offload_spec: "TieringOffloadingSpec"
secondary_tier: "filesystem on local NVMe"
secondary_tier_threads: "64 read / 64 write"
workload: "AIPerf AgentX/Weka agentic replay"
random_seed: 20260707
duration_seconds: 1800
cache_cleaning_state: "not established; unchanged from accepted corpus"
---

# COSTAR Continuation Readiness A3 — request-readiness allocation

## Executive summary

A3 asks whether allocating CPU retention at whole-request or shared-continuation granularity improves request readiness over independently ranked blocks when every policy receives the same future information.

**It does not.**

At the native 256 GiB CPU capacity:

- c32 has no allocation pressure among the eligible external continuations; independent blocks, deadline prefixes, whole continuations, and shared marginal readiness all complete 36/42 requests;
- c64 independent blocks and deadline-prefix allocation complete 509/789 requests;
- c64 whole-continuation and shared marginal-readiness allocation complete 504/789, five fewer than the independent policies;
- the online 30-second TTL baseline completes 540/789, 31–36 more than the oracle-informed allocators, because the latter retain real but distant continuations for too long.

The 192/320 GiB sweep confirms the negative result:

- c32 grouping is identical to independent allocation at every capacity;
- c64 whole/marginal grouping is four requests worse at 192 GiB, five worse at 256 GiB, and identical at 320 GiB;
- shared-prefix incremental accounting never improves complete-request outcomes.

The tested grouping hypothesis is therefore rejected. The remaining issue is **eligibility/value information and pressure-aware retention horizon**, not whether blocks are packed as whole request sets.

**Decision:** do not carry whole-continuation set packing or marginal-readiness optimization into live vLLM. Preserve 30-second TTL and independent deadline ordering as baselines. The next research gate is A4: determine what application/lifecycle signal can identify valuable continuations early enough and suppress distant/terminal pollution.

## Validity verdict

# Conditionally valid negative result

The comparison is valid for isolating the allocation objective under one equal-information oracle regime:

- every allocation policy receives the same observed positive continuation;
- every policy receives the same exact next-turn deadline;
- every policy protects only keys known in the previous turn's candidate working set;
- the per-continuation value proxy is the next external working-set size;
- mandatory admission, capacity, arrivals, and demand events are identical;
- no proactive read, bypass, reserve, lease, or hard pin is used;
- shared keys are charged once by the marginal policy;
- 192/256/320 GiB capacities are evaluated separately.

This is not an online predictor or end-to-end serving simulation. Positive continuation and exact deadline are future labels. Counterfactual demand-fill feedback and live ref-count constraints are absent. Newly introduced counterfactual read durations remain unknown.

The value function deliberately uses the same simple size/deadline information for all four policies. A different semantic value signal could change which continuations are eligible; that is A4, not evidence for retaining grouping complexity.

## Hypothesis and tested objectives

### Hypothesis

Whole-continuation or marginal-readiness allocation should outperform independent block ranking because keeping most—but not all—of a request's required KV may fail to make the request CPU-ready.

### Equal-information intent

After turn i finishes, an intent is created only when the observed next turn has external KV demand. The candidate keys are the previous turn's known reusable working set. The intent remains until that next external lookup.

All objectives rank with the same value/hold-time inputs. Only the allocation unit differs.

### Policies

- **Independent block:** rank individual candidate keys; a continuation may be partially selected at the capacity boundary.
- **Deadline prefix:** admit ordered continuation keys until the protected budget is full.
- **Whole continuation:** protect a continuation only when its complete candidate set fits.
- **Marginal readiness:** greedily select complete continuation sets by value divided by incremental unique keys, charging shared keys once.
- **30-second TTL:** A2 online-realizable block-level baseline; no oracle resume/deadline label.
- **Matched next use:** existing finite clairvoyant placement reference.

The allocation budget is a soft-priority domain, not reserved physical capacity. Unprotected LRU state remains resident until normal admissions require eviction.

## Main takeaways

- **Measured:** grouping never improves complete-request count over independent allocation in any corpus/capacity cell.
- **Measured:** c64 whole/marginal loses 4/5/0 requests versus independent at 192/256/320 GiB.
- **Measured:** c32 has zero partial-selection observations and all objectives are identical.
- **Measured:** c64 independent allocation records partial-boundary observations at 192/256 GiB, yet allowing those partial selections still produces more complete requests than rejecting them as whole-continuation allocation does.
- **Measured:** shared marginal cost changes some c64 key identities at 192 GiB but not the number of complete requests.
- **Inference:** partial-set waste is not the dominant remaining placement error in this trace.
- **Inference:** retaining distant true continuations creates more opportunity cost than grouping can repair.
- **Decision:** reject A3 grouping complexity; proceed to information-value/eligibility research.

## Reference-capacity results

| Corpus / policy | Complete external targets | Key-hit rate | Reads avoided | Gross service avoided | Oracle recovery | New misses | Net miss delta |
|---|---:|---:|---:|---:|---:|---:|---:|
| c32 recorded | 30/42 | 69.20% | 0 | 0.000 s | 0% | 0 | 0 |
| c32 30 s TTL | 27/42 | 63.55% | 2 | 5.620 s | 15.42% | 5 | +3 |
| c32 independent block | 36/42 | 83.46% | 10 | 29.304 s | 80.42% | 4 | -6 |
| c32 deadline prefix | 36/42 | 83.46% | 10 | 29.304 s | 80.42% | 4 | -6 |
| c32 whole continuation | 36/42 | 83.46% | 10 | 29.304 s | 80.42% | 4 | -6 |
| c32 marginal readiness | 36/42 | 83.46% | 10 | 29.304 s | 80.42% | 4 | -6 |
| c32 matched next use | 42/42 | 100% | 12 | 36.440 s | 100% | 0 | -12 |
| c64 recorded | 577/789 | 74.00% diagnostic | 0 | 0.000 s | 0% | 0 | 0 |
| c64 30 s TTL | 540/789 | 70.06% | 41 | 141.215 s | 26.13% | 78 | +37 |
| c64 independent block | **509/789** | 68.55% | 94 | 354.101 s | 65.53% | 162 | +68 |
| c64 deadline prefix | **509/789** | 68.55% | 94 | 354.101 s | 65.53% | 162 | +68 |
| c64 whole continuation | 504/789 | 68.53% | 94 | 349.417 s | 64.66% | 167 | +73 |
| c64 marginal readiness | 504/789 | 68.53% | 94 | 349.417 s | 64.66% | 167 | +73 |
| c64 matched next use | 716/789 | N/A | 144 | 540.369 s | 100% | N/A | N/A |

The c64 recorded key-hit percentage is shown only as a diagnostic approximation from the prior validated corpus. The decision uses exact request completeness and policy-replay counts.

Figure 1 compares the correct request-level outcome at 256 GiB. Provenance: exact A3 replay outcomes plus the recorded and matched-next-use references; no aggregation.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 1 — Complete external requests at 256 GiB",
  "width": 720,
  "height": 310,
  "data": {
    "values": [
      {"condition_policy": "c32 · recorded", "complete": 30, "corpus": "c32"},
      {"condition_policy": "c32 · TTL 30 s", "complete": 27, "corpus": "c32"},
      {"condition_policy": "c32 · independent", "complete": 36, "corpus": "c32"},
      {"condition_policy": "c32 · whole", "complete": 36, "corpus": "c32"},
      {"condition_policy": "c32 · marginal", "complete": 36, "corpus": "c32"},
      {"condition_policy": "c32 · next use", "complete": 42, "corpus": "c32"},
      {"condition_policy": "c64 · recorded", "complete": 577, "corpus": "c64"},
      {"condition_policy": "c64 · TTL 30 s", "complete": 540, "corpus": "c64"},
      {"condition_policy": "c64 · independent", "complete": 509, "corpus": "c64"},
      {"condition_policy": "c64 · whole", "complete": 504, "corpus": "c64"},
      {"condition_policy": "c64 · marginal", "complete": 504, "corpus": "c64"},
      {"condition_policy": "c64 · next use", "complete": 716, "corpus": "c64"}
    ]
  },
  "mark": {"type": "bar"},
  "encoding": {
    "x": {"field": "condition_policy", "type": "nominal", "title": "Corpus and policy", "sort": null, "axis": {"labelAngle": -28}},
    "y": {"field": "complete", "type": "quantitative", "title": "CPU-complete external requests (count)", "scale": {"zero": true}},
    "color": {"field": "corpus", "type": "nominal", "title": "Corpus", "scale": {"domain": ["c32", "c64"], "range": ["#4c78a8", "#f58518"]}},
    "tooltip": [
      {"field": "condition_policy", "type": "nominal", "title": "Condition"},
      {"field": "complete", "type": "quantitative", "title": "Complete requests"}
    ]
  }
}
~~~

Figure 1 shows that A3's oracle information is valuable at c32 but the allocation unit does not matter. At c64, the independent objective is slightly better than grouping, while the shorter TTL avoids more overall substitution.

## Capacity-sensitivity result

| Corpus | Capacity | TTL 30 s | Independent | Whole | Marginal | Whole minus independent |
|---|---:|---:|---:|---:|---:|---:|
| c32 | 192 GiB | 18 | 31 | 31 | 31 | 0 |
| c32 | 256 GiB | 27 | 36 | 36 | 36 | 0 |
| c32 | 320 GiB | 34 | 40 | 40 | 40 | 0 |
| c64 | 192 GiB | 412 | **449** | 445 | 445 | -4 |
| c64 | 256 GiB | **540** | 509 | 504 | 504 | -5 |
| c64 | 320 GiB | **624** | 571 | 571 | 571 | 0 |

Figure 2 directly plots grouping's incremental request value over independent block allocation. Provenance: matched A3 replay at each simulated CPU capacity.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 2 — Grouping gain over independent block allocation",
  "width": 680,
  "height": 300,
  "data": {
    "values": [
      {"capacity_gib": 192, "series": "c32 · whole", "delta_complete": 0},
      {"capacity_gib": 256, "series": "c32 · whole", "delta_complete": 0},
      {"capacity_gib": 320, "series": "c32 · whole", "delta_complete": 0},
      {"capacity_gib": 192, "series": "c32 · marginal", "delta_complete": 0},
      {"capacity_gib": 256, "series": "c32 · marginal", "delta_complete": 0},
      {"capacity_gib": 320, "series": "c32 · marginal", "delta_complete": 0},
      {"capacity_gib": 192, "series": "c64 · whole", "delta_complete": -4},
      {"capacity_gib": 256, "series": "c64 · whole", "delta_complete": -5},
      {"capacity_gib": 320, "series": "c64 · whole", "delta_complete": 0},
      {"capacity_gib": 192, "series": "c64 · marginal", "delta_complete": -4},
      {"capacity_gib": 256, "series": "c64 · marginal", "delta_complete": -5},
      {"capacity_gib": 320, "series": "c64 · marginal", "delta_complete": 0}
    ]
  },
  "layer": [
    {
      "mark": {"type": "line", "point": true, "strokeWidth": 2},
      "encoding": {
        "x": {"field": "capacity_gib", "type": "quantitative", "title": "Simulated CPU capacity (GiB)"},
        "y": {"field": "delta_complete", "type": "quantitative", "title": "Additional complete requests vs independent (count)"},
        "color": {"field": "series", "type": "nominal", "title": "Corpus and grouped policy", "scale": {"scheme": "category10"}},
        "tooltip": [
          {"field": "series", "type": "nominal", "title": "Policy"},
          {"field": "capacity_gib", "type": "quantitative", "title": "CPU capacity (GiB)"},
          {"field": "delta_complete", "type": "quantitative", "title": "Complete-request delta"}
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

There is no capacity at which grouping is positive. The tightest c64 condition, where set packing should matter most, is the condition in which grouping loses four requests.

Figure 3 compares the online 30-second horizon with the longer positive-continuation oracle. Provenance: exact net miss counts at each capacity; negative values are improvements versus the recorded reference.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 3 — c64 pressure crossover: short TTL versus long-lived oracle intent",
  "width": 680,
  "height": 300,
  "data": {
    "values": [
      {"capacity_gib": 192, "policy": "TTL 30 s", "net_misses": 165},
      {"capacity_gib": 256, "policy": "TTL 30 s", "net_misses": 37},
      {"capacity_gib": 320, "policy": "TTL 30 s", "net_misses": -47},
      {"capacity_gib": 192, "policy": "Independent deadline", "net_misses": 128},
      {"capacity_gib": 256, "policy": "Independent deadline", "net_misses": 68},
      {"capacity_gib": 320, "policy": "Independent deadline", "net_misses": 6},
      {"capacity_gib": 192, "policy": "Whole continuation", "net_misses": 132},
      {"capacity_gib": 256, "policy": "Whole continuation", "net_misses": 73},
      {"capacity_gib": 320, "policy": "Whole continuation", "net_misses": 6}
    ]
  },
  "layer": [
    {
      "mark": {"type": "line", "point": true, "strokeWidth": 2},
      "encoding": {
        "x": {"field": "capacity_gib", "type": "quantitative", "title": "Simulated CPU capacity (GiB)"},
        "y": {"field": "net_misses", "type": "quantitative", "title": "Net additional incomplete requests (count)"},
        "color": {"field": "policy", "type": "nominal", "title": "Policy", "scale": {"scheme": "category10"}},
        "tooltip": [
          {"field": "policy", "type": "nominal", "title": "Policy"},
          {"field": "capacity_gib", "type": "quantitative", "title": "CPU capacity (GiB)"},
          {"field": "net_misses", "type": "quantitative", "title": "Net miss delta"}
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

At 192 GiB, exact positive/deadline information helps more than TTL. At 256/320 GiB, retaining real but distant continuations creates more opportunity cost than the short TTL. The missing mechanism is a pressure-aware eligibility threshold or better future-value signal, not whole-request packing.

## Why grouping did not help

### The safe horizon does not bind the cache

A2's 30-second TTL protects at most 76,305 resident c64 keys at the reference capacity—well below 131,072. Within the safe horizon, there is no large set-packing conflict to solve.

### Longer-lived positive intent is still too broad

A continuation can truly return and still be a poor retention choice when its deadline is far away and nearer unknown/native traffic needs the space. The A3 allocators rank deadlines but fill the available protection budget without a calibrated opportunity-cost threshold.

### Partial allocations were not the dominant regret

At c64 192/256 GiB, independent selection crosses partial boundaries, but it still completes more requests than whole-set rejection. Keeping partial state sometimes preserves useful LRU structure or multiple future candidates; refusing a large set does not automatically convert the freed space into another complete valuable request.

### Shared-prefix savings were too small to change decisions

Marginal unique-key accounting changes some key-level outcomes at 192 GiB but produces exactly the same 445 complete requests as ordinary whole-continuation selection. At 256/320 GiB it is outcome-identical.

## Stop/go decision

The predefined A3 criterion is:

> Continue readiness-aware structure only if it improves request-level service materially over the best block-level policy.

It does not.

- whole/marginal never beats independent;
- it is worse under the two c64 pressure points where grouping binds;
- shared accounting adds no complete requests;
- exact marginal recomputation is substantially more expensive than the simple objectives.

### Verdict: KILL grouping as a primary mechanism

Do not implement:

- whole-continuation set packing on the live eviction path;
- exact shared-prefix marginal optimization;
- request-level knapsack recomputation;
- prefix-frontier allocation as a separate live subsystem.

Keep request readiness as an **evaluation metric**, because it correctly exposes harmful substitution. Do not keep it as a complex allocator when the data shows no allocation benefit.

## What remains promising

A1 is not invalidated. Continuation information still recovers substantial gross oracle value. A2/A3 refine the diagnosis:

1. exact continuation candidate discovery is easy;
2. naïve long retention is harmful;
3. short TTL has a small robust benefit;
4. grouping does not repair long-horizon pollution;
5. the unresolved variable is which continuation deserves retention under current pressure.

That is an information-value and eligibility problem.

## Next experiment

Proceed to A4 semantic information-value ladder.

The current traces can evaluate only:

- I0: ordinary key/history signals;
- I1: stable continuation identity plus elapsed time/deadline.

Those regimes are already largely represented by the failed history policies, A2 TTL, and A3 deadline oracle. The discriminating A4 regimes require a fresh semantically enriched trace with:

- lifecycle state: tool pending, user wait, active, finished;
- tool or agent class;
- workflow node and candidate successors;
- explicit terminal/session-close event;
- early tool/workflow timestamps;
- stable session/root/continuation identity.

Before another live policy, instrument and capture those fields on matched c32/c64 AgentX runs. Then measure incremental oracle recovery and decision regret per information regime. If lifecycle/tool state does not materially reduce false protection over 30-second TTL, stop local semantic retention and move to A6 route-to-data or secondary data-plane work.

## Reproducibility

Implementation:

- tools/costar/readiness_allocation.py
- tools/costar/run_readiness_allocation.py
- tools/costar/run_readiness_allocation_capacity.py
- tests/tools/test_costar_readiness_allocation.py

Verification:

- Ruff clean.
- Combined A1/A2/A3/finite-retention/oracle-decomposition suite: 16 passed.
- Native 256 GiB replay completed on c32/c64.
- Independent 192/320 GiB capacity replay completed on c32/c64.
- Nothing was committed.

## Sources

- [[Research/ABC/Continuation Readiness/03 - A2 bounded soft-TTL frontier|A2 bounded soft-TTL frontier]]
- [[Research/ABC/Continuation Readiness/02 - A1 continuation-retention oracle|A1 continuation-retention oracle]]
- [[Research/ABC/Continuation Readiness/01 - A0 semantic trace certification|A0 semantic trace certification]]
- [[Research/ABC/Future-Value Placement/06 - Experiment 6 C64 independent pressure validation|C64 pressure validation]]
- [c32 MLflow run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- [c64 MLflow run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow)