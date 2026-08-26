---
title: "COSTAR Continuation Readiness A0 — semantic trace certification"
date: "2026-08-26"
type: "research-experiment"
experiment: "COSTAR Continuation Readiness — A0"
status: "conditionally-certified"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "quay.io/rh-ee-aperdomo/vllm:v0.27.0-costar-oracle-trace-v1@sha256:0e79705305e63b50ac80454641c4ca277014ae0c47a664df5784a46eb079e17f"
vllm_version: "v0.27.0 experimental trace build"
tensor_parallelism: 8
replicas: 1
gpu: "8x H100"
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: "default / not explicitly set"
concurrency: [32, 64]
cpu_bytes: 274877906944
offload_spec: "TieringOffloadingSpec"
secondary_tier: "filesystem on local NVMe"
secondary_tier_threads: "64 read / 64 write"
workload: "AIPerf AgentX/Weka agentic replay"
random_seed: 20260707
duration_seconds: 1800
cache_cleaning_state: "not established by A0; unchanged from accepted corpus"
---

# COSTAR Continuation Readiness A0 — semantic trace certification

## Executive summary

A0 asked whether the accepted c32/c64 COSTAR corpora contain enough stable identity and timing information to evaluate continuation-aware retention without collecting another live trace.

**Answer: yes for A1–A3; no for A4 and tool-triggered A7.**

All 897 c32 and 1,262 c64 AIPerf records join exactly and unambiguously to their vLLM oracle-trace request through the client UUID embedded in the server request ID. The correct concrete continuation identifier is `x_correlation_id`. Grouped by that field, every one of the 782 c32 and 1,056 c64 observed continuation edges is chronological, advances by exactly one `turn_index`, and does not overlap the prior turn.

The files therefore support:

- A1 continuation-retention oracle over observed continuation chains;
- A2 static/empirical TTL replay with explicit right-censoring;
- A3 request-readiness-aware allocation using authoritative vLLM working sets.

They do not contain explicit tool start/end, lifecycle state, completion reason, workflow-node identity, or candidate successor fields. A4's semantic information ladder and A7's tool/workflow early-signal replay require a new enriched capture.

## Validity verdict

# Conditionally valid — certified for A1–A3

The request join and continuation ordering are exact for every exported client record. The certification is conditional because final observed turns are right-censored rather than explicitly terminal, and 3 c32/17 c64 aborted server requests lack exported client identity. Those requests must remain in physical replay as unknown/native-priority traffic rather than being assigned oracle continuation labels.

## Main takeaways

- **Measured:** c32 has 115 concrete continuations, 71 multi-turn; c64 has 206 concrete continuations, 123 multi-turn.
- **Measured:** 853/897 c32 records (95.10%) and 1,179/1,262 c64 records (93.42%) belong to an observed multi-turn continuation.
- **Measured:** all 1,838 observed edges are consecutive and chronologically ordered under `x_correlation_id`.
- **Measured:** median end-to-next-start gap is approximately 1.6 s in both corpora; p95 is 181.52 s at c32 and 261.44 s at c64.
- **Measured:** parent correlation is available for 258 c32 and 435 c64 records; 257 and 377 respectively resolve to another exported continuation.
- **Measured:** explicit tool, lifecycle, workflow-node, candidate-successor, and terminal-state coverage is zero.
- **Inference:** `conversation_id` is content/source identity, not a concrete serving session. At c64 the same source conversation is replayed in multiple concurrent trajectory instances. Using it as `session_id` creates false turn reversals and overlaps.
- **Decision:** use `x_correlation_id` as `continuation_id`, `root_correlation_id` as the root execution/session tree, and `conversation_id` only as reusable source/template identity.

## Corpus and identity coverage

| Measure | c32 | c64 |
|---|---:|---:|
| vLLM requests in normalized trace | 901 | 1,280 |
| AIPerf records | 897 | 1,262 |
| Exact AIPerf → vLLM joins | 897 (100%) | 1,262 (100%) |
| Ambiguous/unmatched AIPerf records | 0 / 0 | 0 / 0 |
| Unique source `conversation_id` values | 115 | 177 |
| Concrete `x_correlation_id` continuations | 115 | 206 |
| Multi-turn continuations | 71 | 123 |
| Records in multi-turn continuations | 853 (95.10%) | 1,179 (93.42%) |
| Observable continuation edges | 782 | 1,056 |
| Consecutive, increasing turn edges | 782 (100%) | 1,056 (100%) |
| Duplicate continuation/turn pairs | 0 | 0 |
| Overlapping edges within a continuation | 0 | 0 |
| Root execution/session trees | 56 | 99 |
| Child/branch continuations | 60 | 112 |

