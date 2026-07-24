---
repo: vllm-project/vllm
last_updated: 2026-07-24
---
# Backlog — vllm-project/vllm

## Workable Issues

- **Compile correctness test exhausts L4 memory before reaching its assertions** · Medium · Confirmed · `issue #49672` — Reproduce with a cold Inductor cache immediately before and after PR #48155, then either fix the identified memory regression or select a smaller model/context configuration that preserves the test’s compile coverage.  <!-- fp: vllm-project/vllm:issue:compile-correctness-cold-start-memory -->

- **MkDocs and markdownlint generate incompatible heading fragments** · Low · Confirmed · `issue #49662` — Confirm the preferred rendered-anchor convention, configure `toc.slugify` and the fragment validator consistently, and add a rendered-site link check covering punctuation-heavy headings.  <!-- fp: vllm-project/vllm:issue:mkdocs-markdownlint-anchor-slug-drift -->

## Bugs

- **Mooncake bootstrap startup can wait forever after a server-thread failure** · Medium · Confirmed · `vllm/distributed/kv_transfer/kv_connector/v1/mooncake/mooncake_utils.py:79` — Add a bounded startup deadline and break when the thread is no longer alive, raising an exception containing the host, port, and underlying Uvicorn startup failure; add a regression test using an occupied port.  <!-- fp: vllm-project/vllm:bug:mooncake-bootstrap-start-infinite-wait -->

## Performance

- **Ubatch metadata slicing allocates tensors outside CUDA graphs on every step** · Medium · Likely · `vllm/v1/worker/ubatch_utils.py:125` — Profile DBO decode with allocation and CUDA-graph tracing enabled, then construct sliced query-start locations in reusable runner-owned buffers and include the metadata preparation in graph capture where shapes permit.  <!-- fp: vllm-project/vllm:perf:ubatch-attention-metadata-breaks-cudagraph -->

## Features & RFCs

- **Extend the V2 pooling runner beyond normalized last-token embeddings** · Medium · Confirmed · `vllm/v1/worker/gpu/pool/pooling_runner.py:22` — Problem: V2 GPU pooling runner only supports `embed` task with last hidden state and unconditional L2 normalization; prevents raw embeddings, configurable mean/CLS pooling, and classification/reward heads. Proposed approach: Draft RFC defining pooling strategy, normalization, and output-head semantics; implement raw vs normalized last-token first, then add mean/CLS and classification/reward tasks with parity tests. Impact: Would make V2 runner useful across diverse pooling workloads, eliminating model-specific forks. Effort: 6-9 weeks total (Phase 1: 1-2 weeks, Phase 2: 2-3 weeks, Phase 3: 3-4 weeks).  <!-- fp: vllm-project/vllm:feature:v2-pooling-task-and-normalization-controls -->

## Recently Resolved