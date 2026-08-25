---
title: "Experiment 9 — AgentX identity coverage and held-out lineage information"
date: "2026-08-25"
type: "experiment-report"
experiment: "COSTAR-KV future-value-aware admission, retention, and eviction"
status: "complete"
result: "positive-but-weak-information-signal"
---

# Experiment 9 — AgentX identity coverage and held-out lineage information

## Goal

Determine, entirely offline, whether the existing AgentX/Weka artifacts contain stable activity identity and whether that information improves future-value ranking beyond the static bundle features that failed in Experiments 7 and 8.

This is an information test, not a cache-policy replay and not evidence of end-to-end performance.

## Inputs and exact join

- C32 MLflow run: [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- C64 MLflow run: [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow)
- AIPerf artifact in each run: benchmark/profile_export.jsonl
- Normalized oracle databases from the accepted C32 and C64 corpora

AIPerf exports x_request_id. vLLM embeds that UUID verbatim in its server request ID, for example chatcmpl-<x_request_id>-<suffix>. The join is therefore exact; it does not use timestamps, token counts, prompt contents, or nearest-neighbor matching.

| Corpus | AIPerf records | Exact joins | Ambiguous | Bundle examples joined | Conversations | Records in repeated conversations |
|---|---:|---:|---:|---:|---:|---:|
| C32 | 897 | 897 (100%) | 0 | 888 / 891 (99.66%) | 115 | 853 |
| C64 | 1,262 | 1,262 (100%) | 0 | 1,241 / 1,258 (98.65%) | 177 | 1,190 |

The few unmatched bundle examples are non-AIPerf server-side requests; they are excluded from the ranking comparison.

## Available information

The existing client artifact already contains:

- conversation_id and source_trace_id;
- turn_index, source_outer_idx, and source_inner_idx;
- source kind: main, subagent, or flat;
- agent depth;
- parent and root correlation identity;
- request start time, allowing online prior-turn count and prior-gap features.

It does not contain explicit tool name/state, workflow DAG node, application class, route or replica transition, session suspension state, or destination-node identity. These fields also are not directly present in the oracle trace. The present offline join works only because x_request_id survives inside the vLLM request ID.

## Method

For every completed ordinary-mirror request bundle, the experiment reused Experiment 7's future-use labels and compared frozen models in both directions:

- bundle context: prompt, output, source reuse, bundle size/duplication/duration, readiness delay, and CPU pressure;
- lineage structure: turn/outer/inner positions, source kind, agent depth, parent presence, prior conversation count, and prior conversation gap;
- combined: bundle plus lineage structure;
- same-corpus trace identity bound: a smoothed source_trace_id lookup trained on the other pressure run. This is a memorization upper bound because both runs replay the same corpus, not evidence of cross-workload generalization.

Capacity-weighted AUC is the main rank metric. Top-25%-byte value lift uses tie-aware expected capture, so a constant score is exactly 1.0× rather than inheriting arbitrary request order.

## Results

### Reuse-probability ranking

| Train → test | Model | Byte-weighted AUC | Byte-weighted AP | Top-25%-byte future-value lift |
|---|---|---:|---:|---:|
| C32 → C64 | Bundle context | 0.394 | 0.684 | 0.555× |
| C32 → C64 | Lineage structure | 0.430 | 0.711 | 0.735× |
| C32 → C64 | Combined | **0.548** | **0.828** | **1.049×** |
| C32 → C64 | Same-corpus trace identity bound | 0.565 | 0.845 | 1.372× |
| C64 → C32 | Bundle context | 0.504 | 0.382 | 0.872× |
| C64 → C32 | Lineage structure | 0.479 | 0.316 | 0.553× |
| C64 → C32 | Combined | **0.567** | **0.420** | **1.320×** |
| C64 → C32 | Same-corpus trace identity bound | 0.557 | 0.407 | 1.154× |

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Top-25%-capacity future-value lift",
  "width": 700,
  "height": 300,
  "data": {
    "values": [
      {"category": "C32→C64 · Bundle", "direction": "C32→C64", "lift": 0.555},
      {"category": "C32→C64 · Lineage", "direction": "C32→C64", "lift": 0.735},
      {"category": "C32→C64 · Combined", "direction": "C32→C64", "lift": 1.049},
      {"category": "C32→C64 · Identity bound", "direction": "C32→C64", "lift": 1.372},
      {"category": "C64→C32 · Bundle", "direction": "C64→C32", "lift": 0.872},
      {"category": "C64→C32 · Lineage", "direction": "C64→C32", "lift": 0.553},
      {"category": "C64→C32 · Combined", "direction": "C64→C32", "lift": 1.320},
      {"category": "C64→C32 · Identity bound", "direction": "C64→C32", "lift": 1.154}
    ]
  },
  "layer": [
    {
      "mark": {"type": "bar"},
      "encoding": {
        "x": {"field": "category", "type": "nominal", "sort": null, "title": null, "axis": {"labelAngle": -30}},
        "y": {"field": "lift", "type": "quantitative", "title": "Value lift over random capacity", "scale": {"zero": true}},
        "color": {"field": "direction", "type": "nominal", "title": "Held-out direction"},
        "tooltip": [
          {"field": "category", "type": "nominal"},
          {"field": "lift", "type": "quantitative", "format": ".3f"}
        ]
      }
    },
    {
      "mark": {"type": "rule", "strokeDash": [5, 5], "color": "#555"},
      "encoding": {"y": {"datum": 1.0}}
    }
  ]
}
~~~

