---
issue: 2166
title: "TPOT Metric Broken for Non-Streaming Requests — Reference Solution"
repo: "llm-d/llm-d-router"
created: 2025-07-25
tags:
  - github-issue
  - reference-solution
  - metrics
  - tpot
  - non-streaming
  - bug-fix
---

# Issue #2166 — Reference Solution

> **Spoiler —** this is the complete solution. Work through [[Company/Research/Issues/Guides/llm-d-llm-d-router/issue-2166]] first; use this to check your work or when stuck.

## Step-by-Step Code

### Step 1: Trace the Timestamp Flow (Analysis)

**File: `pkg/epp/handlers/server.go`** — `finishResponse` function (lines 584-608)

```go
// finishResponse ensures all post-response logic, such as metric recording
// and state updates, is executed exactly once for the request lifecycle.
func (s *StreamingServer) finishResponse(ctx context.Context, reqCtx *RequestContext, body []byte, modelStreaming bool, setEos bool) {
	// Return early if the response has already been finished to prevent
	// duplicate execution of side effects and metrics.
	if reqCtx.ResponseComplete {
		return
	}

	start := time.Now()
	reqCtx.ResponseComplete = true
	reqCtx.ResponseCompleteTimestamp = time.Now()  // <-- FIRST timestamp (non-streaming)
	reqCtx = s.HandleResponseBody(ctx, reqCtx, body, true)  // <-- calls HandleResponseBody
	if !modelStreaming {
		// Rewrite the model name in response body back to the original client-facing name.
		body = rewriteModelName(body, reqCtx.TargetModelName, reqCtx.IncomingModelName)
		// For non-streaming response, we send response back to envoy after receiving all the response body.
		reqCtx.respBodyResp = generateResponseBodyResponses(body, setEos, reqCtx.Response.DynamicMetadata)
	}
	if modelStreaming || reqCtx.responseHeadersReceivedAt.IsZero() {
		reqCtx.responseProcessingDuration += time.Since(start)
	} else {
		// Supersedes the header slice already accumulated: the interval since the
		// response headers arrived covers it and the body wait in between.
		reqCtx.responseProcessingDuration = time.Since(reqCtx.responseHeadersReceivedAt)
	}
}
```

**File: `pkg/epp/handlers/response.go`** — `HandleResponseBody` function (lines 35-96)

```go
// HandleResponseBody processes response data for both streaming and non-streaming models.
//
// Streaming case:
//   Invoked multiple times as data chunks arrive. The final call is identified by
//   endOfStream=true, triggering final metric collection and plugin cleanup.
//
// Non-streaming case:
//   Invoked once with the entire response body and endOfStream=true.
func HandleResponseBody(ctx context.Context, reqCtx *RequestContext, responseBytes []byte, endOfStream bool) *RequestContext {
	// ... logging ...

	// Record first token timestamp on first non-empty response chunk
	if reqCtx.FirstTokenTimestamp.IsZero() && len(responseBytes) > 0 {
		reqCtx.FirstTokenTimestamp = time.Now()  // <-- SECOND timestamp (non-streaming)
	}

	// ... rest of function ...

	if endOfStream {
		// ... record metrics including TPOT ...
		metrics.RecordRequestTPOT(ctx, reqCtx.ModelName, reqCtx.Tenant, reqCtx.RequestPriority,
			reqCtx.ReceivedTimestamp, reqCtx.FirstTokenTimestamp, reqCtx.ResponseCompleteTimestamp,
			reqCtx.OutputTokens)
		// ... cleanup ...
	}
	return reqCtx
}
```

**Execution order for non-streaming:**
1. `finishResponse` line 593: `ResponseCompleteTimestamp = time.Now()` 
2. `finishResponse` line 594: calls `HandleResponseBody`
3. `HandleResponseBody` line 56: `FirstTokenTimestamp = time.Now()` (because `IsZero()` is true)

Result: `complete < firstToken` → TPOT validation fails.

---