Figure 1 summarizes the coverage gates. Each percentage uses the relevant denominator: all AIPerf records for joins and multi-turn membership, observed edges for ordering, present parent links for parent resolution, and all records for explicit semantic events.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 1 — Continuation trace coverage gates",
  "width": 700,
  "height": 300,
  "data": {
    "values": [
      {"criterion": "C32 · exact request join", "coverage_pct": 100.0},
      {"criterion": "C64 · exact request join", "coverage_pct": 100.0},
      {"criterion": "C32 · records in multi-turn continuation", "coverage_pct": 95.10},
      {"criterion": "C64 · records in multi-turn continuation", "coverage_pct": 93.42},
      {"criterion": "C32 · consecutive ordered edges", "coverage_pct": 100.0},
      {"criterion": "C64 · consecutive ordered edges", "coverage_pct": 100.0},
      {"criterion": "C32 · resolvable parent links", "coverage_pct": 99.61},
      {"criterion": "C64 · resolvable parent links", "coverage_pct": 86.67},
      {"criterion": "C32 · explicit lifecycle/tool/workflow events", "coverage_pct": 0.0},
      {"criterion": "C64 · explicit lifecycle/tool/workflow events", "coverage_pct": 0.0}
    ]
  },
  "mark": {"type": "bar"},
  "encoding": {
    "y": {
      "field": "criterion",
      "type": "nominal",
      "title": "Corpus and certification criterion",
      "sort": null
    },
    "x": {
      "field": "coverage_pct",
      "type": "quantitative",
      "title": "Coverage (%)",
      "scale": {"domain": [0, 100], "zero": true}
    },
    "color": {
      "field": "criterion",
      "type": "nominal",
      "title": "Criterion",
      "scale": {"scheme": "category10"},
      "legend": null
    },
    "tooltip": [
      {"field": "criterion", "type": "nominal"},
      {"field": "coverage_pct", "type": "quantitative", "title": "Coverage (%)"}
    ]
  }
}
~~~

The chart shows complete join/order coverage and the exact semantic gap: lifecycle and workflow events were not captured.

## Continuation timing

| Gap from previous request end to next request start | c32 | c64 |
|---|---:|---:|
| p50 | 1.596 s | 1.602 s |
| p95 | 181.516 s | 261.439 s |

Figure 2 shows the observed end-to-next-start gap distribution summaries. These are replayed workload gaps, including the benchmark's agentic timing transformation; they are not raw production tool durations.

~~~vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "title": "Figure 2 — Observed continuation gap quantiles",
  "width": 620,
  "height": 260,
  "data": {
    "values": [
      {"corpus_quantile": "C32 · p50", "gap_seconds": 1.595536222},
      {"corpus_quantile": "C32 · p95", "gap_seconds": 181.516219723},
      {"corpus_quantile": "C64 · p50", "gap_seconds": 1.60170857},
      {"corpus_quantile": "C64 · p95", "gap_seconds": 261.439200844}
    ]
  },
  "mark": {"type": "bar"},
  "encoding": {
    "x": {
      "field": "corpus_quantile",
      "type": "nominal",
      "title": "Corpus and quantile",
      "sort": ["C32 · p50", "C32 · p95", "C64 · p50", "C64 · p95"]
    },
    "y": {
      "field": "gap_seconds",
      "type": "quantitative",
      "title": "End-to-next-start gap (s)",
      "scale": {"zero": true}
    },
    "color": {
      "field": "corpus_quantile",
      "type": "nominal",
      "title": "Corpus and quantile",
      "scale": {"scheme": "category10"},
      "legend": null
    },
    "tooltip": [
      {"field": "corpus_quantile", "type": "nominal"},
      {"field": "gap_seconds", "type": "quantitative", "title": "Gap (s)"}
    ]
  }
}
~~~

The p50 near 1.6 s and long p95 tail justify a TTL frontier rather than one fixed unbounded protection interval. A2 must measure byte-seconds and harmful protection rather than choose TTL from the mean.

## Field-level certification

| Required A0 field | c32 coverage | c64 coverage | Decision |
|---|---:|---:|---|
| `x_request_id` | 897/897 | 1,262/1,262 | Exact server join |
| `x_correlation_id` | 897/897 | 1,262/1,262 | Concrete continuation ID |
| `root_correlation_id` | 897/897 | 1,262/1,262 | Root execution/session tree |
| `conversation_id` / `source_trace_id` | 897/897 | 1,262/1,262 | Content/source identity only |
| `turn_index` | 897/897 | 1,262/1,262 | Ordered turn within continuation |
| Request start/end time | 897/897 | 1,262/1,262 | Exact observed continuation deadline/gap |
| `source_kind` / `agent_depth` | 897/897 | 1,262/1,262 | Structural proxy; already tested in prior lineage experiments |
| `parent_correlation_id` | 258/897 | 435/1,262 | Present for branch relations; not every parent survives export |
| Explicit completion/session terminal reason | 0 | 0 | Missing |
| Tool start/end and tool class | 0 | 0 | Missing |
| Lifecycle state | 0 | 0 | Missing |
| Workflow node / candidate successors | 0 | 0 | Missing |

