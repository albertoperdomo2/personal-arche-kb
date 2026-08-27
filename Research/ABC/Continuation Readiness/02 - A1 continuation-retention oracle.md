---
title: "COSTAR Continuation Readiness A1 — continuation-retention oracle"
date: "2026-08-27"
type: "research-experiment"
experiment: "COSTAR Continuation Readiness — A1"
status: "conditionally-valid-mixed-positive"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-costar-oracle-trace-v1@sha256:0e79705305e63b50ac80454641c4ca277014ae0c47a664df5784a46eb079e17f"
vllm_version: "v0.27.0 experimental trace build plus offline continuation replay"
tensor_parallelism: 8
replicas: 1
gpu: "8x H100"
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: "default / not explicitly set"
concurrency: [32, 64]
cpu_bytes: 274877906944
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
offload_spec: "TieringOffloadingSpec"
secondary_tier: "filesystem on local NVMe"
secondary_tier_threads: "64 read / 64 write"
workload: "AIPerf AgentX/Weka agentic replay"
random_seed: 20260707
duration_seconds: 1800
cache_cleaning_state: "not established; unchanged from accepted corpus"
---

# COSTAR Continuation Readiness A1 — continuation-retention oracle

## Executive conclusion

A1 asked whether continuation identity explains a meaningful part of the finite-capacity CPU-placement opportunity.

**Yes—but it does not justify naïvely retaining every live continuation.**

Using only the previous turn's already-known reusable KV working set and oracle knowledge that the same concrete continuation will resume:

- the equal-priority continuation oracle recovers **80.42%** of the c32 and **61.29%** of the c64 matched next-use oracle's avoidable secondary-service seconds;
- adding the exact next-turn deadline leaves c32 unchanged and raises c64 recovery to **65.44%**;
- the prior working set covers 150,328/150,328 c32 and 3,021,790/3,021,791 c64 next-turn external key references.

This clears the research program's **strong-go information gate** of at least 50% recovery on both traces. Continuation identity is therefore a high-value signal, and the project should continue to A2/A3.

However, the gross recovery metric is not a deployment verdict. At c64, deadline-ordered whole-continuation retention avoids 95 recorded reads but makes 159 previously complete external requests incomplete: **64 net additional misses**. It completes 513/789 external targets, below recorded placement's 577/789. The policy protects an average 48.13% of CPU capacity and reaches 100% peak protected occupancy.

The correct decision is:

> **Keep continuation-aware retention as the research direction; reject unconditional whole-continuation protection as the live policy. Run the bounded TTL frontier and readiness-aware allocation next.**

## Hypothesis and mechanism

### Hypothesis

A large fraction of the finite next-use placement gap can be recovered by knowing that a logical continuation will resume, without knowing exact per-block next-use order.

### What the replay knows

At the end of turn i, the replay associates that turn's finish working set and its ordinary mirrored arrivals with its exact "x_correlation_id".

It does **not** inspect turn i+1's block list to construct the protected set. The future trace is used only to supply two oracle labels:

1. **resume-binary:** whether this continuation is observed to resume;
2. **next-deadline:** the exact time at which its next request reaches the external-demand boundary.

Final observed turns are right-censored and are not treated as known-dead sessions.

### What the replay does

Both policies use:

- the recorded 131,072-block CPU capacity;
- the common recorded CPU-ready arrival stream;
- mandatory admission;
- a soft victim priority over LRU;
- shared-key ownership charged once;
- no proactive read;
- no admission bypass;
- no reserved pool;
- no lease or hard pin.

When an unprotected victim exists, it is evicted before protected state. If all resident state is protected, protected state remains evictable. The deadline variant evicts the protected block whose continuation is needed furthest in the future.

This is an offline placement replay, not a live vLLM implementation.

## Validity verdict

### Conditionally valid for the A1 information-value decision

The semantic identity and timing inputs were certified by [[Research/ABC/Continuation Readiness/01 - A0 semantic trace certification|A0]]. Every exported client request joins exactly to a vLLM request; continuation edges are ordered under "x_correlation_id".

