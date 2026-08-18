---
repo: llm-d/llm-d-router
last_updated: 2026-08-18
---

# Backlog — llm-d/llm-d-router

## Workable Issues

- **Native generate requests bypass the multimodal encoder** · High · Likely — Add generate-format extraction to `mmItemsForFanout` + integration test  <!-- fp: llm-d/llm-d-router:issue:generate-api-skips-multimodal-encoder -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2108
- **Add coordinated scheduling for disaggregated inference stages** · Medium · Confirmed — Design cross-stage selection metadata; benchmark deferred vs coordinated  <!-- fp: llm-d/llm-d-router:issue:coordinated-disaggregated-stage-selection -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2135
- **`make presubmit` fails on main: `zmq-listener` pinned to `:latest`** · High · Confirmed — Pin image to digest/version or exempt the perf-test path with documented rationale  <!-- fp: llm-d/llm-d-router:issue:presubmit-zmq-listener-latest-tag -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2181
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/test/perf/config/llm-d-sim-deployment-zmq.yaml#L92
- **inflight-load-producer holds prefill request counter until EndOfStream, inflating saturation credit** · High · Confirmed — Release the prefill profile's request counter at `StartOfStream` in both config paths; add saturation-detector test under P/D workload  <!-- fp: llm-d/llm-d-router:issue:inflight-load-prefill-counter-inflates-saturation-credit -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2174
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L358
- **Utilization detector pool aggregation (average vs. max/routing-aware) needs reconsideration** · Medium · Likely — Propose max/routing-aware variant behind config flag with benchmark under skewed load  <!-- fp: llm-d/llm-d-router:issue:utilization-detector-pool-aggregation -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2173
- **Remove deprecated `--vllm-port` flag and YAML key (v0.12)** · Low · Confirmed — Track for v0.12; remove alias + deprecation log, update release notes  <!-- fp: llm-d/llm-d-router:issue:remove-deprecated-vllm-port-flag -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2172
- **`probabilistic-admitter-epp-config.yaml` header no longer describes admission composition** · Low · Confirmed — Update header comment against `pkg/epp/requestcontrol/admission.go`  <!-- fp: llm-d/llm-d-router:issue:probabilistic-admitter-config-header-stale -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2182
- **Chart ships no RBAC for the EPP's authenticated /metrics endpoint (401 anonymous, 500 with token)** · High · Confirmed — Add a gated `system:auth-delegator` ClusterRoleBinding for the EPP ServiceAccount plus a `<release>-metrics-reader` ClusterRole behind a chart value; document the bearer-token scrape pattern  <!-- fp: llm-d/llm-d-router:issue:chart-metrics-rbac-missing -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2370
  - code: https://github.com/llm-d/llm-d-router/blob/800ec0e453d8b962ca0af6f03d3bfbe8e685350c/cmd/epp/runner/runner.go#L390-L399
- **Support enabling EPP feature gates without duplicating the full plugin config** · Medium · Confirmed — Add a `--feature-gates` flag in `pkg/epp/server/options.go`, append parsed entries to `rawConfig.FeatureGates` after the config file, and add a structured `router.epp.pluginsConfig` chart map  <!-- fp: llm-d/llm-d-router:issue:epp-feature-gates-flag -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2358
- **Remove deprecated UDS/legacy tokenizer code paths and drop the llm-d-kv-cache dependency** · Low · Confirmed — Remove the UDS/legacy tokenizer paths and `TokenizersPoolConfig`, then drop `github.com/llm-d/llm-d-kv-cache` from `go.mod`/`go.sum` and confirm `make presubmit` passes  <!-- fp: llm-d/llm-d-router:issue:drop-deprecated-uds-tokenizer-kvcache-dep -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2361
- **`pkg/epp/README.md` links 404 after the GIE API consolidation** · Low · Confirmed — Repoint links to current locations or convert to relative in-repo references; verify with link checker (.lychee.toml)  <!-- fp: llm-d/llm-d-router:issue:epp-readme-broken-links -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2433
  - code: https://github.com/llm-d/llm-d-router/blob/dd65342ee538ab76d4b9f7e1efb5d8e95385d06e/pkg/epp/README.md#L3-L13

## Bugs

