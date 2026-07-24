---
repo: llm-d/llm-d-kv-cache
last_updated: 2026-07-24
---
# Backlog — llm-d/llm-d-kv-cache

## Workable Issues

- **Narrow integer block hashes cause KV-event batches to be discarded** · Medium · Confirmed · `issue #662` — Add table-driven coverage for every signed/unsigned integer width; reject negative values and overflow; submit fix to llm-d-router; close #662.  <!-- fp: llm-d/llm-d-kv-cache:issue:narrow-integer-block-hashes -->

## Bugs

- **Concurrent Add and Evict can lose an engine-to-request mapping** · Medium · Confirmed · `pkg/kvcache/kvblock/in_memory.go:170` — Acquire mutex before publishing engine mapping and retain through request-key insertion; add deterministic interleaving test; port to llm-d-router.  <!-- fp: llm-d/llm-d-kv-cache:bug:inmemory-add-evict-mapping-race -->

- **Failed tokenizer initialization leaks its gRPC connection and monitor goroutine** · Medium · Confirmed · `pkg/tokenization/uds_tokenizer.go:133` — Close connection on every post-dial constructor failure; start context-monitor goroutine only after initialization completes; add failure-path test.  <!-- fp: llm-d/llm-d-kv-cache:bug:uds-initialization-resource-leak -->

- **Periodic metrics logging continues after its owning context is cancelled** · Low · Confirmed · `pkg/kvcache/metrics/collector.go:121` — Replace channel range with select over ticker.C and ctx.Done(); add test proving goroutine exits after cancellation.  <!-- fp: llm-d/llm-d-kv-cache:bug:metrics-logger-ignores-context -->

## Performance

- **Cost-aware event ingestion rescans every pod entry on each mutation** · Medium · Confirmed · `pkg/kvcache/kvblock/cost_aware_memory.go:280` — Maintain estimated byte cost incrementally when Add/Delete changes membership; benchmark high-overlap prefixes; use cached value for Ristretto updates.  <!-- fp: llm-d/llm-d-kv-cache:perf:cost-aware-add-recounts-pod-cache -->

## Features & RFCs

- **Add context-aware tokenizer operations for cancellation and trace propagation** · Medium · Confirmed · `pkg/tokenization/tokenizer.go:37`
  - **Problem**: Public `Tokenizer` methods (`RenderChat`, `Render`) accept no `context.Context`. The UDS implementation creates RPC contexts from `context.Background()`, so request cancellation, deadlines, logger state, and OTel span context never reach the tokenization gRPC call. Abandoned requests consume resources until fixed internal deadlines (5 s receive, 30 s close) expire.
  - **Proposed approach**: Add `RenderChatWithCtx(ctx, ...)` and `RenderWithCtx(ctx, ...)` to the `Tokenizer` interface (or a new `ContextTokenizer` sub-interface for backward compatibility). Propagate the request context into gRPC calls while wrapping with `context.WithTimeout` for max RPC deadlines. Keep existing no-ctx methods as thin wrappers calling `context.Background()`.
  - **Impact**: Enables deadline propagation, cancellation of stuck tokenization RPCs, OTel trace continuity, and graceful shutdown of in-flight requests. Reduces resource waste from orphaned tokenization work.
  - **Rough effort**: ~1 day interface change + UDS implementation; ~0.5 day for callers and compatibility wrappers; ~0.5 day tests. Total: ~2 days, low risk.  <!-- fp: llm-d/llm-d-kv-cache:feature:context-aware-tokenizer-api -->

## Recently Resolved

NONE