The policy comparison preserves capacity and recorded CPU-ready arrivals. It therefore answers whether the continuation signal can change victim selection usefully under the accepted trace model.

### Important limitations

- "resume_binary" is a positive-only oracle: it knows which observed continuations return. A live system must estimate this or receive lifecycle intent.
- "next_deadline" additionally knows exact future timing.
- The replay has exogenous recorded arrivals and does not feed counterfactual demand-fill completions back into later occupancy.
- Live transfer/ref-count constraints are not simulated.
- Gross measured service saved is known for eliminated recorded reads. The service cost of newly created counterfactual misses is unknown, so gross seconds cannot be called net latency savings.
- c32 and c64 are independent pressure conditions from the same workload family and seed, not two unrelated production distributions.
- A1 tests whole-continuation protection. It does not yet optimize TTL, prefix frontier, or marginal request readiness.

## Results

### Signal coverage

| Metric | c32 | c64 |
|---|---:|---:|
| Exact client/server joins | 897/897 | 1,262/1,262 |
| Usable observed continuation edges | 782 | 1,056 |
| Right-censored final turns | 115 | 206 |
| Next-turn edges with external demand | 40 | 767 |
| Candidate key recall | 150,328/150,328 (100%) | 3,021,790/3,021,791 (99.99997%) |
| Fully covered external next turns | 40/40 (100%) | 766/767 (99.87%) |

The candidate-set result is decisive: for this workload, the previous turn's known reusable working set almost exactly identifies the next continuation's external KV requirement. The unsolved problem is not block discovery. It is deciding how much of that state deserves scarce CPU residency, for how long, relative to competing requests.

### Placement and request outcomes

| Corpus / policy | Complete external targets | Recorded reads avoided | Gross measured service avoided | Oracle service recovery | New misses | Net miss delta |
|---|---:|---:|---:|---:|---:|---:|
| c32 recorded | 30/42 | 0/12 | 0.000 s | 0% | 0 | 0 |
| c32 always-admit LRU | 27/42 | 2/12 | 5.620 s | 15.42% | 5 | +3 |
| c32 continuation, resume only | 36/42 | 10/12 | 29.304 s | **80.42%** | 4 | **-6** |
| c32 continuation + deadline | 36/42 | 10/12 | 29.304 s | **80.42%** | 4 | **-6** |
| c32 matched next-use | 42/42 | 12/12 | 36.440 s | 100% | 0 | -12 |
| c64 recorded | 577/789 | 0/212 | 0.000 s | 0% | 0 | 0 |
| c64 always-admit LRU | 537/789 | 35/212 | 117.250 s | 21.70% | 75 | +40 |
| c64 continuation, resume only | 498/789 | 90/212 | 331.171 s | **61.29%** | 169 | **+79** |
| c64 continuation + deadline | 513/789 | 95/212 | 353.615 s | **65.44%** | 159 | **+64** |
| c64 matched next-use | 716/789 | 144/212 | 540.369 s | 100% | not assigned | not assigned |