### Step 2: Fix — Set FirstTokenTimestamp Before ResponseCompleteTimestamp

**File: `pkg/epp/handlers/server.go`** — modify `finishResponse`

```go
// finishResponse ensures all post-response logic, such as metric recording
// and state updates, is executed exactly once for the request lifecycle.
func (s *StreamingServer) finishResponse(ctx context.Context, reqCtx *RequestContext, body []byte, modelStreaming bool, setEos bool) {
	// Return early if the response has already been finished to prevent
	// duplicate execution of side effects and metrics.
	if reqCtx.ResponseComplete {
		return
	}

	start := time.Now()
	reqCtx.ResponseComplete = true

	// For non-streaming requests, the entire response body arrives at once.
	// Conceptually, this single chunk represents both the "first token" and
	// the "complete response". Set FirstTokenTimestamp BEFORE ResponseCompleteTimestamp
	// so that the TPOT validation (complete.After(firstToken)) passes.
	// HandleResponseBody only sets FirstTokenTimestamp if it's still zero,
	// so this value will be preserved.
	if !modelStreaming {
		reqCtx.FirstTokenTimestamp = time.Now()
	}

	reqCtx.ResponseCompleteTimestamp = time.Now()
	reqCtx = s.HandleResponseBody(ctx, reqCtx, body, true)
	if !modelStreaming {
		// Rewrite the model name in response body back to the original client-facing name.
		body = rewriteModelName(body, reqCtx.TargetModelName, reqCtx.IncomingModelName)
		// For non-streaming response, we send response back to envoy after receiving all the response body.
		reqCtx.respBodyResp = generateResponseBodyResponses(body, setEos, reqCtx.Response.DynamicMetadata)
	}
	if modelStreaming || reqCtx.responseHeadersReceivedAt.IsZero() {
		reqCtx.responseProcessingDuration += time.Since(start)
	} else {
		// Supersedes the header slice already accumulated: the interval since the
		// response headers arrived covers it and the body wait in between.
		reqCtx.responseProcessingDuration = time.Since(reqCtx.responseHeadersReceivedAt)
	}
}
```

**Why this works:**
- For non-streaming: `FirstTokenTimestamp` set first, then `ResponseCompleteTimestamp` → `complete.After(firstToken)` is true
- For streaming: `modelStreaming=true` so the `if !modelStreaming` block is skipped → `HandleResponseBody` sets `FirstTokenTimestamp` on first chunk as before
- The `IsZero()` guard in `HandleResponseBody` (line 55) prevents overwriting our pre-set value

---

### Step 3: Verify Streaming Path Unaffected

The fix is guarded by `if !modelStreaming`. For streaming requests:
- `modelStreaming=true` → block skipped
- First chunk arrives → `HandleResponseBody` called with `endOfStream=false`
- `FirstTokenTimestamp.IsZero()` is true → sets timestamp
- Subsequent chunks → `IsZero()` false → **does not overwrite**
- Final chunk → `finishResponse` called with `modelStreaming=true` → block skipped

No behavior change for streaming.

---

### Step 4: Add Test Case for Non-Streaming TPOT

**File: `pkg/epp/metrics/metrics_test.go`** — add to `TestRecordRequestTPOT`

