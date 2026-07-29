---
title: "AgentX MVP workload definition"
date: 2026-07-29
type: reference
topic: KV Cache Offloading
source: https://raw.githubusercontent.com/ai-dynamo/aiperf/a75c4612b6f8cf7e3bc88e6cd560fbf9be7c1a69/docs/tutorials/agentx-mvp.md
benchmark: aiperf-agentx-inference
trace: semianalysisai/cc-traces-weka-with-subagents-060826
---

# AgentX MVP — workload definition

This document characterizes the **AgentX MVP** workload that drives every standardized KV-cache offloading experiment in this research. It captures *how the workload is generated* — the trace corpus, session/turn/subagent topology, arrival structure, cache-bust mechanism, and warmup/profiling phases — distinct from the *observed run outcomes* recorded in the per-model experiment reports. Source: the AIPerf `agentx-mvp.md` tutorial at the pinned commit above.

The [[01 - Calibration Protocol|calibration protocol]] and [[02 - Per-Model Methodology Template|per-model methodology template]] both reference a "Workload Definition"; this is it.

## What it is

AgentX MVP is a multi-turn, agentic-coding benchmark proposed by SemiAnalysis as part of their InferenceX effort. It replays real coding-agent sessions against an inference server instead of synthetic single-turn prompts, so it exercises **KV-cache reuse and inter-turn think time** — exactly the mechanisms this research stresses. It is run through AIPerf (`benchmark: aiperf-agentx-inference`).

## Trace corpus

| Field | Value |
|---|---|
| Corpus | `semianalysisai/cc-traces-weka-with-subagents-060826` (HuggingFace, public, no auth) |
| Format | WEKA agentic-coding traces, captured by Callan Fox's `kv-cache-tester` |
| Variant | `with-subagents` — parent sessions may spawn helper conversations that rejoin the parent |
| Trace count | **391 traces** for the pinned with-subagents corpus |
| Subagent entries | 615 subagent entries expand to ~3.1k chain children |

Each trace is a full multi-turn coding session (real Claude Code sessions captured byte-for-byte), with recorded request-start timestamps, inter-turn idle gaps, and hash-id LCP chains for prefix tracking.

## Session / turn / subagent topology

- A **parent** coding session is a sequence of user/assistant **turns** with recorded start times and idle gaps between them.
- Parent turns can spawn **one or more helper conversations (subagents)** that rejoin the parent before it resumes. The parent's next anchored turn waits on the corresponding `SPAWN_JOIN` prerequisite.
- Subagents with a preceding and following parent anchor become `SPAWN`/`JOIN` branches.
- **Background subagents** with no following anchor do **not** block the parent.
- Adjacent subagents sharing the same anchors collapse into one multi-child branch.
- Within a subagent entry, nested hash-id LCP-chain detection splits inner requests into per-context-chain children: the subagent's own thread `::sa:<agent_id>` plus `:c000`, `:c001`, … siblings.
- Each chain child dispatches at its **recorded offset** from the spawn rather than bursting.
- Helper conversations can run alongside their parent and push **instantaneous in-flight request count above `--concurrency`**.

