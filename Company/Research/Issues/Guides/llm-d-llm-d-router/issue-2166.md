---
issue: 2166
title: "TPOT Metric Broken for Non-Streaming Requests"
repo: "llm-d/llm-d-router"
url: "https://github.com/llm-d/llm-d-router/issues/2166"
created: 2025-07-25
status: analyzing
tags:
  - github-issue
  - implementation-guide
  - metrics
  - tpot
  - non-streaming
  - bug-fix
---

# Issue #2166: TPOT Metric Broken for Non-Streaming Requests

> **Issue metadata** — Repo: `llm-d/llm-d-router` · Issue: [#2166](https://github.com/llm-d/llm-d-router/issues/2166) · State: Open · Labels: bug, metrics
> Reference solution: [[Company/Research/Issues/Guides/llm-d-llm-d-router/issue-2166-solution]]
> Repository backlog: [[Company/Research/Issues/llm-d-llm-d-router]]

> **⚠️ Availability check —** Assignees: None · Active PRs: None · Claim signals in comments: None.
> If any of these indicate someone is already working on this issue, do not proceed without checking the linked PRs and recent comments first, to avoid duplicating effort.

## Overview

The **Time Per Output Token (TPOT)** metric is silently dropped for **all non-streaming requests** due to a timestamp ordering bug. In the non-streaming code path, `ResponseCompleteTimestamp` is set *before* `FirstTokenTimestamp`, causing the validation in `RecordRequestTPOT` to fail and log an error for every non-streaming request with more than one output token.

This guide walks through understanding the bug, the metrics pipeline, and implementing a minimal fix.

## Issue Context

### Problem Statement

For non-streaming requests:
1. `finishResponse()` in `pkg/epp/handlers/server.go:593` sets `ResponseCompleteTimestamp = time.Now()`
2. Then calls `HandleResponseBody()` at line 594
3. `HandleResponseBody()` in `pkg/epp/handlers/response.go:55-56` sets `FirstTokenTimestamp = time.Now()` (because it's the first/only chunk)
4. The two `time.Now()` calls are microseconds apart but **complete is set before firstToken**
5. `RecordRequestTPOT()` at `pkg/epp/metrics/metrics.go:687` checks `!complete.After(firstToken)` — this is **true** when complete ≤ firstToken
6. Result: **error logged** + **TPOT histogram never observed** for every non-streaming request with >1 output token

### Affected Components

| File | Role |
|------|------|
| `pkg/epp/handlers/server.go:584-608` | `finishResponse()` — sets timestamps in wrong order for non-streaming |
| `pkg/epp/handlers/response.go:35-96` | `HandleResponseBody()` — sets `FirstTokenTimestamp` on first chunk |
| `pkg/epp/metrics/metrics.go:681-698` | `RecordRequestTPOT()` — validates timestamp order, logs error, drops metric |

### Reproduction Steps

1. Send a non-streaming completion request (e.g., `curl -X POST ... -d '{"stream": false}'`)
2. Observe logs: `Error: "Request latency values are invalid for TPOT calculation"` for every such request
3. Check Prometheus: `llmd_request_tpot` histogram has **zero samples** for non-streaming traffic

## Theoretical Background

> **Key concepts —** the foundations you need to understand the problem and its solution. Don't skip this; it makes the implementation steps make sense.

### Concept 1: TPOT (Time Per Output Token)

**What it is:** The average time to generate each output token *after* the first token. For a request producing N tokens where the first token arrives at T₁ and the last at Tₙ:

$$\text{TPOT} = \frac{T_n - T_1}{N - 1}$$

**Why it matters:** TPOT measures decode-phase throughput. TTFT (Time To First Token) measures prefill latency; TPOT measures sustained generation speed. Both are critical SLOs for LLM serving.

**In this codebase:** `RecordRequestTPOT()` computes `(complete - firstToken).Seconds() / float64(outputTokens - 1)` and records it to a Prometheus histogram (`llmd_request_tpot`).

### Concept 2: Streaming vs Non-Streaming Response Handling

| Aspect | Streaming | Non-Streaming |
|--------|-----------|---------------|
| **Response delivery** | Chunked (SSE) | Single HTTP response |
| **`HandleResponseBody` calls** | Multiple (per chunk) | Once (entire body) |
| **First token detection** | First chunk with data | The single response body |
| **End-of-stream** | Explicit `endOfStream=true` | Implicit in `finishResponse` |

In the non-streaming path, the **entire response body arrives at once**. The code treats this as "first token = complete response" conceptually, but the timestamp ordering breaks the metric.

### Concept 3: The Timestamp Validation Logic

```go
// pkg/epp/metrics/metrics.go:681-698
func RecordRequestTPOT(...) bool {
    // ... validation ...
    if !complete.After(firstToken) {
        logger.Error(nil, "Request latency values are invalid for TPOT calculation",
            "firstTokenTimestamp", firstToken,
            "completeTimestamp", complete)
        return false  // Metric NOT recorded
    }
    // ... compute and record TPOT ...
}
```

The check `complete.After(firstToken)` requires **strictly after**. For non-streaming, `complete == firstToken` (same `time.Now()` call effectively), so the check fails.

### How These Concepts Connect

The bug is a **timestamp ordering issue** specific to the non-streaming code path:
- Streaming: first chunk → `FirstTokenTimestamp` set → later chunks → final chunk → `ResponseCompleteTimestamp` set ✓
- Non-streaming: `finishResponse` sets `ResponseCompleteTimestamp` → then calls `HandleResponseBody` which sets `FirstTokenTimestamp` ✗

The fix must ensure `FirstTokenTimestamp ≤ ResponseCompleteTimestamp` for non-streaming requests.

## Knowledge Gap Analysis

> **What you need to know** — check these off as you get comfortable. If any are unclear, revisit the background or the linked resources.

- [ ] Understanding of TPOT vs TTFT metrics in LLM serving
- [ ] Go `time.Time` comparison methods (`After`, `Before`, `Equal`)
- [ ] The ext-proc request/response flow in llm-d-router
- [ ] Difference between streaming and non-streaming response handling
- [ ] Prometheus histogram metrics in this codebase (`llmd_request_tpot`)

## Guided Implementation

> **How to use this section —** each step explains *why* you're making a change, gives you enough context to write the code yourself, then offers an approach hint and non-spoiler scaffolding. Try to write it before opening the [[…issue-2166-solution]] article.

### Step 1: Understand the Current Timestamp Flow

**What and why:** Trace through the non-streaming path to see exactly where timestamps are set and in what order. This confirms the root cause.

**Where to look:**
- `pkg/epp/handlers/server.go:584-608` — `finishResponse()` function
- `pkg/epp/handlers/response.go:35-96` — `HandleResponseBody()` function

> **Try it yourself —** Read both functions. Add a comment at each timestamp assignment showing the order they execute for a non-streaming request. What are the two `time.Now()` calls and which happens first?

> **Approach hint —** In `finishResponse`, the non-streaming path is the `!modelStreaming` branch. The call to `HandleResponseBody` happens *after* `ResponseCompleteTimestamp` is set.

**Scaffolding** (fill in the `TODO`s):
```go
// In pkg/epp/handlers/server.go — finishResponse function
func (s *StreamingServer) finishResponse(ctx context.Context, reqCtx *RequestContext, body []byte, modelStreaming bool, setEos bool) {
    // ... existing code ...

    // TODO: Add comment showing execution order for non-streaming:
    // 1. Line 593: reqCtx.ResponseCompleteTimestamp = time.Now()  <-- FIRST
    // 2. Line 594: reqCtx = s.HandleResponseBody(...)  <-- calls HandleResponseBody
    //    Inside HandleResponseBody (response.go:55-56):
    // 3. Line 56: reqCtx.FirstTokenTimestamp = time.Now()  <-- SECOND

    reqCtx.ResponseComplete = true
    reqCtx.ResponseCompleteTimestamp = time.Now()  // <-- FIRST timestamp
    reqCtx = s.HandleResponseBody(ctx, reqCtx, body, true)  // <-- sets FirstTokenTimestamp
    // ...
}
```

➡️ **Full code for this step:** see *Step 1* in [[Company/Research/Issues/Guides/llm-d-llm-d-router/issue-2166-solution]].

---

### Step 2: Choose the Fix Strategy

**What and why:** There are two valid approaches. Pick one and justify it.

**Option A: Set `FirstTokenTimestamp` before `ResponseCompleteTimestamp` in `finishResponse`**
- Pro: Minimal change, fixes ordering at the source
- Con: Need to handle the case where `HandleResponseBody` might overwrite it (but it only sets if `IsZero()`)

**Option B: Modify `RecordRequestTPOT` to accept `complete == firstToken` as valid for non-streaming**
- Pro: Handles the semantic case (non-streaming = single chunk = first token IS complete)
- Con: Changes metric validation logic, might mask real bugs in streaming path

> **Try it yourself —** Which approach is safer? Consider: does `HandleResponseBody` unconditionally overwrite `FirstTokenTimestamp`, or only if it's zero?

> **Approach hint —** Check `response.go:55`: `if reqCtx.FirstTokenTimestamp.IsZero() && len(responseBytes) > 0`. It only sets if zero. So pre-setting it in `finishResponse` is safe.

**Scaffolding** (fill in the `TODO`s):
```go
// In pkg/epp/handlers/server.go — finishResponse function
func (s *StreamingServer) finishResponse(ctx context.Context, reqCtx *RequestContext, body []byte, modelStreaming bool, setEos bool) {
    // ... existing code ...

    // TODO: For non-streaming, set FirstTokenTimestamp BEFORE ResponseCompleteTimestamp
    // Guard with !modelStreaming so streaming path is unaffected
    if !modelStreaming {
        reqCtx.FirstTokenTimestamp = time.Now()
    }

    reqCtx.ResponseComplete = true
    reqCtx.ResponseCompleteTimestamp = time.Now()
    reqCtx = s.HandleResponseBody(ctx, reqCtx, body, true)
    // ...
}
```

➡️ **Full code for this step:** see *Step 2* in [[Company/Research/Issues/Guides/llm-d-llm-d-router/issue-2166-solution]].

---

### Step 3: Add a Test Case

**What and why:** Verify the fix works by adding a test for the `firstToken == complete` case with `outputTokens > 1`.

**Where to look:** `pkg/epp/metrics/metrics_test.go:1436-1472` — `TestRecordRequestTPOT`

> **Try it yourself —** Add a new `t.Run` case that calls `RecordRequestTPOT` with `received == firstToken == complete` and `outputTokens = 10`. Expect it to return `true` and record TPOT = 0.

> **Approach hint —** Follow the existing test pattern. Use `timeBaseline` for all three timestamps. Verify the histogram gets one sample with sum ≈ 0.

**Scaffolding** (fill in the `TODO`s):
```go
// In pkg/epp/metrics/metrics_test.go — TestRecordRequestTPOT
t.Run("non-streaming equal timestamps", func(t *testing.T) {
    // TODO: For non-streaming, firstToken and complete are effectively the same moment.
    // With outputTokens > 1, TPOT = 0 / (N-1) = 0, which is a valid observation.
    require.True(t, RecordRequestTPOT(ctx, "m10", "t10", "tenant-a", "3", timeBaseline, timeBaseline, timeBaseline, 10))

    h, err := getHistogramVecLabelValues(t, llmdRequestTPOT, "m10", "t10", "tenant-a", "3")
    require.NoError(t, err)
    require.Equal(t, uint64(1), h.GetSampleCount())
    require.InDelta(t, 0.0, h.GetSampleSum(), 0.001)  // TPOT = 0
})
```

➡️ **Full code for this step:** see *Step 3* in [[Company/Research/Issues/Guides/llm-d-llm-d-router/issue-2166-solution]].

---

## Testing Strategy

> **Verify as you go**, not just at the end.

### Unit Tests
- [ ] `TestRecordRequestTPOT` — new case "non-streaming equal timestamps" passes
- [ ] All existing TPOT tests still pass (no regression on streaming path)
- [ ] Handler tests for `finishResponse` / `HandleResponseBody` pass

### Integration / Manual Testing
- [ ] Send non-streaming request → verify no TPOT error in logs
- [ ] Check Prometheus: `llmd_request_tpot` has samples for non-streaming traffic
- [ ] Send streaming request → verify TPOT still recorded correctly

Example test code lives in the [[…issue-2166-solution]] article.

## Related Resources

- [Issue #2166](https://github.com/llm-d/llm-d-router/issues/2166) — original issue
- [TPOT metric documentation](https://github.com/llm-d/llm-d-router/blob/main/docs/metrics.md) — if exists
- [ext-proc protocol](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_proc_filter) — Envoy external processor protocol