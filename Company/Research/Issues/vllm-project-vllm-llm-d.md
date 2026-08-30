---
repo: vllm-project/vllm
last_updated: 2026-08-30
---
# Backlog — vllm-project/vllm (llm-d lens)

## Direct llm-d impact

- **Scheduler pads a not-yet-prefilled PD request for spec decode; a later Mamba align-split truncates the window without cancelling placeholders, crashing the decode worker** · High · Confirmed · `issue #54392`; `vllm/v1/core/sched/scheduler.py:966` (pad condition lacks prefill-complete guard), `:1005` (align-split clips), `:1174` (placeholders emitted unconditionally) — Add `and num_computed_tokens >= request.num_prompt_tokens` to the pad condition at scheduler.py:966-971; add a post-limiter invariant (cancel padding if final width != padded width); add the #54392 regression test (hybrid Mamba align, block_size=384, K=7, PD-admitted 380-token request with external_computed=379).  <!-- fp: vllm-project/vllm:bug:pd-mamba-spec-pad-window-truncation -->
  - issue: https://github.com/vllm-project/vllm/issues/54392
  - code: https://github.com/vllm-project/vllm/blob/1dc464d42681d22f38caf1fdc1eb632dc4421c45/vllm/v1/core/sched/scheduler.py#L966-L971
  - code: https://github.com/vllm-project/vllm/blob/1dc464d42681d22f38caf1fdc1eb632dc4421c45/vllm/v1/core/sched/scheduler.py#L1005-L1010
  - code: https://github.com/vllm-project/vllm/blob/1dc464d42681d22f38caf1fdc1eb632dc4421c45/vllm/v1/core/sched/scheduler.py#L1174-L1177
  - llm_d_rationale: Failing path is KV-connector (decode) admission in disaggregated P/D with MooncakeConnectorV1 — the llm-d-shaped topology from #54392.

- **NIXL hybrid SSM disagg silently requires VLLM_SSM_CONV_STATE_LAYOUT=DS; undocumented and connector HMA compatibility not visible** · Medium · Confirmed · `issue #54387` (1P1D NIXL hybrid bring-up on GB10); `#42882` (proposes DS as default) — Drive #42882 (DS as default) or land the #54387 docs PR (DS footnote in NixlConnector matrix + cross-connector HMA note in disagg_prefill.md) as interim.  <!-- fp: vllm-project/vllm:issue:nixl-hybrid-ds-conv-layout-docs -->
  - issue: https://github.com/vllm-project/vllm/issues/54387
  - issue: https://github.com/vllm-project/vllm/issues/42882
  - llm_d_rationale: NixlConnector is the KV-transfer path llm-d builds on; the silent env-var gate blocks hybrid disagg bring-up for llm-d-shaped deployments.

## Likely llm-d impact

- **RFC #54363 — filesystem KV offload tier has no block checksums, no I/O deadline, no stale-temp cleanup, no errno retry** · High · Confirmed · `issue #54363` — Review the RFC and signal which of the four items to merge first; item 1 (checksum) and item 2 (deadline) are High priority. Item 3 (stale-temp cleanup) needs no `csrc` change and is independently shippable.  <!-- fp: vllm-project/vllm:feature:kv-offload-fs-tiering-integrity-deadline -->
  - issue: https://github.com/vllm-project/vllm/issues/54363
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/kv_offload/tiering/fs/io.py#L89
  - llm_d_rationale: Dependency surface (b) — touches the KV connector API and KV transfer paths (filesystem secondary tier) and connector lifecycle/failure modes that the shared `TieringOffloadingManager` mediates for all secondary tiers including the P2P path llm-d uses for disaggregated prefill/decode.
  - Expanded: **Problem** — FS KV offload tier lacks block checksums, I/O deadlines, stale-temp cleanup, and errno-based retry; silent corruption and unbounded stalls. **Proposed approach** — four independently-mergeable changes to `io.py` + `csrc/fs_io.cpp`: (1) CRC32C checksums with on-disk format versioning via `FileMapper._compute_base_path`; (2) monotonic `JobState` deadline with terminal-flag reconciliation for the reaper-vs-`task_done` race; (3) stale `*.tmp` cleanup by mtime threshold at startup; (4) `errno` transient/terminal classification with bounded retry. Default-off config knobs keep each change inert until validated. **Impact** — adds the integrity and liveness signals the KV offload tier currently lacks entirely; unblocks llm-d P2P path reliability via the shared manager. **Rough effort** — items 1-2 High priority; item 3 independently shippable (no `csrc` change); each PR adds unit tests for both I/O paths using the `test_fs_tier.py` pattern.