The future-references-per-block target gives the same qualitative result: combined byte-weighted AUC is 0.520 for C32→C64 and 0.571 for C64→C32. The conclusion is not an artifact of choosing a binary label.

## Interpretation

Measured result: structural lineage alone is worse than random on capacity-weighted ranking in both directions. It becomes useful only in interaction with bundle context. The combined model crosses 0.5 byte-weighted AUC in both directions and gives positive top-capacity lift, but the C32→C64 gain is only 4.9%.

Inference: activity identity contains some real information, but the presently exported structural fields do not identify future value strongly enough to justify a live cache policy yet. Even the favorable same-corpus identity bound is only 1.15–1.37× and changes by pressure direction. The result argues against both extremes: identity is not useless, but conversation ID by itself is not the missing oracle.

Important limitation: C32 and C64 use the same source corpus and seed. The identity-bound result can memorize trace-specific behavior. The structural ablation is more defensible, but two executions are insufficient for uncertainty or cross-workload claims.

## Decision

**Positive data-availability result; weak predictive result; no live implementation yet.**

- No new GPU run was required.
- Do not discard lineage.
- Do not implement raw conversation-ID retention or another static classifier.
- Proceed to one more offline discriminating gate: add lineage to the age/deadline-conditioned arrival-versus-victim utility test, frozen bidirectionally.
- Require it to beat the stronger simple action at the same horizon in both directions, with positive net substitution utility—not merely AUC or gross hits.
- If that gate fails, stop model elaboration on these fields and pursue explicit application intent, route-to-data, tool/suspension events, or deterministic session placement.

## Reproduction

Local implementation:

- tools/costar/identity_information_audit.py
- tools/costar/run_identity_information_audit.py
- tests/tools/test_costar_identity_information_audit.py

Validation: 51 COSTAR tool tests passed; Ruff and git diff checks passed.

## Relationship to prior experiments

This result refines, rather than overturns, [[07 - Experiment 7 held-out soft bundle-value ranking|Experiment 7]] and [[08 - Experiment 8 held-out age-conditioned reuse hazard|Experiment 8]]. Static bundle and coarse hazard features remain insufficient. Lineage adds a small interaction signal, which now needs to survive the stricter deadline- and victim-aware utility gate before any policy replay.