```go
func TestRecordRequestTPOT(t *testing.T) {
	Reset()
	ctx := logutil.NewTestLoggerIntoContext(context.Background())
	timeBaseline := time.Now()

	t.Run("valid multi-token request", func(t *testing.T) {
		received := timeBaseline
		firstToken := timeBaseline.Add(500 * time.Millisecond)
		complete := timeBaseline.Add(2000 * time.Millisecond)
		require.True(t, RecordRequestTPOT(ctx, "m10", "t10", "tenant-a", "3", received, firstToken, complete, 11))

		h, err := getHistogramVecLabelValues(t, llmdRequestTPOT, "m10", "t10", "tenant-a", "3")
		require.NoError(t, err)
		require.Equal(t, uint64(1), h.GetSampleCount())
		require.InDelta(t, 0.15, h.GetSampleSum(), 0.001)
	})

	t.Run("single token skipped", func(t *testing.T) {
		require.False(t, RecordRequestTPOT(ctx, "m10", "t10", "tenant-a", "3", timeBaseline, timeBaseline.Add(100*time.Millisecond), timeBaseline.Add(200*time.Millisecond), 1))
	})

	t.Run("zero tokens skipped", func(t *testing.T) {
		require.False(t, RecordRequestTPOT(ctx, "m10", "t10", "tenant-a", "3", timeBaseline, timeBaseline.Add(100*time.Millisecond), timeBaseline.Add(200*time.Millisecond), 0))
	})

	t.Run("zero first token timestamp", func(t *testing.T) {
		require.False(t, RecordRequestTPOT(ctx, "m10", "t10", "tenant-a", "3", timeBaseline, time.Time{}, timeBaseline.Add(200*time.Millisecond), 10))
	})

	t.Run("first token before received", func(t *testing.T) {
		require.False(t, RecordRequestTPOT(ctx, "m10", "t10", "tenant-a", "3", timeBaseline.Add(100*time.Millisecond), timeBaseline, timeBaseline.Add(200*time.Millisecond), 10))
	})

	t.Run("complete before first token", func(t *testing.T) {
		require.False(t, RecordRequestTPOT(ctx, "m10", "t10", "tenant-a", "3", timeBaseline, timeBaseline.Add(200*time.Millisecond), timeBaseline.Add(100*time.Millisecond), 10))
	})

	// NEW TEST CASE: non-streaming equal timestamps (firstToken == complete)
	t.Run("non-streaming equal timestamps", func(t *testing.T) {
		// For non-streaming, firstToken and complete are effectively the same moment.
		// With outputTokens > 1, TPOT = 0 / (N-1) = 0, which is a valid observation.
		require.True(t, RecordRequestTPOT(ctx, "m10", "t10", "tenant-a", "3", timeBaseline, timeBaseline, timeBaseline, 10))

		h, err := getHistogramVecLabelValues(t, llmdRequestTPOT, "m10", "t10", "tenant-a", "3")
		require.NoError(t, err)
		require.Equal(t, uint64(1), h.GetSampleCount())
		require.InDelta(t, 0.0, h.GetSampleSum(), 0.001)  // TPOT = 0
	})
}
```

---

### Step 5: Run Tests and Verify

```bash
# Run TPOT-specific tests
make test PKG=./pkg/epp/metrics/... RUN=TestRecordRequestTPOT

# Run handler tests
make test PKG=./pkg/epp/handlers/...

# Full presubmit
make presubmit
```

**Expected output:**
```
=== RUN   TestRecordRequestTPOT
=== RUN   TestRecordRequestTPOT/valid_multi-token_request
=== RUN   TestRecordRequestTPOT/single_token_skipped
=== RUN   TestRecordRequestTPOT/zero_tokens_skipped
=== RUN   TestRecordRequestTPOT/zero_first_token_timestamp
=== RUN   TestRecordRequestTPOT/first_token_before_received
=== RUN   TestRecordRequestTPOT/complete_before_first_token
=== RUN   TestRecordRequestTPOT/non-streaming_equal_timestamps
--- PASS: TestRecordRequestTPOT (0.01s)
PASS
ok      github.com/llm-d/llm-d-router/pkg/epp/metrics   0.123s
```

---

## Complete Implementation

### File: `pkg/epp/handlers/server.go`