A negative net miss delta is beneficial. A positive value means more previously ready requests are harmed than recorded reads are recovered.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 1 — Fraction of matched next-use service opportunity recovered",
  "width": 700,
  "height": 300,
  "data": {
    "values": [
      {"condition_policy": "c32 · LRU", "recovery_pct": 15.42, "condition": "c32"},
      {"condition_policy": "c32 · resume only", "recovery_pct": 80.42, "condition": "c32"},
      {"condition_policy": "c32 · exact deadline", "recovery_pct": 80.42, "condition": "c32"},
      {"condition_policy": "c64 · LRU", "recovery_pct": 21.70, "condition": "c64"},
      {"condition_policy": "c64 · resume only", "recovery_pct": 61.29, "condition": "c64"},
      {"condition_policy": "c64 · exact deadline", "recovery_pct": 65.44, "condition": "c64"}
    ]
  },
  "layer": [
    {
      "mark": {"type": "bar"},
      "encoding": {
        "x": {"field": "condition_policy", "type": "nominal", "title": "Corpus and policy", "sort": null, "axis": {"labelAngle": -22}},
        "y": {"field": "recovery_pct", "type": "quantitative", "title": "Gross oracle service recovery (%)", "scale": {"domain": [0, 100], "zero": true}},
        "color": {"field": "condition", "type": "nominal", "title": "Corpus", "scale": {"domain": ["c32", "c64"], "range": ["#4c78a8", "#f58518"]}},
        "tooltip": [
          {"field": "condition_policy", "type": "nominal", "title": "Condition"},
          {"field": "recovery_pct", "type": "quantitative", "title": "Recovery (%)", "format": ".2f"}
        ]
      }
    },
    {
      "mark": {"type": "rule", "color": "#b91c1c", "strokeDash": [6, 4]},
      "encoding": {"y": {"datum": 50}}
    }
  ]
}
~~~