- **Sidecar connector handlers buffer request bodies without a size limit** · Medium · Confirmed — Add `MaxBytesReader` route boundary with configurable limit + 413  <!-- fp: llm-d/llm-d-router:bug:sidecar-unbounded-request-body -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/connector_p2p.go#L42
- **Detached prefill requests can remain blocked indefinitely** · Medium · Likely — Derive lifecycle-bound context with configurable prefill deadline + test  <!-- fp: llm-d/llm-d-router:bug:detached-prefill-goroutine-no-deadline -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/connector_p2p.go#L124
- **DataProducer plugin failure is logged-and-continued; scheduling proceeds with missing data** · Medium · Likely — Emit counter on DataProducer failure; consider failing fast for `Required` producers  <!-- fp: llm-d/llm-d-router:bug:dataproducer-failure-silent-schedule-degradation -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/epp/requestcontrol/director.go#L309-L313
- **SSRF allowlist informer `AddEventHandler` errors are discarded** · Medium · Likely — Propagate registration error; add startup self-check that allowlist is non-empty  <!-- fp: llm-d/llm-d-router:bug:allowlist-informer-registration-errors-swallowed -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/allowlist.go#L157
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/allowlist.go#L320
- **P2P prefill panic is recovered and logged but decode proceeds assuming KV was stored** · Medium · Likely — Track prefill outcome via shared state; fail fast with 502 if prefill errored; record metric for prefill panics  <!-- fp: llm-d/llm-d-router:bug:p2p-prefill-panic-recovery-no-propagation -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/connector_p2p.go#L150-L162
- **`p2pPortFor` silently falls back to rank 0 on out-of-range/unparsable target ports** · Medium · Likely — Return error or emit WARN on out-of-range rank so multi-pod DP misconfigs are diagnosable  <!-- fp: llm-d/llm-d-router:bug:p2pportfor-silent-rank0-fallback -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/connector_p2p.go#L260-L285
- **`managedQueue` panics on negative length in the flow-control hot path** · Low · Speculative — Replace with error return / metric + clamp-to-zero, or gate behind debug build tag  <!-- fp: llm-d/llm-d-router:bug:managedqueue-negative-length-panic-hotpath -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/epp/flowcontrol/registry/managedqueue.go#L206
- **`dataParallelProxies` map shared across DP clones without synchronization guard** · Low · Speculative — Freeze map after startup or add explicit contract + test against runtime mutation  <!-- fp: llm-d/llm-d-router:bug:dataparallelproxies-shared-map-no-sync-guard -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/proxy.go#L333
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/data_parallel.go#L23
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/data_parallel.go#L50
- **CostAwareMemoryIndex and the Index interface have no Close/release, leaking Ristretto + LRU resources and OOMing CI** · Medium · Confirmed — Add `Close() error` (or `io.Closer`) to the `Index` interface, implement on every backend (Ristretto `cache.Close()`), register `t.Cleanup` in `createCostAwareIndexForTesting`, and use a small `NumCounters` in tests  <!-- fp: llm-d/llm-d-router:bug:costaware-index-no-close -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2349
  - code: https://github.com/llm-d/llm-d-router/blob/800ec0e453d8b962ca0af6f03d3bfbe8e685350c/pkg/kvcache/kvblock/cost_aware_memory.go#L53-L92
  - code: https://github.com/llm-d/llm-d-router/blob/800ec0e453d8b962ca0af6f03d3bfbe8e685350c/pkg/kvcache/kvblock/index.go
  - code: https://github.com/llm-d/llm-d-router/blob/800ec0e453d8b962ca0af6f03d3bfbe8e685350c/pkg/kvcache/kvblock/cost_aware_memory_test.go#L31-L37
- **rewriteModelName does an unscoped global byte-replace per streaming chunk (split-field misses, content corruption)** · Low · Likely — Carry the last `len("\"model\":\"\"")+len(targetModel)` bytes across chunks so split fields still match, and scope the replacement to JSON token boundaries instead of a literal `ReplaceAll`  <!-- fp: llm-d/llm-d-router:bug:rewrite-model-name-global-replace -->
  - code: https://github.com/llm-d/llm-d-router/blob/800ec0e453d8b962ca0af6f03d3bfbe8e685350c/pkg/epp/handlers/server.go#L538-L541
  - code: https://github.com/llm-d/llm-d-router/blob/800ec0e453d8b962ca0af6f03d3bfbe8e685350c/pkg/epp/handlers/server.go#L627-L641