This topology is why the nominal concurrency is not the effective active concurrency — observed across our runs (see [[#Relation to observed run behavior|below]]).

## Arrival and dependency structure

- **Timing mode**: `agentic_replay` — a multi-turn agentic-replay scheduler. Locked.
- Trace-derived delays are **preserved** (not ignored), with idle-gap compression applied.
- **Idle-gap cap**: `--trace-idle-gap-cap-seconds = 60`. Gaps between recorded request starts over 60 s are compressed to 60 s per trace. The cap preserves relative subagent overlap rather than clamping each parent-turn delay independently.
- When a trajectory finishes its conversation, its trace ID goes into a **FIFO recycle queue** and the slot picks up the next trace ID.
- As long as the corpus is larger than the trajectory count, every trace is played at least once before any trace is replayed twice.
- Subagent chain children replay on the recorded timeline relative to their spawn point.

Effective offered concurrency is therefore bounded by both `--concurrency` and the subagent fan-out at any instant:

$$
N_{\text{in-flight}}(t) \le \underbrace{N_{\text{active trajectories}}}_{=C} + \underbrace{N_{\text{live subagent children}}(t)}_{\text{fan-out above } C}.
$$

## Prefix reuse and cache-bust

- **Cache-bust**: `--cache-bust first_turn_prefix` (locked). A unique per-conversation marker like `[rid:8a3f2c1b9e7d]\n\n` is prepended to the first user turn for every play — one injection per play, shared across all turns of that play.
- The tag is derived deterministically **within a single run** from benchmark ID, recycle pass, trajectory index, and trace ID.
- Without it, every recycled replay would warm the server's prefix cache further on identical content and inflate steady-state cache-hit rates. The bust keeps measured reuse attributable to **within-play** prefix sharing, not cross-play cache warming.
- **Warmup-to-profiling continuity**: a trajectory's warmup turn `k_i` and its first profiling turn `k_i+1` carry the *same* `[rid:…]`, so KV-cache prefix work done during warmup transfers into measurement.

### Live vs. pre-canned assistant turns

- **Default (pre-canned)**: assistant content is synthesized from `prev_out_tokens` and recorded `hash_ids`; the wire prompt's hash chain matches the original recording byte-for-byte. But server-generated assistant tokens from turn N never appear in turn N+1's prompt, so **measured cache-hit rate underweights the assistant prefix**.
- **Optional** (`AIPERF_DATASET_WEKA_LIVE_ASSISTANT_RESPONSES=1`): the worker emits user-only deltas, captures the server's live response, and threads it into the next turn's prompt — cache-hit rate then reflects what a real agentic user experiences.
- Caveat with live mode: server-generated assistant length will not exactly match the trace's recorded `output_length`, so hash-id equality past turn 0 is not preserved.

## Token sizes

- **Max context**: `--max-context-length` (example `128_000`; should match the server's configured `max-model-len`). Traces whose peak input length exceeds this are dropped before replay.
- **Output**: `ignore_eos = true` is locked — the server is told to ignore its end-of-stream token and generate the full requested length.
- **No client-side truncation**: `--synthesis-max-isl` is rejected.
- The tutorial does not publish a corpus-wide ISL/OSL histogram; per-turn lengths come from the trace. In our runs the **mean input-token count per request is ~50k–58k**, and theoretical AgentX KV reuse is ~93–94% — the workload is a large-prefix, high-reuse stressor (see [[AgentX Cross-Model/00 - Report|AgentX cross-model report]]).

## Concurrency and duration

- `--concurrency` is a single integer (comma-list sweeps are rejected). Example / standard value: **32**. It controls the number of active parent trajectories, not a hard cap on every parent-plus-subagent request.
- If the usable trajectory pool is smaller than requested concurrency, AIPerf wrap-fills extra lanes by cycling through that pool.
- **Duration**: default **1,800 s** (30 min) when `--benchmark-duration` is unset; minimum **900 s** (15 min); shorter runs are rejected.
- Profiling ends when duration elapses; in-flight requests finish during a cooldown window and are included in metrics.

## Warmup phase

Trajectory-based warmup is specific to the agentic-replay scheduler (not generic AIPerf warmup):

1. Scheduler builds N active trajectory lanes (N = `--concurrency`).
2. For each lane, samples a random starting turn `k_i` between 25% and 75% of that conversation's turns (`--trajectory-start-min-ratio` / `--trajectory-start-max-ratio`), clamped to leave at least one profile turn after warmup.
3. Dispatches warmup turn(s): turn `k_i` for simple (non-subagent) trajectories, with the full prefix history (turns 0 through `k_i-1`) attached as message context. Lanes with live subagent branches at `k_i` may dispatch one warmup credit per ready branch.
4. Warmup ends when every warmup request has resolved (success or failure). The warmup grace period defaults to no limit.
5. If any warmup request fails terminally, AIPerf aborts with `TrajectoryWarmupFailedError`.

The `k_i` values are deterministic given the random seed, and `k_i` shares its `[rid:…]` marker with the first profiling turn.

## Profiling phase

- Each trajectory keeps replaying its conversation from turn `k_i + 1` onward, honoring the recorded request-start schedule after the 60 s idle-gap compression.
- **Recycled traces start at turn 0**, not at a random `k_i` — the mid-conversation start applies only to initial trajectories.
- Each play of a trace gets a fresh cache-bust marker.
- Profiling ends when `--benchmark-duration` elapses; in-flight requests finish during cooldown.

## Configuration knobs

### Locked by scenario (need `--unsafe-override` to change)

| Setting | Locked value |
|---|---|
| `timing_mode` | `agentic_replay` |
| `extra_inputs.ignore_eos` | `true` |
| `--ignore-trace-delays` | off (delays preserved) |
| `--trace-idle-gap-cap-seconds` | 60 |
| `--cache-bust` | `first_turn_prefix` |
| `--benchmark-duration` | ≥ 900 (default 1800) |
| `--synthesis-max-isl` | rejected (no client-side truncation) |
| `--random-seed` | set (auto-picked if not passed) |
| Loader | `semianalysis_cc_traces_weka_with_subagents`, `weka_trace`, or constrained `weka_hf` |

### Not locked (user chooses)

| Flag | Notes |
|---|---|
| `--model` | Rewritten into trace requests; a single model maps all trace models to it; multiple models map by first-appearance order |
| `--max-context-length` | Drops traces exceeding this; should match server config |
| `--concurrency` | Single integer; user's choice |
| `--streaming` | Not forced; recommended since traces were captured against streaming responses |
| `--num-profile-runs` | Optional; ≥2 recommended for confidence-interval reporting |
| `--num-dataset-entries` | Optional; omitted = full 391 traces |
| `--apply-chat-template` | Optional; affects ISL reporting (wire-token total vs. bare text) |
| `--use-server-token-count` | Optional; trusts server `usage.completion_tokens` for OSL instead of local re-tokenization |
| `AIPERF_DATASET_WEKA_LIVE_ASSISTANT_RESPONSES` | Env var; defaults to `0`/`False` |

## Metrics / output fields

- **Per-run**: `profile_export.json` / `profile_export_aiperf.json` with a `metadata` block holding `scenario`, `submission_valid`, and `submission_invalid_reasons`.
- **Aggregate** (when `--num-profile-runs >= 2`): `aggregate/profile_export_aiperf_aggregate.json`.
- `submission_valid`: `true` / `false` / absent (absent when run without `--scenario`).
- `submission_invalid_reasons`: tags — `"unsafe_override"`, `"context_overflow_rate_exceeded"`, `"run_cancelled"`.
- `context_overflow_count` in `metrics` (requests); threshold is >1% of responses.
- Standard AIPerf metrics in the `metrics` block: throughput, latency percentiles (TTFT / ITL / E2E), cache-hit rate, ISL/OSL, prefill/decode mix. Per-request identity (`conversation_id`, turn, depth) supports the matched-pair tail analysis used in our reports.

`--unsafe-override` converts every scenario-rule violation from an error into a warning and stamps `submission_valid: false` with `"unsafe_override"` (only when a rule was actually broken; otherwise a no-op). It cannot be un-set at runtime — the result is marked invalid forever. It is a no-op without `--scenario`.

## Relation to observed run behavior

This generation mechanism explains several things we have repeatedly observed in the experiment reports:

- **Effective concurrency < nominal concurrency.** The `2026-07-19 vLLM 0.24 fixed-EPP` analysis found effective active concurrency of ~61 (no-offload) dropping to ~43 (CPU+NVMe offload) at nominal C=128. This is the subagent-fan-out-plus-idle-gap structure above: agentic dependencies and idle gaps yield far fewer simultaneously active model requests than `--concurrency` implies, and offload variants complete work faster so they carry fewer in-flight requests. The workload is therefore good for demonstrating cache benefit but, at standard settings, not a strong secondary-tier stress test.
- **Large, highly-reusable prefixes.** Theoretical reuse ~93–94% and mean input ~50–58k tokens per request are a direct consequence of replaying full multi-turn coding sessions with `first_turn_prefix` cache-bust but otherwise shared turn history within a play.
- **Deterministic cross-config pairing.** Because trajectories and `k_i` are seed-determined and per-turn identity is stable, our reports pair 1,316 matched requests by `conversation_id`/turn/depth and find only a 3-token p95 prompt-length difference between draws — ruling out easy/hard trace-draw artifacts.
- **Duration-boundary cancellations, not failures.** In-flight requests at the 1800 s boundary (plus the cooldown / cancellation-credit window) are cancelled, not HTTP errors. A clean run reports zero 500/503 and zero truncated streams; `submission_valid=true` does not by itself mean a clean steady-state capacity point (see the `2026-07-20 C256 U0.85` stress analysis: 234 of 1,384 requests cancelled at the boundary, 16.9%).

## Open gaps

- **No corpus-wide ISL/OSL / session-length histogram** is published in the tutorial or captured here; per-turn lengths are trace-derived. If a raw distribution is needed (e.g., to size a secondary tier against the prefix-length CDF), it must be derived from the trace repository or the benchflow profile directly, not from run outcomes.
- **Live-assistant mode** (`AIPERF_DATASET_WEKA_LIVE_ASSISTANT_RESPONSES=1`) has not been exercised in this research; all runs use pre-canned assistant turns, so measured cache-hit rates underweight the assistant prefix.

## Related

- [[00 - Index|KV Cache Offloading index]]
- [[01 - Calibration Protocol|Calibration protocol]] — pressure targets this workload is calibrated against
- [[02 - Per-Model Methodology Template|Per-model methodology template]]
- [[Cross-Model Synthesis|Cross-model synthesis]]
- [[AgentX Cross-Model/00 - Report|AgentX cross-model report]] — observed run outcomes for this workload