## Correct identity model

The accepted identity mapping for the next replay is:

```text
root_correlation_id       root AgentX execution/session tree
    └── x_correlation_id  one concrete main/flat/subagent continuation
          └── turn_index  ordered request within that continuation

conversation_id           source/content identity; may be replayed more than once
x_request_id              unique client request; joins to vLLM request_id
```

This distinction is necessary at c64. Grouping only by `conversation_id` produced apparent turn resets, duplicate turns, and overlaps because a completed/source trajectory may be replayed again while an earlier instance is still finishing. Grouping by `x_correlation_id` removes every anomaly without heuristic repair.

## Replay handling rules

A1–A3 should use the following rules:

1. Join AIPerf and vLLM requests only through embedded `x_request_id`; do not use timestamps or token-count proximity.
2. Treat `x_correlation_id` as continuation identity.
3. Treat each adjacent `turn_index` pair within that continuation as an observed positive continuation edge.
4. Treat final observed turns as **right-censored**, not known-dead sessions.
5. Report two terminal bounds where terminal behavior matters:
   - optimistic oracle label: release after the last observed turn;
   - online-observable TTL: retain only until TTL or trace end because no terminal event exists.
6. Keep the 3 c32 and 17 c64 aborted server requests in physical occupancy/admission replay with unknown/native priority. Do not attach semantic oracle labels.
7. Exclude the one 7-token internal probe in each trace from policy scoring while retaining its physical events if the baseline replay requires them.
8. Preserve warmup/profiling chronology. The same concrete continuation can cross the phase boundary.
9. Do not interpret replayed gap duration as a measured real tool duration; the profile uses `system_idle_gap_cap_seconds=10` and agentic replay scheduling.

## Experiment eligibility decision

| Planned experiment | Existing artifacts sufficient? | Qualification |
|---|---|---|
| A1 continuation-retention oracle | **Yes** | Observed edges only; terminal turns right-censored |
| A2 soft TTL frontier | **Yes** | Use TTL/trace-end charging and explicit censoring |
| A3 readiness-aware allocation | **Yes** | Exact request join gives authoritative vLLM working sets |
| A4 semantic information ladder I0–I1 | **Yes** | Key history + stable continuation/time only |
| A4 lifecycle/tool/workflow regimes I2–I5 | **No** | New semantic capture required |
| A5 Markov/signature execution model | **Not yet** | Requires A4 evidence and tool/agent state |
| A6 route-to-data oracle | **No** | Requires multi-replica residency and routing trace |
| A7 tool/workflow prefetch feasibility | **No** | No legitimate early event timestamps |
| A8 combined action oracle | **No** | Requires A6/A7 inputs |

## Reproducibility

The audit implementation is local in:

- `tools/costar/continuation_trace_audit.py`
- `tools/costar/run_continuation_trace_audit.py`

Command:

```bash
PYTHONPATH=. /Users/aperdomo/workspace/redhat/vllm/.venv/bin/python \
  -m tools.costar.run_continuation_trace_audit \
  /private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite \
  /private/tmp/abc-oracle-validation-20260825/c32/profile_export.jsonl \
  /private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite \
  /private/tmp/abc-oracle-validation-20260825/c64/profile_export.jsonl
```

The implementation was run over both complete profile exports. The focused identity test passed (1/1), Python bytecode compilation succeeded, and Ruff reported no findings.

## Run registry

| Corpus | MLflow run | Disposition |
|---|---|---|
| c32 | [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow) | Certified for A1–A3 |
| c64 | [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow) | Certified for A1–A3 |

## Conclusion and next step

The existing trace corpus is sufficient to begin A1 immediately. No new vLLM image or live benchmark is required.

The next experiment is the finite-capacity continuation-retention oracle using mandatory CPU admission, no proactive reads, `x_correlation_id` continuation labels, observed next-turn deadlines, and right-censored terminal handling. Its result must be reported as recovery of the existing finite next-use oracle's avoidable secondary-service seconds.

A new semantically enriched trace should be deferred until A1–A3 establish whether continuation identity and readiness structure recover meaningful oracle value. If they do, the new capture must add explicit tool/lifecycle/workflow fields for A4.