- **Streaming usage accumulation replaces the whole Usage struct, dropping Anthropic prompt/cache tokens from earlier chunks** · High · Confirmed — Replace struct replacement at response.go:81 with per-field merge (copy non-zero PromptTokens/CompletionTokens/PromptTokenDetails, recompute TotalTokens); add handler-level test driving two Anthropic events as separate chunks  <!-- fp: llm-d/llm-d-router:bug:anthropic-streaming-usage-prompt-tokens-overwrite -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2431
  - code: https://github.com/llm-d/llm-d-router/blob/dd65342ee538ab76d4b9f7e1efb5d8e95385d06e/pkg/epp/handlers/response.go#L80-L81
  - code: https://github.com/llm-d/llm-d-router/blob/dd65342ee538ab76d4b9f7e1efb5d8e95385d06e/pkg/epp/framework/plugins/requesthandling/parsers/anthropic/anthropic.go#L201-L209
- **Always-disagg P/D decider runs prefill work on decode pods when no prefiller is READY instead of failing** · Medium · Likely — Trace prefill-filter + pd_profile_handler path for empty-READY-prefiller case; propose fail-closed (503/rejected-no-endpoints) so misconfigured prefill tiers don't silently fall through to decode  <!-- fp: llm-d/llm-d-router:issue:pd-always-disagg-runs-prefill-on-decode-when-no-prefillers-ready -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2427
  - code: https://github.com/llm-d/llm-d-router/blob/dd65342ee538ab76d4b9f7e1efb5d8e95385d06e/pkg/epp/framework/plugins/scheduling/profilehandler/disagg/always_disagg_pd_decider.go#L48-L50

## Performance

- **Every scorer allocates a full endpoint map on the scheduling hot path** · Low · Likely — Benchmark; prototype reusable per-request score accumulator  <!-- fp: llm-d/llm-d-router:perf:per-scorer-endpoint-score-maps -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/epp/scheduling/scheduler_profile.go#L173
- **DataProducer plugins run sequentially with 400ms per-producer timeout on the request path** · Medium · Likely — Fan out independent producers concurrently with shared deadline via errgroup; preserve ordering only where `Consumes()` declares dependency  <!-- fp: llm-d/llm-d-router:perf:sequential-dataproducer-plugins-400ms-timeout -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/epp/requestcontrol/director.go#L395
- **datalayer.Scope rebuilds the plugin's static allowedPut/allowedGet key sets per filter/scorer invocation per request** · Medium · Likely — Compute each plugin's `allowedPut`/`allowedGet` once at registration (or memoize on the plugin/ScopedEndpoint) and reuse the immutable maps across `Scope` calls  <!-- fp: llm-d/llm-d-router:perf:datalayer-scope-rebuilds-static-keymaps -->
  - code: https://github.com/llm-d/llm-d-router/blob/800ec0e453d8b962ca0af6f03d3bfbe8e685350c/pkg/epp/datalayer/endpoint_scope.go#L189-L231
  - code: https://github.com/llm-d/llm-d-router/blob/800ec0e453d8b962ca0af6f03d3bfbe8e685350c/pkg/epp/scheduling/scheduler_profile.go
- **podsPerKeyPrintHelper builds a full string map dump on every ScoreTokens call regardless of trace logging being enabled** · Medium · Confirmed — Gate behind `traceLogger.V(logging.TRACE).Enabled()` or use a lazy marshaler so the O(B·N) map walk + quadratic string concatenation only runs when TRACE logging is on  <!-- fp: llm-d/llm-d-router:perf:scoretokens-podsperkeyprinthelper-eager-eval -->
  - code: https://github.com/llm-d/llm-d-router/blob/dd65342ee538ab76d4b9f7e1efb5d8e95385d06e/pkg/kvcache/indexer.go#L189-L190
  - code: https://github.com/llm-d/llm-d-router/blob/dd65342ee538ab76d4b9f7e1efb5d8e95385d06e/pkg/kvcache/indexer.go#L218-L229

## Features & RFCs

