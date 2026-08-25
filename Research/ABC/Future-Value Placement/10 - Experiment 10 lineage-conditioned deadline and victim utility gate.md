---
title: "Experiment 10 — Lineage-conditioned deadline and victim utility gate"
date: "2026-08-25"
type: "experiment-report"
experiment: "COSTAR-KV future-value-aware admission, retention, and eviction"
status: "complete"
result: "valid-negative-result"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
source_image: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-costar-oracle-trace-v1"
workload: "AIPerf AgentX/Weka"
configuration:
  C32:
    concurrency: 32
    requests: 901
  C64:
    concurrency: 64
    requests: 1280
cpu_blocks: 131072
kv_chunk_bytes: 2097152
secondary_tier: "local filesystem/NVMe"
random_seed: 20260707
execution: "offline frozen bidirectional replay"
---

# Experiment 10 — Lineage-conditioned deadline and victim utility gate

## Executive summary

This experiment asks whether the stable AgentX lineage recovered in [[09 - Experiment 9 AgentX identity coverage and held-out lineage information|Experiment 9]] can make the actual full-cache decision that matters: when a new ordinary-mirror block arrives, should COSTAR admit it and evict the LRU resident, or reject it and keep the victim?

The policy is trained on C32 and frozen on C64, then trained on C64 and frozen on C32. It predicts reuse within 30-, 120-, and 600-second horizons, accounting for the candidate's zero age, the victim's current age, request/prefix context, and coarse lineage. The score is horizon-discounted next-use utility, not hit count or AUC.

## Validity verdict

# Valid offline negative result

The exact request identity join is complete, the accepted normalized traces reproduce native movement, and the same decision evaluator is used for all policies. The result is sufficient to reject the tested coarse lineage-conditioned empirical hazard as the next live policy.

Limitations: there are only two executions of the same source corpus; lineage bins and smoothing are fixed heuristics; and pairwise utility is not a full cache replay. These limitations could hide a different and substantially better signal, but they cannot turn the tested policy into a passing result.

## Main takeaways

- The strict gate fails in five of six direction/horizon cells.
- C32→C64 always-admit is stronger than every lineage model at all three horizons. The best lineage result loses 10.3, 24.4, and 12.3 percentage points of oracle utility.
- C64→C32 strongly favors keeping the LRU victim at 30 and 120 seconds. The best lineage model recovers only 2.44% and 35.78% versus 99.47% and 70.20%.
- At C64→C32 and 600 seconds only, request+prefix+lineage reaches 64.21%, beating always-admit's 60.91% by 3.30 points.
- That isolated long-horizon win is not robust enough for cache replay or live implementation.
- Pairwise accuracy is misleading here: at C64→C32/30 seconds the candidate is preferred in 54.91% of pairs, yet always admitting recovers only 0.58% of utility because the less frequent victim wins are far more urgent.
- The weak aggregate lift in Experiment 9 does not survive the deadline- and substitution-aware test.

## Method

Ordinary mirrors begin value generations. Reactive demand promotions do not erase credit from the earlier ordinary mirror; a later ordinary mirror censors it. At each full-cache ordinary arrival, the evaluator compares the absent candidate with the current LRU victim.

For next-use delay $d$ and horizon $H$, utility is:

$$
U(d,H)=e^{-d/H}
$$

A policy's recovery is selected utility divided by clairvoyant next-use utility across all informative pairs.

Lineage is deliberately structural and online-available:

- turn band: 0–7, 8–23, 24–47, or 48+;
- source kind: main, subagent, flat, or unknown;
- agent depth: 0, 1, or 2+;
- prior requests in the conversation: 0, 1–3, 4–15, or 16+.

The lineage tables use 64-interval shrinkage toward the age-conditioned aggregate hazard. Raw conversation or trace IDs are not used.

## Headline results

The “best simple” policy is selected between always-admit and keep-LRU independently for each held-out cell. “Best lineage” is the most favorable of lineage-only, request+lineage, and request+prefix+lineage; this post-hoc choice is intentionally favorable to lineage.

| Train → test | Horizon | Informative pairs | Best simple | Simple utility | Best lineage | Lineage utility | Absolute delta |
|---|---:|---:|---|---:|---|---:|---:|
| C32 → C64 | 30 s | 499,673 | Always admit | 68.52% | Lineage | 58.22% | **−10.30 pp** |
| C32 → C64 | 120 s | 499,673 | Always admit | 78.73% | All lineage variants tie | 54.30% | **−24.42 pp** |
| C32 → C64 | 600 s | 499,673 | Always admit | 84.44% | Lineage | 72.12% | **−12.33 pp** |
| C64 → C32 | 30 s | 91,550 | Keep LRU | 99.47% | Request+prefix+lineage | 2.44% | **−97.04 pp** |
| C64 → C32 | 120 s | 91,550 | Keep LRU | 70.20% | Request+prefix+lineage | 35.78% | **−34.41 pp** |
| C64 → C32 | 600 s | 91,550 | Always admit | 60.91% | Request+prefix+lineage | 64.21% | **+3.30 pp** |