- **Bug #54360 — enabling speculative decoding silently zeroes prefix-cache hits for hybrid GDN models (regression from v0.24.0)** · High · Likely · `issue #54360` — Bisect between v0.24.0 and current main for the `prefix_cache_hits_total → 0` regression with `--speculative-config mtp` on a hybrid GDN model; restore coexistence or emit a startup warning when spec-decode suppresses APC.  <!-- fp: vllm-project/vllm:issue:spec-decode-disables-prefix-cache-hybrid-gdn -->
  - issue: https://github.com/vllm-project/vllm/issues/54360
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/core/kv_cache_utils.py#L335
  - llm_d_rationale: Dependency surface (b) — prefix-cache reuse and the cache-locality signals used for routing are core llm-d scheduler surfaces; a regression that silently disables prefix caching degrades the prefix-aware routing signal llm-d relies on.

- **Filesystem KV offload stores and loads blocks with no content integrity check — bit rot or partial rewrites produce wrong logits silently** · High · Confirmed · `vllm/v1/kv_offload/tiering/fs/io.py:152` (`_load_block` — only short-read detection, no checksum), `vllm/v1/kv_offload/tiering/fs/io.py:89` (`_store_block` — writes raw block bytes, no checksum persisted) — Implement the block-level CRC32C checksum described in #54363 item 1 — compute on store, verify on load, delete + mark-miss on mismatch. Add a `FileMapper.fields` entry so the on-disk format change moves checksummed caches to a new directory, and a `vllm:kv_offload_tiering_checksum_failures` metric.  <!-- fp: vllm-project/vllm:bug:kv-offload-fs-no-block-checksum -->
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/kv_offload/tiering/fs/io.py#L123
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/kv_offload/tiering/fs/io.py#L89
  - code: https://github.com/vllm-project/vllm/blob/1dc464d42681d22f38caf1fdc1eb632dc4421c45/vllm/v1/kv_offload/tiering/fs/io.py#L152-L162
  - llm_d_rationale: Dependency surface (b) — KV cache offload and the KV connector/transfer paths; silent corruption of offloaded KV blocks produces wrong logits with no error signal, a connector failure mode llm-d's P2P path routes through the same `TieringOffloadingManager`.

- **Filesystem KV offload I/O jobs have no deadline — a stalled NFS/FUSE mount blocks the entire 16+16-thread dual queue pool indefinitely** · High · Confirmed · `vllm/v1/kv_offload/tiering/fs/thread_pool.py:29` (`JobState.__init__` — no deadline field), `vllm/v1/kv_offload/tiering/fs/thread_pool.py:103` (`_worker` — calls `task()` with no timeout or cancellation check) — Add a monotonic deadline to `JobState` at enqueue time and a periodic check in `DualQueueThreadPool` that publishes expired jobs to `_finished_q`; use a terminal flag so the first publisher wins and `_inflight_jobs` is decremented exactly once. This is #54363 item 2; PR #53087 addresses the complementary `HIT_PENDING` side in the shared manager. Gate item 4 (errno retry) behind this deadline so retries cannot reintroduce the unbounded wait.  <!-- fp: vllm-project/vllm:bug:kv-offload-fs-no-io-deadline-pool-starvation -->
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/kv_offload/tiering/fs/thread_pool.py#L29
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/kv_offload/tiering/fs/thread_pool.py#L103
  - code: https://github.com/vllm-project/vllm/blob/1dc464d42681d22f38caf1fdc1eb632dc4421c45/vllm/v1/kv_offload/tiering/fs/io.py#L168-L222
  - llm_d_rationale: Dependency surface (b) — KV connector lifecycle and failure modes; the same `TieringOffloadingManager` mediates all secondary tiers including the P2P path llm-d uses, and a stalled filesystem tier blocks the scheduler's lookup loop until the client timeout.

