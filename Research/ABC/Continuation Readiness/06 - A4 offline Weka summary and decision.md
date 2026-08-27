---
title: "COSTAR Continuation Readiness — A4 offline Weka summary and decision"
date: "2026-08-27"
type: "research-summary"
experiment: "COSTAR Continuation Readiness — A4"
status: "complete-negative"
workload: "AIPerf AgentX/Weka agentic replay"
---

# A4 offline Weka summary and decision

## Scope

This report consolidates the work performed after the Continuation Readiness research program identified A4 as the next gate. The question was whether raw Weka workflow/subagent information could provide a useful early signal for retention or prefetch decisions.

No live vLLM policy was changed and no new image was built.

## Source inspected

The cached corpus was:

`/Users/aperdomo/.cache/huggingface/hub/datasets--semianalysisai--cc-traces-weka-062126/blobs/29b6a19e751ff5230771519aab755f80a0f43a4ba9cf96b72d3a6a437ec99276`

The source contains nested subagent request arrays. Top-level events include normal requests and subagent markers. Marker records carry `agent_id`, `subagent_type`, `status`, `tool_use_count`, and nested `requests`. Normal requests carry timing, model, token counts, and KV `hash_ids`.

## Work completed

### 1. Strict request-export semantic audit

The existing c32/c64 AIPerf exports were checked for A4 fields.

| Corpus | Records | I0 | I1 | I2 | I3 | I4 | I5 |
|---|---:|---:|---:|---:|---:|---:|---:|
| c32 | 897 | 897 | 897 | 0 | 0 | 0 | 0 |
| c64 | 1,262 | 1,262 | 1,262 | 0 | 0 | 0 | 0 |

I0 is ordinary request/history information. I1 is stable continuation identity plus timing. I2–I5 require explicit lifecycle, tool, workflow, or successor fields, none of which are present in the exports.

Structural fields such as `source_kind` and `agent_depth` were intentionally not treated as lifecycle or tool semantics.

### 2. Raw Weka adapter

The adapter initially counted only top-level requests. Inspection revealed that subagent markers contain nested requests, so it was corrected to flatten nested records while preserving:

- `source_outer_idx`;
- `source_inner_idx`;
- marker identity and status;
- nested request timing and KV hashes;
- an explicit empty `explicit_semantic_fields` list.

Corrected corpus totals:

| Measure | Count |
|---|---:|
| Traces | 393 |
| Total events | 100,524 |
| Normal requests | 98,827 |
| Subagent markers | 1,697 |
| Markers with agent class/status | 1,697 / 1,697 |
| Exact within-stream next-event edges | 98,434 |

The flattened artifact is `/private/tmp/agentx-weka-a4-semantic.jsonl`.

### 3. Structural information analysis

The raw source contains:

- 1,697 subagent dispatch markers;
- 634 contiguous candidate-marker groups;
- 1,684 candidate-marker edges;
- candidate group size p50 = 2 and p95 = 6;
- only one observed marker class: `Subagent`;
- agent-class entropy = 0 bits.

This establishes that workflow fan-out exists, but the class signal is not discriminative.

### 4. Join to accepted vLLM oracle traces

Each accepted c32/c64 profile record was joined to both the raw Weka trace and the normalized vLLM oracle database.

| Corpus | Profile records | Exact server joins | Raw trace joins | Child records | Candidate groups | Child lead p50/p95 |
|---|---:|---:|---:|---:|---:|---:|
| c32 | 897 | 897 | 897 | 160 | 17 | 38.33 s / 279.90 s |
| c64 | 1,262 | 1,262 | 1,262 | 313 | 24 | 38.45 s / 232.97 s |

The source-to-server join is exact. The child subset accounted for:

- c32: 0.000 seconds of native external service;
- c64: 5.981 seconds of native external service.

### 5. Oracle-denominator comparison

The finite-capacity oracle opportunities from the accepted A1/A3 replay are:

- c32: 36.440 seconds;
- c64: 540.369 seconds.

The Weka subagent candidate subset therefore covers:

- c32: 0.00% of avoidable service;
- c64: approximately 1.11% of avoidable service.

This is an optimistic upper-bound comparison. It grants the structural signal perfect placement and does not charge policy mistakes. Even under that favorable assumption, it cannot explain the main opportunity.

### 6. Existing identity cross-check

The chronological identity audit was rerun:

- c32: 897/897 exact joins;
- c64: 1,262/1,262 exact joins.

Cross-corpus structural ranking remained weak:

| Direction | Combined reuse ROC-AUC | Byte-weighted ROC-AUC |
|---|---:|---:|
| c32 → c64 | 0.604 | 0.548 |
| c64 → c32 | 0.645 | 0.567 |

This independently confirms that coarse history/lineage is not a stable substitute for application intent.

## Implementation and verification

Added in the clean vLLM research worktree:

- `tools/costar/semantic_information_ladder.py`
- `tools/costar/run_semantic_information_ladder.py`
- `tools/costar/weka_semantic_trace.py`
- `tools/costar/run_weka_semantic_trace.py`
- `tools/costar/weka_information_ladder.py`
- `tools/costar/run_weka_information_ladder.py`
- `tools/costar/candidate_signal_join.py`
- `tools/costar/run_candidate_signal_join.py`
- corresponding focused tests under `tests/tools/`

Ruff is clean and all focused tests pass.

## Conclusion

The Weka-only semantic branch is closed as a negative offline result for this corpus.

The traces provide real topology and long marker-to-child lead times, but:

1. the signal covers too few requests;
2. it overlaps almost none of the expensive external KV service;
3. marker classes have no useful entropy;
4. exact successors are hindsight-only;
5. request-level lifecycle/tool/workflow fields are absent.

Therefore, we should not build a Weka-marker-driven vLLM prefetch or retention policy and should not create another image for this branch.

## Recommended next direction

The remaining defensible options are:

1. explicit application-provided lifecycle/tool hints, captured before request admission;
2. multi-replica route-to-data experiments;
3. tier-aware placement/eviction improvements that do not require speculative prediction.

The existing A1/A2/A3 evidence still supports the broader placement-policy problem. A4 only rejects this particular Weka structural signal as the control point.

## Source reports

- [[Research/ABC/Continuation Readiness/05 - A4 semantic information capture audit|A4 semantic information capture audit]]
- [[Research/ABC/Continuation Readiness/04 - A3 request-readiness allocation|A3 request-readiness allocation]]
- [[Research/ABC/Continuation Readiness/01 - A0 semantic trace certification|A0 semantic trace certification]]
- [[Research/ABC/Continuation Readiness/00 - Index|Continuation Readiness index]]