Figure 1 compares the strongest simple action, the strongest lineage result, and the previous request+prefix hazard in each held-out cell. Values come from native-event-resolution offline replay; no temporal samples are aggregated.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 1 — Horizon-discounted next-use utility recovery",
  "width": 760,
  "height": 330,
  "data": {
    "values": [
      {"cell": "C32→C64 · 30 s", "policy": "Best simple", "utility_pct": 68.52},
      {"cell": "C32→C64 · 30 s", "policy": "Best lineage", "utility_pct": 58.22},
      {"cell": "C32→C64 · 30 s", "policy": "Request+prefix", "utility_pct": 54.30},
      {"cell": "C32→C64 · 120 s", "policy": "Best simple", "utility_pct": 78.73},
      {"cell": "C32→C64 · 120 s", "policy": "Best lineage", "utility_pct": 54.30},
      {"cell": "C32→C64 · 120 s", "policy": "Request+prefix", "utility_pct": 54.30},
      {"cell": "C32→C64 · 600 s", "policy": "Best simple", "utility_pct": 84.44},
      {"cell": "C32→C64 · 600 s", "policy": "Best lineage", "utility_pct": 72.12},
      {"cell": "C32→C64 · 600 s", "policy": "Request+prefix", "utility_pct": 65.96},
      {"cell": "C64→C32 · 30 s", "policy": "Best simple", "utility_pct": 99.47},
      {"cell": "C64→C32 · 30 s", "policy": "Best lineage", "utility_pct": 2.44},
      {"cell": "C64→C32 · 30 s", "policy": "Request+prefix", "utility_pct": 0.58},
      {"cell": "C64→C32 · 120 s", "policy": "Best simple", "utility_pct": 70.20},
      {"cell": "C64→C32 · 120 s", "policy": "Best lineage", "utility_pct": 35.78},
      {"cell": "C64→C32 · 120 s", "policy": "Request+prefix", "utility_pct": 34.05},
      {"cell": "C64→C32 · 600 s", "policy": "Best simple", "utility_pct": 60.91},
      {"cell": "C64→C32 · 600 s", "policy": "Best lineage", "utility_pct": 64.21},
      {"cell": "C64→C32 · 600 s", "policy": "Request+prefix", "utility_pct": 60.91}
    ]
  },
  "mark": {"type": "point", "filled": true, "size": 100},
  "encoding": {
    "x": {
      "field": "cell",
      "type": "nominal",
      "sort": null,
      "title": "Held-out direction and decision horizon",
      "axis": {"labelAngle": -25}
    },
    "y": {
      "field": "utility_pct",
      "type": "quantitative",
      "title": "Oracle utility recovered (%)",
      "scale": {"zero": true, "domain": [0, 100]}
    },
    "color": {
      "field": "policy",
      "type": "nominal",
      "title": "Policy",
      "scale": {"scheme": "category10"}
    },
    "shape": {"field": "policy", "type": "nominal", "title": "Policy"},
    "tooltip": [
      {"field": "cell", "type": "nominal"},
      {"field": "policy", "type": "nominal"},
      {"field": "utility_pct", "type": "quantitative", "title": "Utility recovered (%)", "format": ".2f"}
    ]
  }
}
~~~

Figure 1 makes the failure mode visible: the preferred global action flips with pressure and horizon, while coarse lineage does not recover the urgent victim uses in C32.

## Why Experiment 9 looked more encouraging

Experiment 9 asked whether bundles could be ranked by eventual aggregate future value under a capacity budget. It did not ask whether a candidate was more urgent than the particular resident it would displace.

This experiment adds the missing opportunity cost and deadline. A model can assign somewhat better average scores yet still make harmful substitutions. The C64→C32/30-second result is the clearest example: event-count accuracy and eventual value do not capture the enormous utility of a small set of imminent victim reuses.

Measured result → interpretation:

- **Measured:** combined aggregate ranking had positive lift in Experiment 9.
- **Interpretation then:** lineage carried a weak interaction signal worth one stricter gate.
- **Measured now:** the lineage hazard loses to the stronger simple action in five of six cells.
- **Current interpretation:** that signal is not decision-sufficient for online admission/eviction.

## Decision

**Stop this policy branch.**

Do not implement:

- the tested coarse lineage-conditioned hazard;
- raw conversation-ID temperature;
- another static classifier using only the currently exported request and lineage fields;
- a live cache replay based on Experiment 9's aggregate AUC.

This does not falsify future-value placement or activity-aware routing. It falsifies the claim that the present structural fields, converted into a coarse empirical reuse hazard, robustly solve intra-replica candidate-versus-victim placement.

## Next experiment

The next discriminating experiment should remain offline and move to the multi-replica opportunity the current traces can actually test:

1. replay the exact request sequence over 2, 4, and 8 finite CPU caches;
2. compare random/round-robin placement with deterministic conversation affinity;
3. add a key-overlap routing oracle as the upper bound;
4. model the shared CephFS tier as the miss source and count cross-replica avoidable reads/bytes and complete external working sets;
5. preserve a load-balance constraint so cache affinity cannot “win” by sending everything to one replica.

This tests route-to-data/session placement, where stable identity has a deterministic function, rather than asking identity to predict which resident block has the nearest future use. It requires no new GPU run or vLLM image.

If even conversation affinity and the routing oracle have negligible upside, shift to explicit application cache intent/tool-suspension events or abandon speculative identity-based placement for this workload.

## Reproduction

Implementation:

- tools/costar/hazard_ranking.py
- tools/costar/identity_information_audit.py
- tools/costar/run_lineage_hazard_ranking.py
- tests/tools/test_costar_hazard_ranking.py

Validation: 52 COSTAR tool tests passed; Ruff and git diff checks passed. Nothing was committed.