- **`TieringOffloadingManager.lookup()` returns HIT_PENDING with no deadline — a stalled write defers requests until the client timeout** · High · Confirmed · `vllm/v1/kv_offload/tiering/manager.py:230` (unconditional HIT_PENDING, no deadline, no downgrade to MISS); `issue #49829`; open PR #53087 — Verify PR #53087 lands and covers the shared manager (not just the P2P-specific path); the fix should add a configurable per-`(req_id, key)` deadline that downgrades to `MISS` on expiry so the scheduler recomputes rather than deferring indefinitely.  <!-- fp: vllm-project/vllm:bug:tiering-manager-hit-pending-no-deadline -->
  - issue: https://github.com/vllm-project/vllm/issues/49829
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/kv_offload/tiering/manager.py#L230
  - code: https://github.com/vllm-project/vllm/pull/53087
  - llm_d_rationale: Dependency surface (b) — the shared `TieringOffloadingManager` mediates all secondary-tier lookups including the P2P path llm-d uses for disaggregated prefill/decode; a stalled write in any tier defers requests indefinitely, a connector lifecycle failure mode.

- **Filesystem KV offload dual-queue pool stalls in both directions when a single I/O blocks — ~32 stuck ops halt all tiering traffic** · High · Confirmed · `vllm/v1/kv_offload/tiering/fs/thread_pool.py:103` (hot path: `DualQueueThreadPool._worker` → `task()` — per-block `os.write`/`os.readv` runs with no deadline; read-priority threads fall back to the store queue and vice versa, so a stalled read drains store capacity too) — Add a per-job monotonic deadline to `JobState` and a periodic check in `get_finished()` that publishes expired jobs as failed so the scheduler stops waiting; the thread stays stuck until the filesystem returns, but the scheduler is unblocked promptly. This is #54363 item 2.  <!-- fp: vllm-project/vllm:perf:kv-offload-fs-pool-starvation-no-deadline -->
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/kv_offload/tiering/fs/thread_pool.py#L103
  - llm_d_rationale: Dependency surface (b) — KV connector lifecycle and failure modes under load; the shared `TieringOffloadingManager` mediates all secondary-tier transfers including the P2P path llm-d uses for disaggregated prefill/decode.

- **Feature #54354 — no way to budget KV cache per-GPU when a DP rank shares a card with another process** · Medium · Confirmed · `issue #54354` — Implement an absolute per-GPU memory budget or per-rank KV sizing derived from the free memory each worker already measures at startup (`init_snapshot.free_memory`), rather than a uniform fraction of total device memory; plumb it through `request_memory`. Fix #49224 (V2 runner skips CUDA-graph reservation) so CUDA-graph memory is reserved before KV sizing.  <!-- fp: vllm-project/vllm:feature:per-gpu-kv-cache-budget -->
  - issue: https://github.com/vllm-project/vllm/issues/54354
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/worker/gpu_worker.py#L270
  - llm_d_rationale: Dependency surface (b) — KV cache usage is a scheduler-facing stat the llm-d inference scheduler consumes, and DP topology is a llm-d surface; the inability to size KV per-rank in a shared-GPU deployment shapes multi-pod capacity planning.
  - Expanded: **Problem** — `--gpu-memory-utilization` applies one fraction of total device memory to every DP rank, so a shared card caps the whole group (DP=3 with an embedding server on GPU 0 → ~0.15 GiB free, OOM under load) while unshared cards strand ~17 GiB each; `--kv-cache-memory-bytes` is absolute but still one value for all ranks. **Proposed approach** — per-rank/per-GPU absolute memory budget option, or per-rank KV sizing from the free memory each worker already measures; fix #49224 so CUDA-graph memory is reserved before KV sizing; coordinate with `VLLM_ENABLE_STARTUP_PLAN` per-rank persistence. **Impact** — unblocks correct KV sizing in shared-GPU/multi-tenant deployments (also HAMi/MIG/MPS #40937) and shapes multi-pod capacity planning for llm-d. **Rough effort** — Medium: new CLI option + per-rank plumbing through `request_memory` + tests; #49224 fix is a prerequisite.

