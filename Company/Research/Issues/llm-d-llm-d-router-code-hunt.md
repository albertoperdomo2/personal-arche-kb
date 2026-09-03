---
repo: llm-d/llm-d-router
last_updated: 2026-09-03
---

# Code Hunt — llm-d/llm-d-router

Unfiled-work view of the codebase, separate from the issue-triage backlog (`llm-d-llm-d-router.md`). Items here are confirmed findings not yet tracked as issues/PRs.

## Bugs

## Performance

- **`NewEndpoint` unnecessary metadata/metrics clone** · Medium · Confirmed — `meta.Clone()` and `metrics.Clone()` allocate 2N structs + 3N maps per request; all scorers are read-only. Sharing the datastore's `atomic.Pointer` values gives snapshot semantics without allocations. First seen 2026-09-03.
  - code: [https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/interface/scheduling/types.go](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/interface/scheduling/types.go)
  - code: [https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/interface/datalayer/endpoint_metadata.go](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/interface/datalayer/endpoint_metadata.go)
  - code: [https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/interface/datalayer/metrics.go](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/framework/interface/datalayer/metrics.go)
  - cost: Per request for N endpoints: 2N struct + 3N map allocations + 3N `maps.Copy` + N `Attributes.Clone()`. At N=100, ~500K allocations/sec at 1000 req/s. Pure waste — sharing pointers preserves snapshot semantics.
  - suggested_action: In `NewEndpoint`, store the original `meta` and `metrics` pointers without cloning; only `attr.Clone()` is needed (DataProducers write to attributes before scheduling). The datastore's `atomic.Pointer`-based `UpdateMetadata`/`UpdateMetrics` already provide snapshot semantics.
  - effort: S — remove `meta.Clone()`/`metrics.Clone()` in `NewEndpoint`; verify no scheduler path mutates the returned pointers (confirmed: all in-tree scorers/filters read-only).
  - fp: `llm-d/llm-d-router:pkg/epp/framework/interface/scheduling/types.go:NewEndpoint:unnecessary-metadata-metrics-clone`

- **Uncached `PodList` in default (flow-control off) mode** · Low · Confirmed — `DatastoreEndpointCandidates.Locate` runs `sync.Map.Range` + slice-alloc per request; `CachedEndpointCandidates` exists but is gated behind the flow-control feature flag (default off). First seen 2026-09-03.
  - code: [https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/requestcontrol/candidates.go](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/pkg/epp/requestcontrol/candidates.go)
  - code: [https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/cmd/epp/runner/runner.go](https://github.com/llm-d/llm-d-router/blob/b1bf63da5e9a52dc8815264809d00f45f5b5e966/cmd/epp/runner/runner.go)
  - cost: Per request O(N) `sync.Map.Range` + ~log₂(N) slice reallocations; 2× for sheddable requests (called again in `LegacyAdmissionController.Admit`). Small relative to clone costs, but fix is trivial and cache already exists.
  - suggested_action: Wrap `endpointCandidates` in `NewCachedEndpointCandidates` in `initAdmissionControl` regardless of the `flowControl` gate. The 50ms TTL is short enough for routing decisions.
  - effort: S — move the `NewCachedEndpointCandidates` call outside the `if !r.featureGates[flowcontrol.FeatureGate]` branch.
  - fp: `llm-d/llm-d-router:pkg/epp/requestcontrol/candidates.go:DatastoreEndpointCandidates.Locate:uncached-podlist-default`

## Improvements

## Recently Resolved
