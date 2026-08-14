---
title: "Phase 1 — Queued-Request Oracle Prefetch Implementation Guide"
date: "2026-08-14"
type: "implementation-guide"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
phase: "1 — naive proactive prefetching proof of concept"
status: "ready-for-implementation"
supersedes-policy-in: "[[02 - Phase 1 Naive Prefetch Implementation Guide]]"
depends-on: "[[2026-08-14 - Phase 1 queued-request oracle prefetch plan]]"
validation: "[[2026-08-14 - Phase 1 NVMe prefetch validation]]"
codebase: "vLLM experimental/naive-proactive-prefetching @ 4614530b15"
benchmark: "guidellm-nemotron-nvme-prefetch-poc"
---

# Phase 1 — Queued-Request Oracle Prefetch Implementation Guide

## Decision

Replace the current **post-miss read-ahead policy** with an explicitly experimental **admission-time, assumed-resident NVMe→CPU prefetch**.

The old policy should be removed, not repaired for this Phase 1 proof of concept. It asks the filesystem tier whether the keys after a terminal miss exist. With prefix-chained hashes and append-like persistent storage, a stored later key normally implies that the predecessor also exists. Therefore, the keys selected after a resolved terminal miss are normally absent. Fixing asynchronous `RETRY` handling would make the lookup state machine more correct, but it would not make this candidate policy useful.

Do **not** discard the complete prefetch implementation. Retain and reuse the transfer substrate and outcome accounting.

| Component | Decision | Reason |
|---|---|---|
| Scheduler hook in `_maximal_prefix_lookup()` after `MISS` | Delete | Candidate set is structurally wrong for the filesystem tier |
| `_try_promote()` as the Phase 1 prefetch selector | Delete from the Phase 1 path | Repeats the same secondary membership lookup that normal demand already performs |
| Current public `prefetch()` semantics | Replace | Its contract is post-miss membership-driven read-ahead |
| `_initiate_promotion()` | Keep and extend minimally | Correctly reserves CPU space and queues secondary→primary work |
| `_flush_pending_promotions()` | Keep | Already batches loads per tier and request |
| Finished-job polling and `primary_tier.complete_write()` | Keep | Required async completion path |
| Primary residency check | Keep | Prevents duplicate CPU allocation; it is not secondary candidate discovery |
| Attempted/promoted/redundant/skipped/useful/wasted/untracked metrics | Keep, relabel, and refine | They remain valuable for proving the mechanism |
| Generic reactive `lookup()` | Keep unchanged | It remains the demand baseline and fallback |

This guide supersedes only the **implementation policy** in [[02 - Phase 1 Naive Prefetch Implementation Guide]]. That article and the failed validation reports remain historical records.

## Proof-of-concept contract

For a request explicitly marked with:

```json
{
  "kv_transfer_params": {
    "abc_admission_prefetch": true
  }
}
```

the connector must:

1. Build the request's normal ordered offload keys at admission.
2. Take the first `N` keys from the single full-attention KV group.
3. Check only whether each key is already in the CPU primary tier.
4. For every primary miss, directly queue a load from secondary tier 0.
5. Never call the secondary tier's `lookup()` from this prefetch path.
6. Allow the batch to run while the request waits behind the active request.
7. Let the normal demand path consume CPU hits later.

The benchmark, not the vLLM selector, guarantees that these keys exist on NVMe. A missing file is an **oracle violation** and must be visible as a load failure.

Every prefetch system needs block identities. This toy uses identities already available on the admitted request. What it intentionally avoids is membership-driven candidate discovery and demand-triggered selection.

## Configuration

Remove the experimental branch's old `prefetch_chunks` meaning and introduce:

```json
{
  "admission_prefetch_chunks": 100
}
```

Rules:

- default: `0` (disabled);
- type: non-negative integer;
- the request flag must be exactly Boolean `true`;
- for this PoC, source tier is secondary tier 0;
- if tier 0 is absent or rejected by the request's load-tier filter, count skips and do not crash;
- restrict the first implementation to exactly one full-attention group;
- for unsupported hybrid, sliding-window, Mamba, or EAGLE layouts, log once and skip the experimental path.

Do not retain `prefetch_chunks` as an alias on this experimental branch. A clean rename prevents an old deployment from silently enabling a policy with different semantics.

A production design may later replace the request flag and fixed tier with a predictor and placement policy. Those are deliberately out of scope.

## Request API and warm-up gate

The OpenAI chat protocol already accepts `kv_transfer_params`, places it in sampling `extra_args`, and carries it onto the scheduler `Request`. The offloading connector already copies it into `ReqContext`. No new HTTP endpoint or engine-wide mutable switch is needed.

This gate is essential because the GuideLLM pre-warmup runs against an initially empty NVMe tier:

- **pre-warmup requests:** omit the flag or set `abc_admission_prefetch: false`;
- **measured requests:** set `abc_admission_prefetch: true`;
- **control deployment:** set `admission_prefetch_chunks: 0`;
- **prefetch deployment:** set `admission_prefetch_chunks: 100`.

The measured GuideLLM backend body should contain:

```yaml
backend:
  kind: openai_http
  request_format: /v1/chat/completions
  stream: true
  extras:
    body:
      temperature: 0
      seed: 20260814
      kv_transfer_params:
        abc_admission_prefetch: true
```

The `pre_warmup` section must override the backend so its body carries `abc_admission_prefetch: false` while preserving the same temperature and seed. BenchFlow's pre-warmup implementation supports top-level GuideLLM argument overrides, including `backend`.

Do not use a request-count threshold. It is timing-dependent, difficult to audit, and can begin prefetching before the NVMe population phase is complete.

## vLLM changes

### 1. Spec and configuration

File: `vllm/v1/kv_offload/tiering/spec.py`

- Remove documentation, validation, and constructor plumbing for `prefetch_chunks`.
- Add validation and plumbing for `admission_prefetch_chunks`.
- Update metric documentation from "post-miss read-ahead" to "admission-time assumed-resident promotion."
- Add the two new counters described in the metrics section.

File: `vllm/v1/kv_offload/tiering/manager.py`

- Rename `_prefetch_chunks` and its property to `_admission_prefetch_chunks` and `admission_prefetch_chunks`.
- Update the startup log to state that queued-request oracle prefetch is enabled, with `N` and source tier.

### 2. Remove the failed scheduler policy

File: `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`

Delete the entire Phase 1 block from the `LookupResult.MISS` arm of `_maximal_prefix_lookup()`:

```python
n = getattr(self.manager, "prefetch_chunks", 0)
prefetch = getattr(self.manager, "prefetch", None)
...
prefetch(keys[local_idx + 1 : local_idx + 1 + n], req_context)
```

The `MISS` arm should return to simply terminating the maximal-prefix scan. Normal demand promotion behavior elsewhere remains unchanged.

### 3. Add the direct manager API

File: `vllm/v1/kv_offload/tiering/manager.py`

Replace `_try_promote()` plus the existing `prefetch()` Phase 1 path with a narrow method:

```python
def prefetch_assume_resident(
    self,
    keys: Sequence[OffloadKey],
    req_context: ReqContext,
    tier_idx: int = 0,
) -> int:
    ...
```

Required behavior for each supplied key:

1. Increment attempted using the real source-tier label, for example `1:fs`.
2. Call `primary_tier.lookup()`.
3. On `HIT` or `HIT_PENDING`, increment redundant and continue.
4. On a primary miss, call `_initiate_promotion(source_tier, key, req_context, is_prefetch=True)` directly.
5. If CPU allocation succeeds, increment promoted.
6. If it fails, increment skipped with `1:fs`.
7. Never invoke `source_tier.lookup()`.

The method returns the number of queued promotions, not the number of completed loads.

Continue to honor `ReqContext.load_tier_filter`. If the fixed source tier is filtered out, every supplied key is an attributable skip.

### 4. Trigger at request admission

File: `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py`

Extend `OffloadingConnectorScheduler.on_new_request()` after:

- `manager.on_new_request(req_context)`;
- construction of `RequestOffloadState`;
- insertion into `_req_status`.

Then:

```python
params = request.kv_transfer_params or {}
enabled = params.get("abc_admission_prefetch") is True
n = self.manager.admission_prefetch_chunks

if enabled and n > 0:
    req_status.update_offload_keys()
    # PoC guard: exactly one full-attention group.
    keys = req_status.group_states[0].offload_keys[:n]
    self.manager.prefetch_assume_resident(keys, req_context, tier_idx=0)
```

The exact ordering matters. The manager's per-request state must exist before queuing the load, and the connector state must be registered before later scheduler callbacks can observe it.

If the prompt exposes fewer than `N` complete chunks, attempt all available keys. Do not fabricate keys and do not prefetch decode-future blocks whose hashes do not yet exist.

The pending batch will be submitted by the existing `on_schedule_end()` → `_flush_pending_promotions()` path. No background thread is needed.

### 5. Distinguish prefetch jobs from demand promotions

The existing `JobMetadata.is_promotion` identifies transfer direction, but both reactive demand loads and proactive loads are promotions. Add manager-local provenance without widening the shared tier API unnecessarily:

- extend `PendingPromotion` with the keys originating from prefetch;
- pass `is_prefetch=True` into `_initiate_promotion()`;
- when a pending batch is flushed, record its prefetched keys by `job_id`;
- when the job completes, use that side table to attribute prefetch success or failure.

A single batch may contain both demand-initiated and prefetch-initiated keys, so provenance must be per key, not merely one Boolean per job.

Track a key as prefetched only once its promotion has been queued, but remove it explicitly if the load job fails. Do not let a failed NVMe read later appear only as "wasted."

### 6. Retain the reactive fallback

Do not change the normal `TieringOffloadingManager.lookup()` algorithm for this PoC. If admission prefetch is late, skipped, evicted, or failed, the existing demand path must still query secondary tiers and preserve correctness.

The known `RETRY` collapse in the old `_try_promote()` disappears with that method. It does not require a separate fix for this experiment because the new direct prefetch path deliberately performs no secondary lookup.

## Metrics

Retain:

- `prefetch_attempted`;
- `prefetch_promoted`;
- `prefetch_redundant`;
- `prefetch_skipped`;
- `prefetch_useful`;
- `prefetch_wasted`;
- `prefetch_untracked`.

For this design, every attempt has a known source tier. Use `tier="1:fs"`; eliminate the ambiguous aggregate `tier="prefetch"` label from the new path.

Add:

- **`prefetch_load_failed`** — assumed-resident chunks whose asynchronous secondary→primary load completed unsuccessfully. This is the oracle-violation counter.
- **`prefetch_late`** — prefetched keys whose first demand lookup observed `HIT_PENDING`. Count at most once per key. The key may still become useful later, but it did not finish before demand.

Accounting checks:

```text
attempted = promoted + redundant + skipped
promoted = useful + wasted + untracked + still_tracked
oracle failures are a labeled subset of promoted
late is an orthogonal subset of promoted
```

For the benchmark acceptance decision, use:

```text
effective_promoted = promoted - load_failed
ready-before-demand rate = (useful - late_and_later_useful) / effective_promoted
```

The implementation may maintain explicit per-key state to calculate the final ready-before-demand value cleanly. Avoid deriving it solely from scrape timing.

## Tests

### Scheduler unit tests

Add or replace tests in the offloading connector scheduler suite:

1. marked request, `N=100`, at least 100 keys: passes exactly the first 100 ordered keys;
2. marked request with fewer than 100 keys: passes all available keys;
3. unmarked request: no prefetch call;
4. explicitly false request flag: no prefetch call;
5. `admission_prefetch_chunks=0`: no prefetch call;
6. unsupported group layout: skip with no crash;
7. normal terminal `MISS`: does not invoke prefetch;
8. manager request state is created before the prefetch call.

### Manager unit tests

1. assumed-resident promotion does not call `secondary_tier.lookup()`;
2. 100 primary misses with capacity queue one batched load and report 100 promoted;
3. primary `HIT` and `HIT_PENDING` report redundant;
4. primary capacity failure reports `1:fs` skipped;
5. tier-filter rejection reports attributable skips;
6. successful completion followed by demand `HIT` reports useful;
7. first demand `HIT_PENDING` reports late exactly once;
8. failed load reports `prefetch_load_failed`, releases the CPU write reservation, and is not later reported only as wasted;
9. mixed demand/prefetch batch preserves per-key provenance;
10. tracking-capacity behavior still satisfies the accounting invariant.

Remove tests whose only purpose is the deleted post-miss slice or secondary-membership prefetch selector. Adapt outcome-accounting tests instead of deleting them.

### Filesystem integration test

Use a real temporary filesystem tier:

1. pre-populate known files for at least `N` keys;
2. construct a marked request with those ordered keys;
3. invoke admission prefetch;
4. flush and process the asynchronous load job;
5. assert CPU hits for the prefetched keys;
6. assert the filesystem `lookup()` method was not used for candidate discovery;
7. delete or omit one test file and assert one oracle failure without a scheduler crash.

## Benchmark execution

Use the existing BenchFlow profile:

`/Users/aperdomo/workspace/redhat/benchflow/profiles/benchmark/nemotron-nvme-prefetch-poc.yaml`

and Nemotron FP8 deployment experiment:

`/Users/aperdomo/workspace/redhat/benchflow/experiments/rhoai/cpu-offloading.yaml`

Required conditions:

- `max_num_seqs=1`;
- GuideLLM concurrent streams `8`;
- deterministic 4,096-token prompts and 64-token outputs;
- 1,024-request pre-warmup that does **not** opt into prefetch;
- measured replay with the request flag enabled;
- persistent NVMe state across pre-warmup and measurement;
- control `N=0`;
- treatment `N=100`;
- at least five paired repetitions.

The first admitted request may have little or no cover interval. This is acceptable, but target-level analysis should distinguish queue depth or prefetch lead time. The mechanism claim should be based on requests that had a real overlap opportunity, while aggregate throughput should still include the entire run.

## Acceptance gates

Mechanism:

- exactly `min(N, available_prompt_chunks)` attempts per marked request;
- nonzero `promoted{tier="1:fs"}`;
- no unexpected aggregate `tier="prefetch"` series;
- zero `prefetch_load_failed` after NVMe preparation is validated;
- useful/effective-promoted at least 0.9;
- most prefetched keys are not late;
- no material primary-capacity failures.

Performance:

- paired measured-request TTFT improves consistently;
- aggregate request throughput improves;
- output-token throughput is reported but treated as secondary for this prompt-heavy workload;
- comparison uses identical model revision, topology, seed, NVMe snapshot, and cold GPU/CPU starting state.

## Interpretation boundary

A successful result proves only:

> Given known queued-request block identities, residency guaranteed by benchmark construction, and enough queue overlap, proactive NVMe→CPU promotion can remove storage work from the request-critical path and improve TTFT or pipeline throughput.

It does not prove that `N=100` is optimal or that a production system can predict the next request. Once this mechanism is demonstrated, the admission flag and blind first-N selector become the seam for Phase 2 heuristics.