- **First boot of a new (max-num-batched-tokens, max-num-seqs) shape gets ~5% smaller KV cache than subsequent identical boots — cold compile cache inflates profiled non-KV memory** · Medium · Confirmed · `issue #54383`; `vllm/v1/worker/gpu_worker.py:554` (`memory_profiling` wraps `profile_run`), `:607` (KV budget = requested − non_kv_cache_memory) — Separate compilation from the KV-budget profile pass (profile after compile artifacts are resolved, or exclude inductor/autotune transient peaks from `non_kv_cache_memory`), or require a warm compile cache before sizing; confirm with the reporter's `replay-sim` harness.  <!-- fp: vllm-project/vllm:perf:first-boot-kv-cache-sizing-compile-cache -->
  - issue: https://github.com/vllm-project/vllm/issues/54383
  - code: https://github.com/vllm-project/vllm/blob/1dc464d42681d22f38caf1fdc1eb632dc4421c45/vllm/v1/worker/gpu_worker.py#L554-L558
  - code: https://github.com/vllm-project/vllm/blob/1dc464d42681d22f38caf1fdc1eb632dc4421c45/vllm/v1/worker/gpu_worker.py#L607-L609
  - llm_d_rationale: Sized num_gpu_blocks/KV usage is the capacity the llm-d inference scheduler reads from /metrics; cold per-pod compile caches make capacity pod-history-dependent.

## Possible llm-d impact

- **Perf #54369 — unsupported ROCm attention backends fall back to rebuilding attention metadata per draft step, capping useful MTP depth at k=4** · Medium · Confirmed · `issue #54369` — Investigate whether the sparse-indexer attention metadata for `ROCM_AITER_MLA_SPARSE`/`KPOOL_TAIL` can be built once and updated in-place across draft steps (the `supports_draft_decode_metadata_update` contract), or document the structural limitation.  <!-- fp: vllm-project/vllm:issue:rocm-attention-metadata-rebuild-per-draft-step -->
  - issue: https://github.com/vllm-project/vllm/issues/54369
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/worker/gpu/spec_decode/autoregressive/speculator.py#L74
  - llm_d_rationale: Dependency surface (b) — touches the decode hot path and speculative decoding, part of the disaggregated prefill/decode surface, but the issue is ROCm-backend-specific and the llm-d blast radius is indirect.

- **Stale `*.tmp` files from crashed engines are never reclaimed on shared PVCs — unbounded growth on long-lived shared filesystems** · Medium · Confirmed · `vllm/v1/kv_offload/tiering/fs/io.py:97` (`_store_block` — writes to `dest_path + _get_tmp_suffix()` then `os.replace`; if the engine dies between write and replace the temp file survives with nothing that will ever remove it) — Delete only temp files whose mtime is older than a conservative threshold (default 1 hour) at startup, per #54363 item 3; safe because no in-flight write can plausibly hold a temp file that long, and it does not require encoding instance identity in the suffix.  <!-- fp: vllm-project/vllm:bug:kv-offload-fs-stale-temp-files-never-reclaimed -->
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/kv_offload/tiering/fs/io.py#L97
  - llm_d_rationale: Dependency surface (b) — KV cache offload on shared/networked filesystems (NFS, POSIX PVCs) in Kubernetes, the same deployment shape llm-d targets, though the specific filesystem tier may not be the primary KV transfer path llm-d uses.

