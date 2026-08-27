---
repo: llm-d/llm-d-router
last_updated: 2026-08-27
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
- **Data race in TestFlowControlAdmissionController_Admit under -race (parallel sub-tests share reqCtx)** · Medium · Confirmed — Give each sub-test its own `reqCtx` via a `newReqCtx()` helper inside each `t.Run` closure, matching sibling `TestFlowControlAdmissionController_StampsQueueDuration`  <!-- fp: llm-d/llm-d-router:issue:flowcontrol-admission-test-data-race -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2543
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/requestcontrol/admission.go#L185-L186
- **flowcontrol: no startup warning when refresh-metrics-interval exceeds a detector's staleness threshold** · Medium · Confirmed — After plugin defaulting and refresh-interval floor clamp, log a startup warning naming the detector and both effective values when `refreshMetricsInterval > metricsStalenessThreshold`  <!-- fp: llm-d/llm-d-router:issue:flowcontrol-refresh-exceeds-staleness-no-warning -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2514
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/server/options.go
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/flowcontrol/saturationdetector/utilization/config.go
- **Add OTel traces to prefix-cache filter decisions (metrics-only today)** · Medium · Confirmed — Add a `tracer.Start` span in `Filter` emitting decision outcome, affinity threshold, sticky/non-sticky counts, and TTFT gate result as span attributes, mirroring the disagg handler's `llm_d.epp.*` attribute scheme  <!-- fp: llm-d/llm-d-router:issue:otl-traces-prefix-cache-filter-decisions -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2539
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/scheduling/filter/prefixcacheaffinity/plugin.go#L163-L214
- **Custom pod readinessGates not honored in router endpoint discovery** · Medium · Confirmed — Add a config field (e.g. on the endpoint discovery source) listing required `readinessGates` condition types; treat a missing/False condition as not-ready. No code matches for `readinessGates` anywhere in the repo — capability is genuinely absent.  <!-- fp: llm-d/llm-d-router:issue:custom-pod-readinessgates-endpoint-discovery -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2541
- **Helm --wait never completes for multi-replica active-passive EPP** · High · Confirmed — Decide supported contract: leader-counted readiness gate/PDB strategy so Helm treats leader+warm-standby as healthy, or document `--wait` incompatibility with a no-wait + leader-Ready-Service health gate. Do not route Service traffic to standbys before election.  <!-- fp: llm-d/llm-d-router:issue:helm-wait-active-passive-multi-replica -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2568
- **Add tracing to the coordinator (underspecified)** · Medium · Confirmed — Scope before implementing: root span around `pipeline.Execute` in `handleInference` plus per-step child spans in `pkg/coordinator/steps`, reusing `pkg/common/observability/tracing`; confirm correlation with EPP `request`/`request_orchestration` spans.  <!-- fp: llm-d/llm-d-router:issue:coordinator-tracing -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2578

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
- **inflightload gauges use raw client-supplied fairness_id label, bypassing boundFairnessID cardinality guard** · High · Confirmed — Route the inflightload `fairnessID` through `metrics.BoundFairnessID(...)` at all four `WithLabelValues` call sites (PreRequest Inc/Add, OnEvicted Sub/Dec, releaseTokensEarly Sub) so paired increment/decrement stay balanced under the same bounded value  <!-- fp: llm-d/llm-d-router:bug:inflightload-fairness-id-label-unbounded-cardinality -->
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L410
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L456-L457
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/metrics.go#L32
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/metrics.go#L41
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/metrics/cardinality.go#L100-L112
- **inflightload GaugeVec series leaked on endpoint deletion (cardinality leak; issue's value-drift claim contradicted by OnEvicted)** · High · Confirmed — Prune the series in the `EventDelete` branch of `Extract` using `DeletePartialMatch(prometheus.Labels{"endpoint_name": name})` on both vectors; add a registry-scrape regression test asserting zero series for a deleted endpoint after in-flight requests drain. Open PR #2577 targets this — verify it prunes via `DeletePartialMatch` and threads the endpoint *name* (not just ID) into the delete path.  <!-- fp: llm-d/llm-d-router:bug:inflightload-gaugevec-series-leaked-on-endpoint-deletion -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2529
  - issue: https://github.com/llm-d/llm-d-router/pull/2577
  - code: https://github.com/llm-d/llm-d-router/blob/d32484978d1068c6ef8bc9d525bd10181c58b40e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L710-L713
  - code: https://github.com/llm-d/llm-d-router/blob/d32484978d1068c6ef8bc9d525bd10181c58b40e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L351-L353
  - code: https://github.com/llm-d/llm-d-router/blob/d32484978d1068c6ef8bc9d525bd10181c58b40e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/metrics.go#L32
  - code: https://github.com/llm-d/llm-d-router/blob/d32484978d1068c6ef8bc9d525bd10181c58b40e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/metrics.go#L41
  - code: https://github.com/llm-d/llm-d-router/blob/d32484978d1068c6ef8bc9d525bd10181c58b40e/pkg/epp/metrics/metrics.go#L1
- **InitTracing silently substitutes 10% sampling for unsupported sampler values and unparseable args** · Medium · Confirmed — Map each standard `OTEL_TRACES_SAMPLER` value to its SDK constructor in a `newSampler` helper, route parse and range failures through the error handler, reject ratios outside `[0,1]`; add table-driven tests asserting `sampler.Description()` per value  <!-- fp: llm-d/llm-d-router:bug:otel-sampler-fallback-silently-10pct -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2530
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/common/observability/tracing/telemetry.go#L76-L86
- **Attributes.Clone() shares value references despite "deep copy" contract** · Low · Likely — Make `Put` clone the value (or have `Clone` call `v.Clone()` before re-`Put`) and add a test that mutating a stored value after `Clone` does not affect the clone; confirm no in-tree producer mutates a stored `Cloneable` in place. `Get` clones on read, mitigating reader aliasing.  <!-- fp: llm-d/llm-d-router:bug:attributemap-clone-is-shallow -->
  - code: https://github.com/llm-d/llm-d-router/blob/e1ca56b1d6baf7ea4f9b2ac20ef4af7ba6922f6f/pkg/epp/framework/interface/datalayer/attributemap.go#L62
  - code: https://github.com/llm-d/llm-d-router/blob/e1ca56b1d6baf7ea4f9b2ac20ef4af7ba6922f6f/pkg/epp/framework/interface/datalayer/attributemap.go#L92

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
- **estimateRequestTokens / EstimateInput recomputed in PreRequest after Produce already computed and stored it** · Low · Likely — In `PreRequest`, read the stored `UncachedRequestTokens` attribute via `endpoint.Get(p.uncachedRequestTokensDk)` instead of recomputing `estimateRequestTokens`; confirm equality with a test before relying on it  <!-- fp: llm-d/llm-d-router:perf:inflightload-redundant-estimate-request-tokens-produce-prerequest -->
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L372
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L380
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L409
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L452
- **prefix-cache-affinity filter uses locked global math/rand on the per-request scheduling hot path** · Low · Likely — Replace `math/rand` import with `math/rand/v2` (lock-free, per-P state) as a drop-in; add a parallel benchmark confirming the exploration path no longer contends  <!-- fp: llm-d/llm-d-router:perf:prefixcacheaffinity-global-rand-lock-on-filter-hotpath -->
  - code: https://github.com/llm-d/llm-d-router/blob/d32484978d1068c6ef8bc9d525bd10181c58b40e/pkg/epp/framework/plugins/scheduling/filter/prefixcacheaffinity/plugin.go#L166-L180
- **InMemoryIndex.Lookup scans all request prefix blocks without early-stopping on a cache miss** · Medium · Likely — Track the longest contiguous hit run from the start and stop once a miss ends the viable prefix chain (or cap the scan at a configurable max-prefix-blocks); consider iterating `pods.cache.Range` instead of `Keys()` to avoid the per-key slice allocation. The only early exit is a found-but-empty `PodCache`; a not-found key `continue`s, so the loop runs to completion for long-context/agentic prompts.  <!-- fp: llm-d/llm-d-router:perf:kvblock-lookup-no-early-stop-on-miss -->
  - code: https://github.com/llm-d/llm-d-router/blob/e1ca56b1d6baf7ea4f9b2ac20ef4af7ba6922f6f/pkg/kvcache/kvblock/in_memory.go#L109
  - code: https://github.com/llm-d/llm-d-router/blob/e1ca56b1d6baf7ea4f9b2ac20ef4af7ba6922f6f/pkg/kvcache/kvblock/in_memory.go#L127

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
- **Generalize boundFairnessID/boundModel into a bounded-label wrapper enforced for all producer-owned metric vectors** · Medium · Confirmed
  - **Problem:** The project's `boundedLabel` cardinality guard (cap 1000, collapse to "other") is opt-in per call site in `pkg/epp/metrics`, but producer-owned GaugeVecs in `pkg/epp/framework/plugins/...` call `WithLabelValues` with raw client-derived values directly — the inflightload producer bypasses the guard entirely (see the High-severity bug above). Any new producer metric can reintroduce the same unbounded-cardinality defect.
  - **Proposed approach:** Propose a `BoundedGaugeVec`/`BoundedCounterVec` wrapper (or a `WithBoundedLabelValues` helper) that producers must use for any label fed by request data, so the cardinality cap is enforced at the metric-construction boundary. Migrate inflightload as the first consumer and add a lint/test guard that no producer-owned `*Vec` calls raw `WithLabelValues` on a client-derived label.
  - **Impact:** Makes the cardinality cap structural rather than convention-dependent; prevents memory-exhaustion DoS from client-supplied labels.
  - **Rough effort:** Low-Medium — wrapper type + migrate inflightload + lint guard.
  <!-- fp: llm-d/llm-d-router:feature:unified-bounded-label-wrapper-for-producer-metrics -->
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/metrics/cardinality.go#L100-L112
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/producer.go#L456-L457
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/metrics.go#L32
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/requestcontrol/dataproducer/inflightload/metrics.go#L41
- **Establish an OTel span convention for all scheduling filter/scorer decisions, using the disagg handler as the template** · Low · Likely
  - **Problem:** The disagg `HeadersHandler.PreRequest` opens a `prepare_disaggregation` span and emits structured `llm_d.epp.*` attributes for every decision branch, but the scheduling filters and scorers (e.g. `prefix-cache-affinity-filter`) record outcomes only as Prometheus counter increments with no span — the per-request routing rationale (why sticky was kept or broken, TTFT gate margin) is absent from traces. Issue #2539 requests this for the prefix-cache filter specifically.
  - **Proposed approach:** Define a span convention (span name per plugin type, standard attribute keys for outcome, thresholds, and candidate counts before/after) in the scheduling plugin interface; add spans to `Filter`/`Score` implementations starting with `prefix-cache-affinity-filter`. Gate behind the existing `--tracing` flag so disabled deployments pay no cost.
  - **Impact:** Consistent debugging surface across the whole scheduling pipeline; per-request routing rationale visible in traces.
  - **Rough effort:** Low-Medium — interface change + implement for each filter/scorer.
  <!-- fp: llm-d/llm-d-router:feature:otlp-spans-for-all-scheduling-filter-scorer-decisions -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2539
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/scheduling/filter/prefixcacheaffinity/plugin.go#L163-L214
  - code: https://github.com/llm-d/llm-d-router/blob/149447dd6b9bbb00cd1c9169b07854332cdbb87e/pkg/epp/framework/plugins/scheduling/profilehandler/disagg/disagg_headers_handler.go#L91-L96
- **Support ORCA endpoint-load-metrics response headers for flow-control pressure** · Medium · Confirmed
  - **Problem:** Upstream control planes cannot pace traffic without polling Prometheus; vLLM already supports ORCA but the EPP does not emit `endpoint-load-metrics` response headers. Exposing pool saturation / queue pressure via ORCA would close the feedback loop without requiring out-of-band scraping.
  - **Proposed approach:** Determine where in the ext-proc response path headers can be injected (likely in the post-scheduling response mutation), map flow-control signals (pool saturation, queue depth, admission status) to ORCA `application_utilization` / `queue_length` fields, and wire them through. Requires coordination with upstream envoy/gRPC-proxy ORCA header propagation.
  - **Impact:** Enables real-time traffic pacing from upstream control planes; reduces tail latency spikes from over-admission during saturation.
  - **Rough effort:** Medium-High — ext-proc response-path discovery, ORCA field mapping, upstream propagation testing.
  <!-- fp: llm-d/llm-d-router:issue:orca-load-metrics-response-headers -->
  - issue: https://github.com/llm-d/llm-d-router/issues/2536
- **Optional data-dependency consumers with no producer load silently and score 0.0 — validate at config load** · Medium · Confirmed
  - **Problem:** `CreateMissingDataProducers` errors only when a *Required* consumed key has no producer; for *Optional* keys it logs a one-line warning and continues. Scorers/filters that declare `Optional` dependencies (e.g. `EndpointAttributeScorer`) silently load, run, and score every endpoint 0.0 when their producer is misconfigured or absent — indistinguishable from a scorer that discriminates badly. A benchmark built on such a config would report a meaningless result with no error.
  - **Proposed approach:** Add a config-load validation pass that, for scorers/filters whose Optional dependency has no producer, emits a loud persistent signal (not a single INFO line) or fails; consider per-plugin opt-in to promote specific Optional dependencies to Required so misconfiguration is caught before the first request.
  - **Impact:** Prevents silent routing degradation from misconfigured data-dependency graphs; makes data-graph completeness observable.
  - **Rough effort:** Low-Medium — validation pass + optional per-plugin promotion flag.
  <!-- fp: llm-d/llm-d-router:feature:validate-optional-data-dependency-producer-at-config-load -->
  - code: https://github.com/llm-d/llm-d-router/blob/d32484978d1068c6ef8bc9d525bd10181c58b40e/pkg/epp/datalayer/data_graph.go#L129-L140
  - code: https://github.com/llm-d/llm-d-router/blob/d32484978d1068c6ef8bc9d525bd10181c58b40e/pkg/epp/datalayer/data_graph.go#L97

## Recently Resolved
