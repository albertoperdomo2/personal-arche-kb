---
title: "COSTAR Continuation Readiness A4 — semantic information capture audit"
date: "2026-08-27"
type: "research-experiment"
experiment: "COSTAR Continuation Readiness — A4"
status: "instrumentation-baseline"
workload: "AIPerf AgentX/Weka agentic replay"
corpora: ["c32", "c64"]
---

# A4 — Semantic information capture audit

## Aim

A4 asks whether explicit application state adds decision value beyond ordinary cache history and continuation timing. The information ladder is:

| Regime | Required information |
|---|---|
| I0 | request/key history |
| I1 | stable continuation ID, turn index, elapsed time |
| I2 | lifecycle state: active, tool pending, user wait, finished |
| I3 | tool class and agent class |
| I4 | workflow node and candidate successor set |
| I5 | exact application-known successor |
| I6 | future oracle (counterfactual only) |

The primary output is not a classifier score. It is incremental recovery of finite-capacity oracle service, request completeness, byte-second retention cost, false protection, and victim regret.

## Existing-corpus result

The audit was run against the accepted A0–A3 profile exports:

- c32: `/private/tmp/abc-oracle-validation-20260825/c32/profile_export.jsonl`
- c64: `/private/tmp/abc-oracle-validation-20260825/c64/profile_export.jsonl`

| Corpus | Records | I0 | I1 | I2 | I3 | I4 | I5 |
|---|---:|---:|---:|---:|---:|---:|---:|
| c32 | 897 | 897 | 897 | 0 | 0 | 0 | 0 |
| c64 | 1,262 | 1,262 | 1,262 | 0 | 0 | 0 | 0 |

I0 and I1 are complete. No record contains `session_state`, explicit completion reason, tool timing/class, agent class, workflow node, candidate successors, or an application-known successor request.

This is a measurement result, not a claim that the workload has no tools or workflow structure. The public Weka source format has subagent markers and structural fields such as `agent_id`, `subagent_type`, `tool_use_count`, and `status`; those fields are not present in the request-level AIPerf export used by the oracle replay. Structural `source_kind` and `agent_depth` are therefore not silently promoted to semantic labels.

## Reproducibility

Implementation:

- `tools/costar/semantic_information_ladder.py`
- `tools/costar/run_semantic_information_ladder.py`
- `tests/tools/test_costar_semantic_information_ladder.py`

Command:

```bash
PYTHONPATH=. /Users/aperdomo/workspace/redhat/vllm/.venv/bin/python \
  -m tools.costar.run_semantic_information_ladder \
  c32=/private/tmp/abc-oracle-validation-20260825/c32/profile_export.jsonl \
  c64=/private/tmp/abc-oracle-validation-20260825/c64/profile_export.jsonl
```

The focused tests pass (2/2 with the repository's global teardown disabled; the default macOS test teardown currently trips a PyTorch MPS allocator assertion unrelated to this tool).

## Capture contract for the next matched run

The producer (AgentX/AIPerf or an application-side adapter) must attach an opaque, versioned semantic object to each request before it reaches vLLM. A suggested request-body shape is:

```json
{
  "kv_transfer_params": {
    "costar_semantic_v1": {
      "session_state": "tool_pending",
      "tool_class": "search",
      "agent_class": "planner",
      "workflow_node_id": "node-17",
      "candidate_successors": ["node-18", "node-21"],
      "successor_request_id": null,
      "event_time_ns": 0,
      "terminal_reason": null
    }
  }
}
```

The same object must also be copied into the client profile export under `metadata`, keyed by `x_request_id`. The server-side oracle trace should preserve only the correlation needed for joining; it must not infer semantic state from prompt text.

Rules:

1. Values are enums/identifiers, not free-form prompt content.
2. `session_state` describes state at request admission; `terminal_reason` is set only when the application knows the session is terminal.
3. `candidate_successors` is a set of possible workflow nodes, not a future label.
4. `successor_request_id` is populated only when the application already knows the exact next request; it is I5 evidence, not a hindsight join.
5. Event timestamps must be producer timestamps and must precede request admission; post-hoc timestamps are invalid for lead-time analysis.
6. Missing fields remain missing. The audit must report per-field coverage and never substitute `source_kind`, `agent_depth`, response text, or turn index for semantic state.
7. The control and treatment use identical semantic metadata. The vLLM policy remains disabled for A4 information measurement.

The existing vLLM protocol already transports an opaque `kv_transfer_params` dictionary into request context. A4 does not yet change the live policy; it validates producer-to-request propagation first.

## Decision and next gate

A4 is **not yet a policy result**. The current corpus cannot distinguish I2–I5, so no conclusion about lifecycle/tool/workflow value is justified.

Next:

1. Add the capture object at the AgentX/AIPerf producer boundary.
2. Preserve it in the profile export and join it to vLLM requests by `x_request_id`.
3. Run matched c32 and c64 captures with identical seeds and no prefetch action.
4. Compute the I0→I5 decision ladder using chronological held-out windows and the existing finite-capacity replay.
5. Proceed to A5 only if lifecycle/tool state materially reduces false protection or recovers oracle service beyond the 30-second TTL baseline. If only exact successors help, implement an explicit application hint rather than a predictor.

## Sources

- [[Research/ABC/Continuation Readiness/00 - Index|Continuation Readiness index]]
- [[Research/ABC/Continuation Readiness/01 - A0 semantic trace certification|A0 semantic trace certification]]
- [[Research/ABC/Continuation Readiness/04 - A3 request-readiness allocation|A3 request-readiness allocation]]
- [AIPerf Weka trace format](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/weka-trace.md)
- [AgentX MVP scenario](https://github.com/ai-dynamo/aiperf/blob/main/docs/tutorials/agentx-mvp.md)