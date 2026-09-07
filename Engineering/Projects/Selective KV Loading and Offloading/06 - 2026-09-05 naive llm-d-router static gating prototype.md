---
title: "Naive llm-d-router static selective-KV gating prototype"
date: 2026-09-05
type: implementation-note
status: implemented-local
topic: selective KV loading and offloading
repo: llm-d/llm-d-router
branch: prototype/selective-kv-policy
commit: ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4
base_commit: 44d33599ef95b4d46ae54f431ce4e8b3fd1183f4
---

# Naive llm-d-router static selective-KV gating prototype

## Summary

The first llm-d-router implementation is intentionally naive: it applies one static load policy and one static offload policy to every supported request. It proves the router can authoritatively inject vLLM's per-request controls before backend dispatch, without yet deciding whether a particular cached prefix is worth loading.

The implementation is committed locally as `ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4` on branch `prototype/selective-kv-policy`, based on upstream commit `44d33599ef95b4d46ae54f431ce4e8b3fd1183f4`. The branch has no configured upstream and this note does not claim that the work is pushed or proposed upstream.

## Why this is the naive version

This version answers only the wiring question:

> Can the router independently disable loading existing offloaded KV and/or storing newly computed KV, and can it deliver those decisions to a compatible vLLM request?

It does not answer the policy question:

> For this request, endpoint, prefix length, tier health, and system pressure, is loading faster than recomputation?

The static version is useful because it isolates request mutation and deployment compatibility from cache-evidence quality and calibration. A cluster test can first establish that the backend receives the fields and that vLLM metrics show the corresponding lookup and transfer behavior. The threshold policy can then replace the static decision without redesigning the wire contract.

## Implemented contract

A new alpha request-control plugin is registered as `selective-kv-policy`.

Its configuration has three fields:

| Field | Meaning |
|---|---|
| `engineCapability` | Required operator assertion that every eligible backend implements the binary empty-list opt-out |
| `loadPolicy` | `preserve` or `disable`; omitted defaults to `preserve` |
| `offloadPolicy` | `preserve` or `disable`; omitted defaults to `preserve` |

The only accepted capability value is `binary-opt-out-v1`. This is a startup-time configuration gate, not runtime capability discovery.

The two directions are independent:

| Load policy | Offload policy | Router mutation |
|---|---|---|
| preserve | preserve | Do not mutate either field |
| disable | preserve | Set `kv_load_tiers: []` |
| preserve | disable | Set `max_offload_tokens: 0` |
| disable | disable | Set both fields |

The combined wire result is:

```json
{
  "kv_transfer_params": {
    "kv_load_tiers": [],
    "max_offload_tokens": 0
  }
}
```

The empty load-tier list means that compatible vLLM instances skip all offload sources, including primary CPU and secondary storage, while retaining local HBM prefix-cache reuse. The zero offload-token cap prevents newly computed request KV from being stored through the offloading connector.

## Router implementation

The implementation is under `pkg/epp/framework/plugins/requestcontrol/selectivekv/`.

- `plugin.go` defines the configuration, validates the explicit capability and policy enums, and implements the `requestcontrol.PreRequest` extension.
- `plugin_test.go` covers configuration rejection, all four independent policy combinations, client-value override, sibling preservation, idempotence, malformed transfer parameters, and unsupported request shapes.
- `README.md` documents the configuration and initial scope.
- `cmd/epp/runner/runner.go` registers the plugin as alpha.

During `PreRequest`, the plugin obtains or creates the top-level `kv_transfer_params` JSON object. It overwrites only the policy fields for directions configured as `disable`. Existing sibling entries such as `remote_engine_id` are retained. If a client supplied a conflicting `kv_load_tiers` or `max_offload_tokens`, the router's configured decision wins.

Mutation uses the existing `InferenceRequestBody.MutatePayloadMap` path so the request is marked for reserialization before forwarding.

## Example optimized-baseline configuration

The optimized-baseline scheduling plugins remain unchanged. The router image must contain commit `ac5446eb`, alpha plugins must be enabled, and the new plugin is added to the global plugin list. It is not referenced by a scheduling profile because it is a post-scheduling `PreRequest` hook rather than a filter or scorer.

```yaml
router:
  epp:
    image:
      registry: quay.io/rh-ee-aperdomo
      repository: llm-d-router-endpoint-picker
      tag: selective-kv-policy-v1
      pullPolicy: Always

    flags:
      allow-experimental-plugins: true

    pluginsConfigFile: optimized-baseline-plugins.yaml
    pluginsCustomConfig:
      optimized-baseline-plugins.yaml: |
        apiVersion: llm-d.ai/v1alpha1
        kind: EndpointPickerConfig
        plugins:
        - type: approx-prefix-cache-producer
        - type: inflight-load-producer
        - type: prefix-cache-affinity-filter
        - type: token-load-scorer
        - type: selective-kv-policy
          name: selective-kv-policy
          parameters:
            engineCapability: binary-opt-out-v1
            loadPolicy: disable
            offloadPolicy: disable

        schedulingProfiles:
        - name: default
          plugins:
          - pluginRef: prefix-cache-affinity-filter
          - pluginRef: token-load-scorer
```

The image name above is the intended shape, not evidence that the image has already been built or pushed.

For a load-only experiment that continues to seed the offload cache, use:

```yaml
loadPolicy: disable
offloadPolicy: preserve
```

## Validation completed

The focused test command passed:

```text
make test-filter PATTERN=SelectiveKV TYPE=epp
```

The selected package ran the following behavior groups successfully:

- factory construction and invalid configuration;
- preserve/disable combinations for both directions;
- authoritative overwrite of client policy fields;
- preservation of other transfer parameters;
- repeated `PreRequest` idempotence;
- replacement of malformed `kv_transfer_params`;
- rejection of raw and native-generate request shapes.

The full repository gate also passed:

```text
make presubmit
```

Observed results were clean `go mod tidy`, formatting success, zero lint/typo issues, no called vulnerabilities from `govulncheck`, and no external YAML images using `:latest`.

The first presubmit invocation stopped at the repository's main-branch guard. A local feature branch was created, after which the full gate passed. This was a workflow guard, not a code failure.

A duplicate-work search run before implementation found no open llm-d-router issue or pull request matching selective loading, `kv_load_tiers`, or `max_offload_tokens`. A tracking issue is still required before upstream contribution.

## Intentional limitations

- The policy is constant across requests. It does not inspect matched prefix length, CPU pressure, storage pressure, predicted recompute cost, or the selected endpoint.
- `engineCapability` is an operator assertion. The plugin does not detect mixed compatible/incompatible backend pools.
- Only binary opt-out is represented. Non-empty CPU-only or STORAGE-only selection is out of scope.
- Positive `max_offload_tokens` caps are not configurable in this version.
- The implementation covers parsed OpenAI-compatible JSON on the direct EPP path.
- Native `/inference/v1/generate` requests are rejected when mutation is required because their transfer parameters use a different nested envelope.
- Raw/unparsed payloads are rejected when mutation is required.
- Sidecar, coordinator, P/D, E/P/D, retry, and request-reconstruction propagation are not implemented.
- The plugin has no policy-specific decision metrics. The framework's generic plugin-processing latency remains available.
- The optimized baseline's approximate-prefix producer is unchanged because the static policy consumes no cache evidence. It will not be sufficient for a tier-aware threshold decision.

## Next validation

Build and push a router image containing `ac5446eb`, deploy it with the optimized-baseline configuration, and send a request through EPP to the compatible vLLM image. Validate:

1. EPP starts only when `allow-experimental-plugins=true` and the capability value is accepted.
2. The backend-observed request contains the expected sibling fields.
3. With `loadPolicy=disable`, vLLM offload lookup counters do not increase.
4. With `offloadPolicy=disable`, GPU-to-CPU/store bytes do not increase.
5. A repeated HBM-resident prefix still reports local cached tokens.
6. Removing the plugin restores the original optimized-baseline behavior.

After the static vertical slice is validated, replace `loadPolicy` with a deterministic threshold decision derived from the selected endpoint's precise per-tier contiguous-prefix evidence. Missing, stale, or incompatible evidence must omit the field rather than emit an empty list.

## Decision

Keep this implementation as the naive reference version. Its purpose is to validate capability gating and request propagation. Do not treat static `disable` as the production policy or infer a performance benefit from it.

## Related

- [[00 - Index]]
- [[01 - Initial implementation plan]]
- [[02 - vLLM primary-tier selective loading]]
- [[03 - llm-d-router selective load and offload policy and wiring]]
- [[04 - vLLM selective load - problem statement]]

## Provenance

Direct inspection of local `llm-d/llm-d-router` branch `prototype/selective-kv-policy` at `ac5446ebda7b5ef2b7c42254eff6ef8bce19d6c4`, the implementation and tests under `pkg/epp/framework/plugins/requestcontrol/selectivekv/`, runner registration, the optimized-baseline Helm values, and observed focused/presubmit output from 2026-09-04 through 2026-09-05.