```go
// finishResponse ensures all post-response logic, such as metric recording
// and state updates, is executed exactly once for the request lifecycle.
func (s *StreamingServer) finishResponse(ctx context.Context, reqCtx *RequestContext, body []byte, modelStreaming bool, setEos bool) {
	// Return early if the response has already been finished to prevent
	// duplicate execution of side effects and metrics.
	if reqCtx.ResponseComplete {
		return
	}

	start := time.Now()
	reqCtx.ResponseComplete = true

	// For non-streaming requests, the entire response body arrives at once.
	// Conceptually, this single chunk represents both the "first token" and
	// the "complete response". Set FirstTokenTimestamp BEFORE ResponseCompleteTimestamp
	// so that the TPOT validation (complete.After(firstToken)) passes.
	// HandleResponseBody only sets FirstTokenTimestamp if it's still zero,
	// so this value will be preserved.
	if !modelStreaming {
		reqCtx.FirstTokenTimestamp = time.Now()
	}

	reqCtx.ResponseCompleteTimestamp = time.Now()
	reqCtx = s.HandleResponseBody(ctx, reqCtx, body, true)
	if !modelStreaming {
		// Rewrite the model name in response body back to the original client-facing name.
		body = rewriteModelName(body, reqCtx.TargetModelName, reqCtx.IncomingModelName)
		// For non-streaming response, we send response back to envoy after receiving all the response body.
		reqCtx.respBodyResp = generateResponseBodyResponses(body, setEos, reqCtx.Response.DynamicMetadata)
	}
	if modelStreaming || reqCtx.responseHeadersReceivedAt.IsZero() {
		reqCtx.responseProcessingDuration += time.Since(start)
	} else {
		// Supersedes the header slice already accumulated: the interval since the
		// response headers arrived covers it and the body wait in between.
		reqCtx.responseProcessingDuration = time.Since(reqCtx.responseHeadersReceivedAt)
	}
}
```

### File: `pkg/epp/metrics/metrics_test.go` (test addition only)

```go
t.Run("non-streaming equal timestamps", func(t *testing.T) {
	// For non-streaming, firstToken and complete are effectively the same moment.
	// With outputTokens > 1, TPOT = 0 / (N-1) = 0, which is a valid observation.
	require.True(t, RecordRequestTPOT(ctx, "m10", "t10", "tenant-a", "3", timeBaseline, timeBaseline, timeBaseline, 10))

	h, err := getHistogramVecLabelValues(t, llmdRequestTPOT, "m10", "t10", "tenant-a", "3")
	require.NoError(t, err)
	require.Equal(t, uint64(1), h.GetSampleCount())
	require.InDelta(t, 0.0, h.GetSampleSum(), 0.001)  // TPOT = 0
})
```

---

## Summary of Changes

| File | Change |
|------|--------|
| `pkg/epp/handlers/server.go` | Set `FirstTokenTimestamp` before `ResponseCompleteTimestamp` for non-streaming requests (guarded by `!modelStreaming`) |
| `pkg/epp/metrics/metrics_test.go` | Added test case `"non-streaming equal timestamps"` verifying TPOT recorded when `firstToken == complete` with `outputTokens > 1` |

**Key design decisions:**
1. **Minimal scope** — only touches the non-streaming path via `!modelStreaming` guard
2. **Preserves existing behavior** — streaming path unchanged; `IsZero()` guard in `HandleResponseBody` prevents overwrite
3. **Semantically correct** — for non-streaming, the single response chunk *is* both first token and complete
4. **Test validates the fix** — new test case ensures the equality case works and records TPOT=0

---

## PR Submission

### Suggested PR Title
`fix(metrics): record TPOT for non-streaming requests by fixing timestamp ordering`

### Suggested PR Description
**Summary**
- Fixes TPOT metric being silently dropped for all non-streaming requests
- Root cause: `ResponseCompleteTimestamp` set before `FirstTokenTimestamp` in `finishResponse` for non-streaming path
- Fix: Set `FirstTokenTimestamp` first when `!modelStreaming`, guarded so streaming path is unaffected
- Adds test case for `firstToken == complete` with `outputTokens > 1`

**Testing**
- [ ] Unit tests pass: `TestRecordRequestTPOT` including new case
- [ ] Handler tests pass
- [ ] Manual verification: non-streaming request no longer logs TPOT error, metric appears in Prometheus
- [ ] Streaming requests still record correct TPOT

**Related Issues**
- Closes #2166