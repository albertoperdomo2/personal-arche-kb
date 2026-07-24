---
repo: llm-d/llm-d-router
last_updated: 2025-07-24
---

# Backlog — llm-d/llm-d-router

## Workable Issues

- **Native generate requests bypass the multimodal encoder** · High · Likely · `issue #2108` — Add generate-format extraction to `mmItemsForFanout` + integration test  <!-- fp: llm-d/llm-d-router:issue:generate-api-skips-multimodal-encoder -->
- **Add coordinated scheduling for disaggregated inference stages** · Medium · Confirmed · `issue #2135` — Design cross-stage selection metadata; benchmark deferred vs coordinated  <!-- fp: llm-d/llm-d-router:issue:coordinated-disaggregated-stage-selection -->

## Bugs

- **Sidecar connector handlers buffer request bodies without a size limit** · Medium · Confirmed · `connector_p2p.go:42` — Add `MaxBytesReader` route boundary with configurable limit + 413  <!-- fp: llm-d/llm-d-router:bug:sidecar-unbounded-request-body -->
- **Detached prefill requests can remain blocked indefinitely** · Medium · Likely · `connector_p2p.go:124` — Derive lifecycle-bound context with configurable prefill deadline + test  <!-- fp: llm-d/llm-d-router:bug:detached-prefill-goroutine-no-deadline -->

## Performance

- **Every scorer allocates a full endpoint map on the scheduling hot path** · Low · Likely · `scheduler_profile.go:173` — Benchmark; prototype reusable per-request score accumulator  <!-- fp: llm-d/llm-d-router:perf:per-scorer-endpoint-score-maps -->

## Features & RFCs

## Recently Resolved