- **Unsupported attention backends rebuild metadata per draft step — per-step overhead caps useful MTP depth at k=4 even when draft quality extends further** · Medium · Confirmed · `vllm/v1/worker/gpu/spec_decode/autoregressive/speculator.py:74` (hot path: `_configure_fused_multi_step_decode` → checks `attn_group.supports_draft_decode_metadata_update`; when unsupported, `use_fused_multi_step_decode = False` and `_multi_step_decode` rebuilds `slot_mappings` + `attn_metadata` per step at `speculator.py:195`) — Implement `supports_draft_decode_metadata_update` / `update_draft_decode_metadata` for the `ROCM_AITER_MLA_SPARSE`, `DEEPSEEK_V32_INDEXER`, and `KPOOL_TAIL` backends, or hoist the metadata build out of the per-step loop if the sparse-indexer metadata can be built once and updated in-place.  <!-- fp: vllm-project/vllm:perf:spec-decode-attention-metadata-rebuild-per-draft-step -->
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/worker/gpu/spec_decode/autoregressive/speculator.py#L74
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/v1/worker/gpu/spec_decode/autoregressive/speculator.py#L195
  - llm_d_rationale: Dependency surface (b) — the decode hot path and speculative decoding, part of the disaggregated prefill/decode surface; the issue is ROCm-backend-specific and the llm-d blast radius is indirect (llm-d would benefit if MTP depth were uncapped on its target hardware).

- **RFC #54333 — reduced sampling for tensor-parallel decoding avoids O(B×V) NCCL gather when only argmax or top-k is needed** · Low · Likely · `issue #54333` — Track the RFC feedback period (through Sep 12) and the prototype PR; if the maintainers accept the design, validate the reduced-path correctness guarantees (greedy exact-match, top-k tie handling) and measure NCCL traffic reduction on TP=2+ before broader enablement.  <!-- fp: vllm-project/vllm:feature:reduced-sampling-tensor-parallel -->
  - issue: https://github.com/vllm-project/vllm/issues/54333
  - code: https://github.com/vllm-project/vllm/pull/54332
  - llm_d_rationale: Dependency surface (b) — multi-node and TP/PP/DP topology; reduced NCCL traffic per sampling step benefits multi-replica serving with TP, but the feature is opt-in and the llm-d blast radius is indirect.

## Other vLLM findings

- **XPU AWQ MoE fallback compares CUDA `device_capability` which is always -1 on XPU — blocks all AWQ MoE models on Intel GPU** · Medium · Confirmed · `vllm/model_executor/layers/quantization/moe_wna16.py` (`get_device_capability()` returns `None` on XPU → `device_capability = -1` → `if device_capability < awq_min_capability` always true); `issue #54350` — Bypass the CUDA capability check for XPU in both occurrences in `moe_wna16.py` (the `is_moe_wna16_compatible()` function and the `fused_experts` dispatch), as the reporter's verified patch shows.  <!-- fp: vllm-project/vllm:bug:xpu-awq-moe-capability-check-always-minus-one -->
  - issue: https://github.com/vllm-project/vllm/issues/54350
  - code: https://github.com/vllm-project/vllm/blob/6cddad414ee46796f21aaf7b8643a6e7a00c09b5/vllm/model_executor/layers/quantization/moe_wna16.py#L1
  - llm_d_rationale: XPU (Intel GPU) platform — llm-d targets NVIDIA/AMD; no plausible llm-d connection.

- **cache_salt length is bounded only at the HTTP protocol layer; non-HTTP request construction reads it unbounded** · Low · Likely · `vllm/v1/core/kv_cache_utils.py:606` (consumed with no length check); `vllm/entrypoints/openai/chat_completion/protocol.py:467` (`max_length=1024` added only here, commit 1dc464d4) — Move the `max_length` bound to the Request-construction layer (or `generate_block_hash_extra_keys`) so all entrypoints share it.  <!-- fp: vllm-project/vllm:bug:cache-salt-unbounded-non-http-entrypoints -->
  - code: https://github.com/vllm-project/vllm/blob/1dc464d42681d22f38caf1fdc1eb632dc4421c45/vllm/v1/core/kv_cache_utils.py#L606-L608
  - code: https://github.com/vllm-project/vllm/blob/1dc464d42681d22f38caf1fdc1eb632dc4421c45/vllm/entrypoints/openai/chat_completion/protocol.py#L467-L470
  - code: https://github.com/vllm-project/vllm/commit/1dc464d42681d22f38caf1fdc1eb632dc4421c45
  - llm_d_rationale: llm-d reaches vLLM via the OpenAI-compatible /v1 HTTP endpoint, already bounded at 1024; the residual unbounded path is the in-process LLM API llm-d does not use.

## Recently Resolved

_(none)_