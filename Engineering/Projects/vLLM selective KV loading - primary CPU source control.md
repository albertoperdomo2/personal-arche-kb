---
title: vLLM selective KV loading - primary CPU source control
date: 2026-09-03
type: implementation-guide
status: proposed
topic: KV cache offloading
repo: vllm-project/vllm
inspected_commit: 0e3ac4907d21e77cb4781338c49fef17bfea8f2b
depends_on: vllm-project/vllm#48123
companion: llm-d-router selective KV loading - policy and request wiring
---

# vLLM selective KV loading — primary CPU source control

## Purpose

This is a technical implementation guide for completing selective KV loading in
vLLM after [PR #48123](https://github.com/vllm-project/vllm/pull/48123).
That PR introduced the per-request
`kv_transfer_params.kv_load_tiers` contract and applies it to secondary
offload tiers. The missing behavior is control over loading from the primary CPU
tier.

The companion router design is
[[llm-d-router selective KV loading - policy and request wiring]]. The intended
responsibility split is:

- llm-d-router decides whether reuse is worthwhile for the selected endpoint and
  sends a request-scoped policy.
- vLLM is authoritative for enforcing that policy against its local cache state
  and preserving transfer correctness.
- GPU prefix-cache lookup remains internal to vLLM and outside
  `kv_load_tiers`. The field governs external/offloaded sources only.

This guide separates a safe binary MVP from full source-selective semantics.
They should not be collapsed into one change unless promotion provenance is
implemented and tested at the same time.

## Status and inspected baseline

The design was verified against
`vllm-project/vllm@0e3ac4907d21e77cb4781338c49fef17bfea8f2b`
on 2026-09-03.

Duplicate-work checks found no open vLLM PR implementing primary-tier selective
loading. [PR #50087](https://github.com/vllm-project/vllm/pull/50087) concerns
per-request store strategy, not load selection. Repeat the issue and open-PR
searches immediately before starting a contribution, as required by the vLLM
contribution policy.

## Problem statement

Today the request can restrict secondary-tier lookup, but it cannot prevent a
CPU hit from being loaded to GPU:

1. The connector parses `kv_load_tiers` into a request
   [`TierFilter`](https://github.com/vllm-project/vllm/blob/0e3ac4907d21e77cb4781338c49fef17bfea8f2b/vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py#L431-L486).
2. [`TieringOffloadingManager.lookup`](https://github.com/vllm-project/vllm/blob/0e3ac4907d21e77cb4781338c49fef17bfea8f2b/vllm/v1/kv_offload/tiering/manager.py#L337-L418)
   always checks the CPU primary tier first.
3. The filter is consulted only while iterating secondary tiers.
4. Standalone
   [`CPUOffloadingManager.lookup`](https://github.com/vllm-project/vllm/blob/0e3ac4907d21e77cb4781338c49fef17bfea8f2b/vllm/v1/kv_offload/cpu/manager.py#L133-L140)
   ignores the request filter entirely.

Consequently, a router decision such as “recompute instead of using CPU” cannot
currently be enforced. An empty list disables secondary lookup but still permits
CPU reuse. With standalone `CPUOffloadingSpec`, every value of
`kv_load_tiers` is effectively ignored.

This is a load-path feature. It must not alter storage, eviction, cascading, or
retention. “Lazy offload” — postponing GPU-to-CPU storage until GPU pressure —
is a different contribution.

## Existing contract

### Request syntax

The existing request field is:

```json
{
  "kv_transfer_params": {
    "kv_load_tiers": [
      {"medium": "CPU", "locality": "LOCAL"},
      {"medium": "STORAGE", "locality": "LOCAL"}
    ]
  }
}
```

The parser accepts medium and locality case-insensitively because values are
uppercased before conversion. `TierMatcher` treats an omitted field as a
wildcard. The meaningful input states are:

| Input | Current interpretation | Required interpretation |
|---|---|---|
| field omitted | all offload tiers | unchanged |
| `null` | all offload tiers | unchanged unless separately deprecated |
| `[]` | no secondary tiers, CPU still used | no offloaded load at all |
| CPU matcher | CPU plus any matching secondary | CPU as an original source only |
| STORAGE matcher | matching storage secondary, CPU still used | storage as original source; CPU may be an internal staging tier |
| malformed value | warning and all tiers | keep for compatibility in this contribution |

Do not mix strict validation into the first selective-loading PR. Existing
malformed-input behavior is fail-open to `TierFilter.ALL`; changing it is a
separate API-compatibility decision.

### Source tier versus transport tier

Filters must describe the source selected for reuse, not every medium traversed
by the transfer.

A storage hit necessarily follows:

```text
storage secondary -> CPU staging slot -> GPU
```

Therefore a request permitting STORAGE but excluding CPU must still be allowed
to use the CPU buffer as an implementation detail. It must not reuse an
independently resident CPU copy whose original source is CPU.

This distinction is the central invariant for the full implementation.

## Why a direct primary filter is unsafe

The tempting patch is:

```python
if req_ctx.tier_filter.matches(primary_tier):
    result = primary_tier.lookup(key, req_ctx)
```

That is insufficient. On a secondary hit, the tiering manager immediately
reserves a CPU slot and starts a secondary-to-CPU promotion. Subsequent scheduler
steps observe that slot as a primary `HIT_PENDING`. The request must keep
polling it until the promotion completes even if CPU was not one of the allowed
original sources.

A plain primary filter cannot distinguish:

- a CPU-resident reusable block;
- an in-flight GPU-to-CPU store;
- a storage-to-CPU promotion initiated for this request;
- a promotion initiated by another compatible request;
- a promotion initiated by a request whose policy is incompatible with the
  current request.

The current primary `HIT_PENDING` result carries none of that provenance.
Filtering it blindly can either strand a valid promotion or let an excluded
request wait on and consume it.

## Recommended delivery plan

### Phase 1 — binary selective loading

Implement one narrowly defined semantic first:

> `kv_load_tiers=[]` disables lookup and loading from every offload tier,
> including the CPU primary tier.

All non-empty filters retain the behavior delivered by PR #48123: CPU is
consulted unconditionally and secondary tiers are filtered. This immediately
supports the router’s most important decision — “do not load; recompute” —
without changing promotion machinery.

#### Phase 1 lookup rule

At the manager boundary, before any lookup with side effects:

```text
if the filter was explicitly supplied and has zero matchers:
    return MISS
otherwise:
    execute the existing lookup path
```

The distinction between “field omitted” and “explicit empty list” already exists:
omitted becomes `TierFilter.ALL`; an empty list becomes a filter with no
matchers. Add a named predicate such as `allows_no_tiers` or `is_empty`
rather than reaching into the matcher collection from managers.

Apply the rule to both:

- `TieringOffloadingManager`;
- standalone `CPUOffloadingManager`.

The check must precede job polling, CPU allocation, async secondary lookup, or
promotion initiation attributable to this request. Manager-wide completion
processing may still run as part of the scheduler lifecycle; an opted-out
request must simply cause no new retrieval work.

#### Phase 1 file plan

| File | Change |
|---|---|
| `vllm/v1/kv_offload/base.py` | Add an explicit, read-only empty-filter predicate and document omitted versus empty semantics. |
| `vllm/v1/kv_offload/tiering/manager.py` | Return `MISS` for an explicitly empty filter before primary or secondary lookup. |
| `vllm/v1/kv_offload/cpu/manager.py` | Enforce the same behavior for `CPUOffloadingSpec`. |
| `tests/v1/kv_offload/tiering/test_tiering_offloading.py` | Replace the “primary unaffected” expectation for `[]`; prove no promotion or async lookup occurs. |
| `tests/v1/kv_offload/cpu/test_manager.py` | Add standalone CPU opt-out behavior. |
| connector scheduler tests | Preserve parser coverage for omitted, empty, wildcard, and malformed values. |
| user documentation | State that `[]` means recompute rather than load from offload tiers. |

Avoid adding the policy to the connector scheduler only. Manager-level
enforcement keeps the contract true for every caller and both offloading specs.

### Phase 2 — full source-selective primary control

Phase 2 gives every matcher its literal source-selection meaning. It requires
promotion provenance.

#### Required primary-tier identity

Give the CPU primary tier explicit metadata equivalent to:

```python
TierMatcher(medium=Medium.CPU, locality=Locality.LOCAL)
```

Do not infer CPU identity from the class name in lookup code. Use the same
metadata vocabulary as secondary tiers so matching behavior stays centralized.

#### Promotion record

Introduce manager-owned state keyed by `OffloadKey`. A conceptual record is:

```python
@dataclass
class PromotionRecord:
    source_tier: TierMatcher
    state: PromotionState
    owner_request_ids: set[str]
```

The exact representation should match vLLM’s allocation-conscious style; the
semantics matter more than the dataclass. At minimum the manager must know:

- which secondary tier supplied the block;
- whether the promotion is queued, submitted, complete, or failed;
- which requests are authorized to join/wait for it;
- enough information to remove the record on every terminal path.

Do not use only the request id that first triggered the promotion. Shared-prefix
requests must be able to join the same compatible in-flight work.

#### Full lookup algorithm

For each key:

```text
1. Process finished tier jobs using the existing once-per-step gate.

2. If a promotion record exists:
   a. If this request's filter matches the promotion's source tier:
      - queued/submitted: return HIT_PENDING or RETRY according to the existing
        scheduler contract;
      - complete: continue through the CPU primary load path.
   b. If it does not match:
      - do not wait on or claim that promotion;
      - continue looking only at sources allowed to this request, or return MISS.

3. If no compatible promotion governs the key and CPU is an allowed original
   source:
   - query the primary tier;
   - HIT means reusable CPU source;
   - GPU-to-CPU store HIT_PENDING may be awaited only under the chosen CPU
     semantics.

4. Query matching secondary tiers in configured priority order.

5. On a secondary HIT:
   - reserve CPU staging;
   - create a promotion record carrying that source tier;
   - batch submission exactly as today;
   - return RETRY.

6. If no allowed source hits, return MISS and let vLLM recompute.
```

A completed storage promotion creates an ordinary CPU-resident copy. For the
request that authorized the storage source, it remains usable to finish that
retrieval. For a later request, it is an original CPU source and therefore
requires CPU permission. This implies either a short-lived authorization record
until all waiters have crossed into `prepare_load`, or another explicit
handoff mechanism. Do not erase provenance at `complete_write` before waiting
requests can prove authorization.

#### Concurrency cases that define correctness

| Scenario | Required outcome |
|---|---|
| A allows STORAGE and initiates promotion; B also allows STORAGE | B joins the same promotion; only one secondary load is submitted. |
| A allows STORAGE; B sends `[]` | B does not wait and recomputes. |
| A allows STORAGE; B allows CPU only | B must not consume the in-flight storage promotion merely because its staging slot is in CPU. |
| A allows CPU; block has an unrelated GPU-to-CPU store pending | Behavior follows the documented CPU-source policy; no secondary work is duplicated. |
| A excludes CPU but allows STORAGE | A can use CPU as staging for its authorized storage promotion. |
| Promotion fails or is cancelled | All reservations, waiter references, and provenance are released; later lookup can retry or recompute. |
| CPU slot is unavailable | Preserve current progress guarantee: return MISS rather than retry forever. |
| Request is aborted while sharing a promotion | Other authorized waiters continue; request-owned state is removed. |
| Reset/all-blocks-cleared | Promotion and authorization maps are emptied atomically with existing pending state. |

#### State ownership and lifecycle

Keep promotion truth in `TieringOffloadingManager`, because it coordinates the
source tier and the CPU staging tier. Do not push source provenance into
`CPUOffloadingManager`; that manager also serves standalone CPU offloading and
cannot know why a slot was populated.

Audit cleanup at these boundaries:

- promotion submission failure;
- tier job success and failure;
- `prepare_load` handoff;
- request finish and abort;
- CPU eviction;
- manager reset;
- all-blocks-cleared;
- duplicate/shared-key lookup.

The current `_jobs` and `_pending_load_submissions` collections are natural
integration points, but a request-indexed pending batch alone is not sufficient
for cross-request authorization.

## Request-context contract

Keep the public field in `kv_transfer_params`; do not add a router-specific
connector API. The context should remain immutable from lookup’s perspective.

A future implementation may introduce a separate optimization hint such as
“known absent from CPU” to skip an initial primary probe. That is not equivalent
to source exclusion:

- a skip hint describes router knowledge at one instant;
- `kv_load_tiers` is an authorization/policy decision for the request;
- after a permitted storage promotion starts, the engine must poll its staging
  state regardless of the initial hint.

Do not overload one meaning onto the other without an explicit API decision.

## Tests

### Phase 1 unit matrix

Extend existing suites rather than creating a disconnected test file.

1. Omitted field: CPU and secondary lookup behave exactly as before.
2. `null`: preserves current all-tier behavior.
3. Empty list with a ready CPU block: `MISS`; no `prepare_load`.
4. Empty list with CPU `HIT_PENDING`: `MISS`; request is not deferred.
5. Empty list with a secondary hit: no secondary lookup and no promotion.
6. Empty list with standalone `CPUOffloadingSpec`: `MISS`.
7. Non-empty STORAGE filter: current storage promotion still completes.
8. Malformed filter: preserves the current warning/fail-open behavior.
9. Store path: unchanged under every load filter.

### Phase 2 unit matrix

Add focused tests for every concurrency row above, plus:

- wildcard medium/locality matching;
- LOCAL versus REMOTE source selection;
- multiple secondary tiers with first allowed hit;
- completed promotion authorization handoff;
- cancellation before and after submission;
- failure cleanup and CPU-slot reuse;
- no duplicate I/O for shared block hashes;
- no leaked references after reset.

Assert observable outcomes: lookup result, submitted jobs, selected source,
allocated/free CPU slots, and absence of duplicate submissions. Avoid tests that
only assert private collection shapes.

### Connector-level test

Exercise an actual request context through parsing and manager lookup for:

```text
omitted -> all
[] -> none
[CPU] -> CPU source
[STORAGE] -> storage source with CPU staging
```

This catches wiring errors that manager-only tests cannot.

### Suggested commands

Use the project virtual environment, never system Python:

```bash
.venv/bin/python -m pytest   tests/v1/kv_offload/cpu/test_manager.py   tests/v1/kv_offload/tiering/test_tiering_offloading.py -v

pre-commit run ruff-check --all-files
```

Run the smallest existing connector integration suite that covers offloading
metadata when the environment and gated model access are available. Record any
environmental skip or token requirement in the PR.

## Observability

Add metrics only if they answer rollout questions that current transfer metrics
cannot. Useful bounded dimensions are:

- load policy: default / disabled / filtered;
- lookup outcome: CPU / secondary / recompute;
- promotion joined versus initiated;
- denied resident hit by source medium;
- promotion failure reason.

Do not label metrics with request ids, block hashes, model names supplied by
clients, or raw matcher JSON. Trace attributes may carry more request-specific
detail if existing privacy/cardinality conventions allow it.

Phase 1 can reasonably ship with logs and existing metrics. Phase 2 benefits
from counters for denied hits and joined promotions because those expose policy
effect and concurrency mistakes.

## Compatibility and rollout

1. Preserve omitted-field behavior exactly.
2. Land vLLM Phase 1 before enabling the router to emit `[]`.
3. Gate router emission on a known-compatible vLLM deployment; older/current
   engines interpret `[]` as “no secondary” while still loading CPU.
4. Keep the router feature disabled by default for the first rollout.
5. Compare TTFT, recompute tokens, CPU-to-GPU bytes, secondary bytes, and cache
   hit source before increasing scope.
6. Implement Phase 2 only after agreeing on CPU-source and promotion
   authorization semantics.
7. Document rollback: stop emitting the field. No engine restart or cache
   migration should be required.

## Security and robustness

Treat `kv_load_tiers` as an untrusted request value at the public API boundary.
The parser already bounds the schema to enums; keep that property. The change
must not permit arbitrary tier names, paths, endpoints, or connector parameters.

A disabled load must mean recomputation, not request failure. Failure to perform
an allowed promotion must preserve the current progress behavior and fall back
to recomputation rather than indefinite scheduler deferral.

## Non-goals

- Lazy GPU-to-CPU offload under GPU pressure.
- Selective storage/cascade policy.
- Changing CPU eviction policy.
- Choosing the economically best source inside vLLM.
- Router-side calibration or pressure sensing.
- Strict rejection of malformed `kv_load_tiers`.
- Eliminating CPU as the physical gateway for secondary tiers.

## PR decomposition

### PR A — binary opt-out

- Define explicit-empty semantics.
- Enforce them in tiered and standalone CPU managers.
- Update focused tests and user-facing contract.
- No promotion-state refactor.

### PR B — promotion provenance

- Add source-aware promotion state and lifecycle cleanup.
- Preserve shared-promotion deduplication.
- Initially keep behavior unchanged behind internal plumbing.
- Add concurrency/failure tests.

### PR C — full primary source filtering

- Give the primary tier canonical CPU/LOCAL identity.
- Apply filters using promotion provenance.
- Add full matrix and connector-level coverage.
- Enable matching router policy modes after compatibility gating.

This split keeps each review defensible and makes regression bisection practical.

## Acceptance criteria

Phase 1 is complete when:

- `[]` prevents every offload lookup/load in both offloading specs;
- omitted and non-empty behavior is unchanged;
- no new promotion, pending lookup, or request deferral is caused by an opted-out
  request;
- focused tests and lint pass;
- the API meaning is documented.

Full selective loading is complete when:

- each matcher controls original cache sources consistently;
- storage can still stage through CPU when CPU is excluded as a source;
- incompatible requests cannot consume or wait on one another’s promotions;
- compatible requests deduplicate promotion I/O;
- all terminal paths clean up state;
- the router can send CPU, STORAGE, combined, or empty policies with predictable
  results.

## Open decisions

1. Is CPU exclusion a strict authorization or merely a cost preference?
   This guide assumes strict per-request authorization.
2. May a request that permits CPU wait for a GPU-to-CPU store already in flight,
   or should only ready CPU blocks count?
3. How long should completed-promotion provenance survive to authorize all
   waiters without becoming stale?
4. Should malformed filters remain fail-open indefinitely?
5. Is locality meaningful for the primary tier beyond CPU/LOCAL?
6. Should a separate “known absent” hint be added for cheap probe avoidance?

Resolve the first three before Phase 2 implementation.

## Related

- [[llm-d-router selective KV loading - policy and request wiring]]
- [[vLLM KV offload retrieval path - lookup, promotion, and load]]
- [[vLLM KV offload lookup - worked example]]
- [[vLLM offloading specs - CPUOffloadingSpec vs TieringOffloadingSpec]]
- [[vLLM and llm-d-router KV cache responsibility split]]
- [[vLLM KV Events canonical form]]
- [[Activity-Based KV Cache Offloading]]
- [[01 - Calibration Protocol]]

## Provenance

Direct inspection of the local vLLM checkout at
`0e3ac4907d21e77cb4781338c49fef17bfea8f2b`, PR #48123 and its review
discussion, open-PR duplicate searches, and the existing CPU/tiering test suites
on 2026-09-03. This is a proposed design, not a description of functionality
already present.