The continuation signal clears the predefined 50% strong-go threshold on both corpora. Exact deadline ordering adds 4.15 percentage points at c64, showing that time-to-next-use matters once protected demand competes for the full cache.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 2 — Request substitution versus recorded placement",
  "width": 700,
  "height": 300,
  "data": {
    "values": [
      {"condition_policy": "c32 · LRU", "net_misses": 3, "condition": "c32"},
      {"condition_policy": "c32 · resume only", "net_misses": -6, "condition": "c32"},
      {"condition_policy": "c32 · exact deadline", "net_misses": -6, "condition": "c32"},
      {"condition_policy": "c64 · LRU", "net_misses": 40, "condition": "c64"},
      {"condition_policy": "c64 · resume only", "net_misses": 79, "condition": "c64"},
      {"condition_policy": "c64 · exact deadline", "net_misses": 64, "condition": "c64"}
    ]
  },
  "layer": [
    {
      "mark": {"type": "bar"},
      "encoding": {
        "x": {"field": "condition_policy", "type": "nominal", "title": "Corpus and policy", "sort": null, "axis": {"labelAngle": -22}},
        "y": {"field": "net_misses", "type": "quantitative", "title": "Net additional incomplete external requests"},
        "color": {"field": "condition", "type": "nominal", "title": "Corpus", "scale": {"domain": ["c32", "c64"], "range": ["#4c78a8", "#f58518"]}},
        "tooltip": [
          {"field": "condition_policy", "type": "nominal", "title": "Condition"},
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

Figure 2 prevents the wrong conclusion from Figure 1. The c64 oracle finds many expensive recorded misses, but indiscriminate protection displaces even more previously ready requests. Gross saved seconds alone would hide this substitution because the new counterfactual reads have no measured service duration.

### Retention pressure

| Corpus / policy | Peak protected blocks | Average protected share of CPU | Approx. average protected GiB | Protected evictions |
|---|---:|---:|---:|---:|
| c32 resume/deadline | 119,601 (91.25%) | 31.44% | 80.48 GiB | 0 |
| c64 resume only | 131,072 (100%) | 49.59% | 126.94 GiB | 16,792 |
| c64 exact deadline | 131,072 (100%) | 48.13% | 123.22 GiB | 16,792 |

The average capacity figures are time-weighted protected-resident occupancy. The conversion uses the measured 2 MiB KV chunk size.

c32 never needs to evict protected state and obtains positive net benefit. c64 fills the entire policy domain with protected continuations, forcing protected-vs-protected choices. Deadline ordering improves outcomes but does not eliminate regret. This is direct evidence that “keep every continuation that will return” is too coarse under realistic pressure.

## What A1 proves—and what it does not

### Supported

1. **Continuation is the right unit of candidate discovery.** The current turn already contains essentially all external KV needed by its next continuation.
2. **Continuation information explains a large fraction of the placement oracle.** Gross recovery is above 60% on both pressure conditions and far above ordinary LRU.
3. **Retention is a better first control point than admission-time prefetch for this signal.** No data movement is needed to obtain the measured opportunity.
4. **Deadline information has incremental value under pressure.** It improves c64 readiness, read avoidance, and gross service recovery.
5. **Shared ownership is implementable without double-charging capacity.** One resident key can support multiple continuation owners.

### Rejected

1. **Unconditional whole-continuation protection is not a live candidate.** It regresses request completeness at c64.
2. **Gross service recovery is not sufficient as a policy objective.** It can look excellent while request substitution is harmful.
3. **A binary alive/dead priority is not enough at high pressure.** The cache can become entirely protected.
4. **Exact deadline alone solves the allocation problem.** It reduces but does not remove c64 regret.

### Not established

- live TTFT or throughput benefit;
- optimal TTL;
- the best request/prefix frontier;
- cost-weighted victim regret for newly created misses;
- online ability to predict continuation/resume;
- benefit across unrelated workloads or models.

## Decision

### Research gate: strong go

The predefined A1 criterion is satisfied:

- c32 continuation-only recovery: 80.42%;
- c64 continuation-only recovery: 61.29%.

The missing information in earlier history-only policies was substantially continuation intent.

### Engineering gate: no-go for whole-continuation live implementation

A live implementation of “retain every returning continuation until its next turn” would be unjustified. At c64 it causes 64 net additional incomplete external requests even with exact future resume/deadline knowledge.

The next task is not to patch this policy into vLLM. It is to reduce false protection and allocate capacity at the correct request-level objective.

## Next experiments

### A2 — bounded soft TTL frontier

Run static and pressure-aware TTLs over the same c32/c64 traces. Every TTL remains a soft score, never a pin. Measure:

- net external miss delta;
- gross known service avoided;
- protected GiB-hours;
- peak and average protected capacity;
- protected evictions;
- service saved per protected GiB-hour.

The goal is to find whether most continuation value can be retained without saturating the protected domain at c64.

### A3 — request-readiness-aware allocation

Then compare whole-continuation, prefix-frontier, and marginal-readiness allocation at equal capacity. The primary goal is to stop spending capacity on partial working sets that do not complete a request.

A3 is necessary even if A2 finds a good TTL, because c64 shows direct protected-vs-protected competition.

### Live work remains gated

Proceed to a live shadow policy only if A2/A3 produce:

- non-positive net miss delta on both traces;
- substantial oracle recovery;
- bounded protected occupancy;
- low decision overhead;
- an explicit mapping from policy decisions to avoided/added reads.

## Reproducibility

Implementation:

- tools/costar/continuation_retention.py
- tools/costar/run_continuation_retention.py
- tools/costar/finite_retention.py (adds the reversible key-ID mapping)
- tests/tools/test_costar_continuation_retention.py

Command:

~~~bash
PYTHONPATH=. /Users/aperdomo/workspace/redhat/vllm/.venv/bin/python \
  -m tools.costar.run_continuation_retention \
  /private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite \
  /private/tmp/abc-oracle-validation-20260825/c32/profile_export.jsonl \
  /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite \
  /private/tmp/abc-oracle-validation-20260825/c64/profile_export.jsonl
~~~

Verification:

- Ruff: clean on the touched A1 files.
- Focused A1 tests: 3 passed.
- Combined finite-retention/oracle-decomposition/A1 suite: 12 passed.
- Nothing was committed.

## Sources

- [[Research/ABC/Continuation Readiness/01 - A0 semantic trace certification|A0 semantic trace certification]]
- [[Research/ABC/Future-Value Placement/01 - Experiment 1 matched next-use admission decomposition|Matched next-use decomposition]]
- [[Research/ABC/Future-Value Placement/02 - Experiment 2 practical forced-admit policy benchmark|C32 practical policy benchmark]]
- [[Research/ABC/Future-Value Placement/06 - Experiment 6 C64 independent pressure validation|C64 pressure validation]]
- [c32 MLflow run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- [c64 MLflow run](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow)