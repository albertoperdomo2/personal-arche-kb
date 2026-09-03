---
title: llm-d-router selective KV loading - policy and request wiring
date: 2026-09-03
type: implementation-guide
status: proposed
topic: KV cache routing
repo: llm-d/llm-d-router
inspected_commit: b1bf63da5e9a52dc8815264809d00f45f5b5e966
dependency: vLLM selective KV loading - primary CPU source control
related_issue: llm-d/llm-d-router#1952
---

# llm-d-router selective KV loading — policy and request wiring

## Purpose

This is a technical implementation guide for the llm-d-router half of selective KV loading. The router already knows how much of a request prefix is cached by tier on each candidate endpoint, but it does not send vLLM’s per-request `kv_transfer_params.kv_load_tiers` field.

The desired behavior is:

> After endpoint selection, decide whether loading cached KV from that
> endpoint’s offload tiers is likely to be better than recomputation, then
> inject the decision into the request that actually executes on that endpoint.

The engine implementation and promotion semantics are specified in [[vLLM selective KV loading - primary CPU source control]].

This contribution must not move KV blocks, implement an engine eviction policy, or duplicate vLLM’s cache manager. The router is the decision and request-wiring layer; vLLM remains the enforcement and data-movement layer.

## Status and inspected baseline

The design was verified against `llm-d/llm-d-router@b1bf63da5e9a52dc8815264809d00f45f5b5e966` on 2026-09-03.

No `kv_load_tiers` implementation exists in the repository. Duplicate-work checks found no open selective-loading PR or issue. Relevant adjacent work is:

- [issue #1952](https://github.com/llm-d/llm-d-router/issues/1952), exposing KV cache location to plugins;
- [PR #2397](https://github.com/llm-d/llm-d-router/pull/2397), tier metrics and storage-aware weighting, still open at inspection time;
- merged work exposing per-tier cached blocks and selecting P2P sources by CPU cache presence.

Repeat issue/open-PR searches immediately before coding. llm-d-router’s contribution guide also requires a tracking issue for nontrivial work.

## Existing building blocks

### Per-endpoint tier matches

The precise-prefix-cache data producer computes `matchedBlockCountByTier` and publishes it as `CachedBlocksByTier` on each candidate’s [`PrefixCacheMatchInfo`](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/plugins/datalayer/attribute/prefix/data_types.go#L35-L55).

The calculation is in [`preciseprefixcache/utils.go`](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/plugins/requestcontrol/dataproducer/preciseprefixcache/utils.go#L59-L94), and the producer writes it in [`producer.go`](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/plugins/requestcontrol/dataproducer/preciseprefixcache/producer.go#L385-L395).

The current index records a medium-like `DeviceTier` for each pod/block entry, which is enough for a medium-only MVP. Router KV-event adaptation currently drops vLLM’s newer locality and ownership fields. Therefore:

- CPU versus storage decisions are feasible now;
- LOCAL versus REMOTE block-level policy is not reliable without extending the event model and index;
- locality should be omitted as a wildcard in the MVP or supplied only from trusted static topology configuration.

### Pre-request mutation

[`InferenceRequestBody.MutatePayloadMap`](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/interface/requesthandling/types.go#L146-L155) can modify the parsed OpenAI payload. The director executes `PreRequest` plugins after scheduling and repackages the mutated body before forwarding.

The existing [`p2psource` data producer](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/plugins/requestcontrol/dataproducer/p2psource/producer.go#L249-L325) is the closest analogue: it declares a precise-prefix dependency, inspects the selected destination, and mutates trusted transfer metadata in `PreRequest`.

### Transfer-map reconstruction

Direct EPP forwarding can preserve a nested map by mutation. Sidecar and coordinator paths are more dangerous because several branches synthesize a new `kv_transfer_params` map.

In particular, [`connector_p2p.go`](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/sidecar/proxy/connector_p2p.go) constructs connector parameters for prefiller and decoder requests, and `decodeWithP2PSource` rebuilds the map. Unless changed deliberately, those paths discard `kv_load_tiers`.

The native `/inference/v1/generate` coordinator path has a different envelope: connector parameters must be nested under `sampling_params.extra_args`. A helper in the coordinator already handles this layout; selective-loading tests must cover it explicitly.

## Contract with vLLM

### Payload

For OpenAI-compatible requests:

```json
{
  "kv_transfer_params": {
    "kv_load_tiers": [
      {"medium": "CPU"},
      {"medium": "STORAGE"}
    ]
  }
}
```

For the native generate API, the same transfer map belongs under:

```json
{
  "sampling_params": {
    "extra_args": {
      "kv_transfer_params": {
        "kv_load_tiers": []
      }
    }
  }
}
```

### Three-state output

The plugin must preserve three distinct states:

| Router state | Wire behavior | Engine meaning |
|---|---|---|
| plugin disabled, insufficient evidence, or incompatible engine | omit `kv_load_tiers` | preserve vLLM default/all-tier behavior |
| affirmative source decision | send a non-empty matcher list | load only from authorized original sources |
| authoritative recompute decision | send `"kv_load_tiers": []` | do not load from any offload tier |

Never represent “recompute” by omitting the field. Never emit `[]` as an error fallback.

### Authority

When enabled and able to decide, the router’s value is authoritative and should replace a client-provided `kv_load_tiers`. Allowing an external client to override it defeats calibration, can force expensive I/O, and may reach connector behavior outside the deployment operator’s intent.

The mutation code must preserve connector-owned keys already set by trusted router components, such as transfer ids and remote engine ids. It must not blindly merge arbitrary inbound `kv_transfer_params`, because those maps can contain addresses or connector directives with SSRF and trust-boundary implications.

Recommended rule:

1. parse or create the destination transfer map;
2. retain only keys already produced by trusted in-process components;
3. set/overwrite the validated `kv_load_tiers` value;
4. serialize through the existing request-body abstraction.

If multiple processes must carry the decision, use a trusted internal header that is stripped at ingress and regenerated by the router, following the p2psource pattern. Do not trust the same header when supplied externally.

## Recommended MVP

### Scope

The first router contribution should consume only per-endpoint precise-prefix data and make a binary decision:

- omit the field when no safe decision can be made;
- send `[]` when the selected endpoint’s offloaded prefix is below a configured break-even threshold;
- otherwise omit the field and let existing vLLM load selection operate.

This MVP pairs with Phase 1 of [[vLLM selective KV loading - primary CPU source control]]. It avoids claiming CPU-versus-storage precision before vLLM has promotion-aware primary filtering.

A second router phase can emit CPU and STORAGE matchers after the full engine contract lands.

### Why binary first

Current vLLM filters secondary tiers but always consults CPU. If the router emits a STORAGE-only filter today, the engine may still reuse a CPU copy. If it emits `[]`, the engine currently still reuses CPU. A deployment compatibility gate is therefore mandatory even for the binary mode.

Binary opt-out supplies the primary product value — avoiding low-benefit loads and respecting the router’s recompute decision — with a much smaller semantic surface.

## Plugin architecture

### Extension point

Add a request-control plugin next to `p2psource`, tentatively:

```text
pkg/epp/framework/plugins/requestcontrol/dataproducer/selectiveload/
    config.go
    producer.go
    producer_test.go
    README.md
```

Implement both the data-producer and `PreRequest` roles, matching the analogous p2psource lifecycle:

1. declare a dependency on the configured precise-prefix-cache producer;
2. during production, derive endpoint-specific evidence or a decision from `PrefixCacheMatchInfo`;
3. after scheduling, retrieve the decision for the actual selected endpoint;
4. mutate the outgoing request in `PreRequest`;
5. expose bounded decision metrics.

A PreRequest-only plugin would be smaller, but it would hide the dependency on precise-prefix data and make ordering/configuration errors easier. Prefer the data-producer pattern unless framework maintainers recommend a dedicated post-selection mutation interface.

Register the plugin in the EPP runner alongside other request-control data producers. Provide a README and sample configuration; plugin architecture is the project’s intended feature extension surface.

### Internal decision type

Keep policy computation separate from payload mutation. A conceptual value is:

```go
type SelectiveLoadDecision struct {
    State        DecisionState // Unknown, Recompute, Load
    AllowedTiers []TierMatcher
    Reason       DecisionReason
    Evidence     DecisionEvidence
}
```

Evidence may include per-medium matched token counts and the configured threshold used. It should not be serialized directly into the user request.

Use typed medium constants at the boundary, then serialize the exact vLLM strings `CPU` and `STORAGE`. GPU is not a load tier in this contract; vLLM checks its local GPU prefix cache before the offloading connector.

### Endpoint binding

The decision must be selected after the scheduler chooses the destination. Do not use the maximum cached prefix across all candidates. The hint must describe the endpoint that will execute the request.

For retries or endpoint reselection, recompute or reselect the decision for the new endpoint. Do not carry a decision that was derived from the failed destination.

## Policy

### Input contract

For the selected endpoint, consume:

- number of matched prefix blocks/tokens by CPU medium;
- number of matched prefix blocks/tokens by storage medium;
- request prompt-token or block count;
- static policy configuration;
- optional future tier health/cost signals.

Use contiguous reusable prefix length, not the total number of matching blocks at arbitrary positions. Keep block-to-token conversion aligned with the precise-prefix producer’s tokenizer and block-size assumptions.

### MVP threshold policy

A deterministic threshold is preferable to a model for the first PR:

```text
offloaded_matched_tokens =
    max(cpu_matched_tokens, storage_matched_tokens)

if evidence is missing or stale:
    Unknown -> omit field
else if offloaded_matched_tokens < min_cached_tokens:
    Recompute -> []
else:
    Load -> omit field
```

Use `max`, not a sum, when the same prefix may be represented in multiple tiers. Summing duplicates would overstate saved computation.

Consider both an absolute and relative gate only if existing experiments justify them:

```text
qualifies =
    matched_tokens >= min_cached_tokens
    and matched_tokens / prompt_tokens >= min_cached_fraction
```

Boundary behavior must be explicit: equality should qualify. A zero-token prompt or unknown tokenizer result yields `Unknown`, not `Recompute`.

### Full source-selective policy

After vLLM supports strict primary source filtering, evaluate each source independently:

```text
allowed = []

if cpu evidence is valid and CPU load benefit exceeds CPU cost:
    allowed += CPU

if storage evidence is valid and storage load benefit exceeds storage cost:
    allowed += STORAGE

if no evidence is trustworthy:
    omit the field
else:
    send allowed  // may be []
```

A useful calibration form is:

```text
benefit(source) =
    estimated_recompute_time(matched_tokens)
    - estimated_load_time(source, matched_bytes)
    - queue_or_pressure_penalty(source)

allow source when benefit(source) >= safety_margin
```

Do not land an opaque adaptive policy before the deterministic path is observable. Start with configured thresholds backed by the calibration protocol, then add measured bandwidth, CPU-tier pressure, or storage latency as separate inputs.

### Staleness and uncertainty

Tier events and the router index are eventually consistent. The policy must distinguish a measured zero from absent/stale data.

Recommended fallback:

- missing producer output: omit;
- selected pod absent from output: omit;
- unknown tier string: ignore that tier and record a bounded diagnostic;
- stale beyond configured age: omit;
- valid measured zero across known offload tiers: `[]`;
- mutation/serialization failure: fail the plugin according to existing framework conventions, but do not silently convert it to `[]`.

Fail-open here means “preserve default engine behavior,” not “allow the request to bypass normal routing failure handling.”

## Configuration

A conceptual configuration is:

```yaml
name: selective-load
type: data-producer
parameters:
  prefixMatchInfoProducerName: precise-prefix-cache
  mode: binary
  minCachedTokens: 512
  maxEvidenceAge: 2s
  engineCapability: offload-selective-load-primary-v1
```

For the later source-selective mode:

```yaml
parameters:
  mode: per-tier
  cpu:
    minCachedTokens: 256
  storage:
    minCachedTokens: 1024
  minCachedFraction: 0.25
```

Adapt field names to the repository’s established configuration schema during implementation. Required semantics are:

- disabled by default;
- explicit dependency name;
- validated non-negative thresholds;
- an explicit compatible-engine capability;
- unknown mode/config rejected during startup;
- no silent activation in a heterogeneous pool.

Do not make open PR #2397 a hard dependency for the MVP. Tier-pressure metrics are valuable later, but prefix evidence plus a static threshold is enough to validate wiring and semantics.

## Capability and rollout gate

There is no safe runtime inference that a backend honors primary-tier filtering. Use deployment-level capability configuration initially.

Only emit the field when every eligible endpoint in the relevant pool is known to run a compatible vLLM version/configuration. If the pool is heterogeneous, either exclude incompatible endpoints from the plugin’s scope or omit the field for the request.

Suggested capability levels:

| Capability | Router may emit |
|---|---|
| none/unknown | nothing |
| binary opt-out | `[]` only |
| source-selective v1 | `[]`, CPU, STORAGE, combined |

Treat the capability as operator-controlled deployment metadata, not a client-supplied header.

## Direct EPP request wiring

In `PreRequest`:

1. obtain the scheduled destination;
2. obtain its decision;
3. return without mutation for `Unknown`;
4. validate the outgoing payload is the supported request shape;
5. obtain/create `kv_transfer_params`;
6. overwrite `kv_load_tiers` with the decision list;
7. write through `MutatePayloadMap`;
8. attach trace fields and increment a bounded counter.

Keep the mutation idempotent. Running it twice must produce the same matcher list without duplicating entries or damaging other transfer fields.

Canonicalize order — CPU before STORAGE — so logs, tests, and request hashes are stable.

## Sidecar and disaggregated paths

### Preserve through generated maps

Create one small trusted helper for inserting the selective-load key into a router-generated transfer map. Use it from every branch that constructs or reconstructs `kv_transfer_params`, rather than duplicating merge logic.

Audit at least:

- prefiller map construction;
- decoder map construction;
- `decodeWithP2PSource`;
- NIXL connector variants;
- Mooncake connector variants;
- retry/replay request construction;
- native generate coordinator helpers.

The helper should accept a validated internal decision, not arbitrary client JSON.

### P/D and E/P/D semantics

A decision applies to a compute leg and endpoint, not automatically to the whole logical request.

- Prefill can make its own reuse decision for the selected prefill endpoint.
- Decode may receive KV directly from prefill; in that case offload lookup may be irrelevant or actively undesirable.
- If decode independently performs prefix reuse, derive its decision from the decode endpoint’s cache evidence.
- E/P/D paths require the same separation for each actual vLLM invocation.

Do not copy one endpoint’s `kv_load_tiers` to both prefill and decode requests without proving they share cache state and policy.

### Chunked decode

The sidecar removes `kv_transfer_params` after the first decode chunk. That can be correct: subsequent chunks should reuse the context already established in GPU rather than repeat an external lookup. Add a behavior test that proves:

- the first vLLM decode request carries the selective-load decision;
- continuation chunks do not restart offload loading;
- retries that begin a new backend request receive a freshly selected decision.

If vLLM’s connector lifecycle requires the field on every chunk, change the sidecar behavior explicitly rather than preserving it accidentally.

## KV-event locality follow-up

Full locality-aware matchers require extending the router’s event model. vLLM events include medium, locality, and ownership, while the inspected router adapter/index retains only device tier.

A future contribution should:

1. add typed locality and ownership fields to router BlockStored/BlockRemoved;
2. preserve them in the vLLM event adapter;
3. add them to block-location entries and removal identity;
4. expose per-medium/per-locality prefix counts;
5. define compatibility for older event producers;
6. bound index and metric cardinality;
7. test mixed-version and removal behavior.

Do not block the medium-only selective-loading MVP on this work.

## Observability

Record enough data to answer “did the router prevent a harmful load?” without high-cardinality labels.

Recommended counters/histograms:

- decisions: unknown / recompute / load;
- reason: below-threshold / qualifying-prefix / stale / missing / incompatible-engine;
- allowed tier set: none / cpu / storage / cpu-storage;
- selected endpoint’s matched tokens by medium;
- decision latency;
- overwrite of a client-supplied value;
- sidecar propagation failure.

Trace attributes may include selected endpoint, matched token counts, threshold, and capability. Do not put request ids, block hashes, arbitrary model names, or raw JSON into metric labels.

For rollout analysis, correlate router decisions with vLLM TTFT, recomputed tokens, CPU-to-GPU bytes, storage-to-CPU bytes, and source-hit counters.

## Tests

### Pure policy tests

Use table-driven tests for:

- below, equal to, and above threshold;
- absolute plus fractional threshold boundaries;
- duplicate CPU/storage coverage uses max, not sum;
- valid zero versus missing data;
- stale evidence;
- unknown tier strings;
- zero-length prompt;
- binary versus per-tier capability;
- deterministic matcher order.

Keep the policy function independent of request mutation so these tests are small and exhaustive.

### Plugin lifecycle tests

Extend analogous dataproducer and director tests:

1. dependency on precise-prefix output is declared/resolved;
2. decision is bound to the selected endpoint;
3. reselection changes the decision;
4. `Unknown` leaves the payload untouched;
5. `Recompute` writes an explicit empty list;
6. existing trusted transfer keys survive;
7. client `kv_load_tiers` is overwritten;
8. repeated `PreRequest` is idempotent;
9. disabled/default configuration produces no mutation;
10. incompatible engine capability produces no mutation.

### Sidecar/coordinator tests

For each map-building path, assert the complete outgoing request:

- OpenAI completion/chat envelope;
- prefiller and decoder maps;
- P2P source and non-P2P branches;
- configured NIXL/Mooncake variants;
- native generate nesting;
- first versus continuation decode chunk;
- retry/reselection;
- malicious or malformed inbound transfer keys are not trusted.

Prefer an assertion on the backend-observed JSON body over a unit test of only the helper.

### Integration test

Use a fake backend that records the body:

1. publish deterministic prefix-tier state for two pods;
2. make the scheduler choose one pod;
3. verify that pod’s decision, not the other candidate’s, reaches the backend;
4. repeat with a below-threshold prefix and observe `[]`;
5. repeat with missing/stale state and observe field omission;
6. exercise one disaggregated path.

A later vLLM integration should demonstrate that `[]` produces recomputation even when the chosen backend has a ready CPU copy.

### Suggested commands

The repository’s container-backed Make targets are canonical:

```bash
make test-filter PATTERN=SelectiveLoad TYPE=epp
make test-filter PATTERN=SelectiveLoad TYPE=sidecar
make test-unit
make presubmit
```

Record exact commands and results in the PR. Run broader e2e tests when the request path or deployment configuration changes.

## Failure modes and safeguards

| Failure | Safeguard |
|---|---|
| `[]` reaches an older vLLM and CPU still loads | explicit deployment capability gate |
| decision derived from the wrong candidate | choose in post-scheduling `PreRequest` by endpoint identity |
| sidecar overwrites the hint | central trusted insertion helper plus full-body tests |
| arbitrary client connector fields survive | whitelist trusted in-process fields; overwrite policy |
| stale index causes false recompute | age check; omit on uncertainty |
| duplicate block counts exaggerate benefit | use reusable prefix length/max, not cross-tier sum |
| retries keep stale policy | recompute on endpoint reselection |
| P/D applies one decision to both legs | maintain leg-specific decisions |
| metrics explode in cardinality | fixed enum labels only |
| plugin breaks requests when disabled | no registration/no mutation by default |

## PR decomposition

### PR 1 — binary decision on direct EPP path

- Open tracking issue and repeat duplicate checks.
- Add plugin, config validation, policy function, runner registration, README.
- Consume precise-prefix per-tier evidence.
- Emit only `[]` or omit.
- Add policy/director tests and bounded metrics.
- Require the binary vLLM capability.

### PR 2 — propagation through sidecar/coordinator

- Add the trusted transfer-map insertion helper.
- Wire every applicable connector/map-building branch.
- Cover P/D, native generate, chunk continuation, and retry behavior.
- Document trust-boundary handling.

This may need to land with PR 1 if the target deployment always uses the sidecar; otherwise the split makes review easier.

### PR 3 — per-tier policy

- Require vLLM source-selective capability.
- Emit CPU/STORAGE matcher lists.
- Add calibrated per-tier thresholds and optional pressure/cost inputs.
- Add source-specific metrics and tests.
- Do not enable by default until calibration results are recorded.

### PR 4 — locality-aware evidence, optional

- Preserve locality/ownership through KV events and index.
- Expose locality-specific prefix matches.
- Add LOCAL/REMOTE matchers only after mixed-version behavior is defined.

## Acceptance criteria

The binary contribution is complete when:

- the decision uses the selected endpoint’s valid precise-prefix evidence;
- below-threshold reuse is encoded as explicit `[]`;
- unknown, stale, disabled, and incompatible states omit the field;
- direct and deployed sidecar request paths preserve the decision to vLLM;
- connector-owned trusted fields survive while client policy cannot override the router;
- retries and disaggregated legs cannot reuse a decision for the wrong endpoint;
- metrics and traces explain each decision;
- focused unit tests and `make presubmit` pass;
- the PR documents its vLLM compatibility requirement and AI assistance.

Full source-selective loading is complete when:

- CPU and STORAGE decisions are independently calibrated;
- the router’s matcher list is enforced literally by compatible vLLM;
- locality is either intentionally wildcarded or backed by indexed evidence;
- an end-to-end test distinguishes recompute, CPU reuse, and storage reuse.

## Open decisions

1. What is the initial break-even threshold for the target models/hardware?
2. Is the first production path direct EPP, sidecar P/D, coordinator, or all three?
3. How will compatible vLLM capability be represented in deployment config?
4. Should valid zero cached tokens produce `[]`, or should the policy require a minimum evidence age/sample confidence first?
5. Does a router decision override every client hint, or are there trusted internal callers that need an explicit precedence layer?
6. Which component owns per-leg decisions in P/D deployments?
7. When PR #2397 lands, which tier-pressure metrics are stable enough to enter the policy?
8. What calibration artifact and rollback threshold are required before default enablement?

## Related

- [[vLLM selective KV loading - primary CPU source control]]
- [[vLLM and llm-d-router KV cache responsibility split]]
- [[vLLM KV Events canonical form]]
- [[vLLM KV offload retrieval path - lookup, promotion, and load]]
- [[Activity-Based KV Cache Offloading]]
- [[01 - Calibration Protocol]]

## Provenance

Direct inspection of the local llm-d-router checkout at `b1bf63da5e9a52dc8815264809d00f45f5b5e966`, the current precise-prefix, p2psource, director, sidecar, coordinator, KV-event and index paths, plus issue/open-PR duplicate searches on 2026-09-03. This is a proposed design, not a description of functionality already present.