- **Enable the `gosec` linter; current `nolint:gosec` directives are vestigial** · Medium · Confirmed
  - **Problem:** `gosec` absent from enabled linters in `.golangci.yml`; `nolint:gosec` directives are unenforced. Security linting is absent across TLS, int conversion, and label-cardinality surfaces.
  - **Proposed approach:** Add `gosec` to linters, run `make lint`, triage findings, replace vestigial `nolint:gosec` with scoped `//nolint:gosec // <reason>` where justified.
  - **Impact:** Surfaces weak crypto, hardcoded credentials, TLS-verification gaps on every presubmit.
  - **Rough effort:** Low — config change + one triage pass.
  <!-- fp: llm-d/llm-d-router:feature:enable-gosec-security-linter -->
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/.golangci.yml
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/sidecar/proxy/proxy.go#L457
- **Implement per-request / per-band queue-wait TTL (currently stubbed to 0)** · Medium · Confirmed
  - **Problem:** `InitialEffectiveTTL()` returns 0 (TODO #1090); all requests share one default TTL regardless of priority or unavailability regime. Transient 503 and sustained saturation use the same deadline.
  - **Proposed approach:** Plumb per-request effective TTL from priority + outcome band through `Admit` -> `EnqueueAndWait`; gate behind config flag; add flow-control tests per band.
  - **Impact:** High-priority requests get appropriate deadlines; prevents bulk traffic from consuming priority budget.
  - **Rough effort:** Medium — touches flow-control admission + queue path; needs per-band tests.
  <!-- fp: llm-d/llm-d-router:feature:per-request-queue-wait-ttl-bands -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2183
  - code: https://github.com/llm-d/llm-d-router/blob/f56f3bd9cd40300d55d8756b5d09f53ce0dc666a/pkg/epp/requestcontrol/admission.go#L200
- **RFC: header-phase-profile-handler — header-driven scheduling, secondary profiles, conditional execution** · Medium · Speculative
  - **Problem:** Scheduling profile selection is hardcoded in profile handler implementations; adding new strategies (EPD roles, A/B) requires code changes. The conditional-decode gate in `director.go` is a separate hardcoded mechanism.
  - **Proposed approach:** Declarative profile handler that reads profile selection from request headers; supports secondary profiles and conditional execution. Generalizes `disagg-profile-handler` and the conditional-decode gate into one mechanism.
  - **Impact:** Eliminates code changes for new routing strategies; unifies EPD role selection and A/B scheduling under a single declarative API.
  - **Rough effort:** Medium — design review against `scheduler_profile.go` + disagg-profile-handler contracts; implementation is plugin-layer.
  <!-- fp: llm-d/llm-d-router:feature:header-phase-profile-handler-rfc -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2178
- **Native DisaggregatedSet CRD for encode/prefill/decode role-set routing** · Medium · Speculative
  - **Problem:** EPD topologies require ad-hoc profile-handler wiring; no first-class CRD for role-set routing.
  - **Proposed approach:** DisaggregatedSet CRD defining encode/prefill/decode role sets; scheduler selects role endpoints declaratively.
  - **Impact:** First-class support for EPD topologies; reduces configuration complexity.
  - **Rough effort:** High — CRD design, scheduler integration, sidecar coordination.
  <!-- fp: llm-d/llm-d-router:issue:native-disaggregatedset-routing -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2143
- **KV events silently dropped during ZMQ replay cooldown are unmetricized, hiding prefix-cache index data loss from operators** · Low · Likely
  - **Problem:** When the ZMQ subscriber detects a sequence gap but `canAttemptReplay()` is false (within the 30s `replayCooldown` after a prior failure), it drops the live event with only a `debugLogger.Info` — no `metrics.*` counter is incremented. The same silent drop occurs for the join-mid-stream case when replay is in cooldown. These dropped events are KV-cache residency updates that the prefix-cache scorer relies on; losing them creates stale/missing residency for the affected pods until the next successful replay, silently degrading cache-aware routing accuracy. Every other failure mode in this file increments a `metrics.ZMQErrors` or `metrics.MessagesReceived` series, so these two paths are the only unobservable data-loss exits.
  - **Proposed approach:** Add a `metrics.ZMQEventsDroppedCooldown` counter (labeled by `podIdentifier`, like the existing `ZMQErrors`) incremented at both drop sites (zmq_subscriber.go:211 and :224).
  - **Impact:** Operators can alert on prefix-cache event loss and correlate it with routing-quality regressions; prefix-cache-aware routing accuracy becomes observable.
  - **Rough effort:** Low — counter definition in metrics.go + two increment calls.
  <!-- fp: llm-d/llm-d-router:feature:zmq-replay-cooldown-dropped-events-metric -->
  - code: https://github.com/llm-d/llm-d-router/blob/dd65342ee538ab76d4b9f7e1efb5d8e95385d06e/pkg/kvevents/zmq_subscriber.go#L207-L211
  - code: https://github.com/llm-d/llm-d-router/blob/dd65342ee538ab76d4b9f7e1efb5d8e95385d06e/pkg/kvevents/zmq_subscriber.go#L222-L224

## Recently Resolved
