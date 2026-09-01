---
title: "BenchFlow distributed tracing: instrumentation, collection, metrics, and reports"
date: 2026-09-01
type: learning
topic: BenchFlow distributed tracing
status: implemented
platforms:
  - llm-d
  - RHOAI 3.5
validated_run_id: 35a854d975a7427fb17b7ef64075f8d5
validated_run_url: "https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/377/runs/35a854d975a7427fb17b7ef64075f8d5?workspace=benchflow"
implementation_pr: "https://github.com/albertoperdomo2/benchflow/pull/14"
---

# BenchFlow distributed tracing: instrumentation, collection, metrics, and reports

## Executive summary

BenchFlow tracing is enabled by the **metrics profile**, but it instruments the complete BenchFlow-managed deployment. When tracing is enabled, BenchFlow:

1. installs a shared OpenTelemetry Collector and Jaeger backend in the experiment namespace;
2. injects release-scoped OpenTelemetry configuration into vLLM, EPP, and the llm-d P/D routing proxy when present;
3. records spans during the benchmark window;
4. queries Jaeger after the benchmark, normalizes the spans, and saves them as ordinary BenchFlow artifacts;
5. generates a standalone HTML distribution report before uploading the complete artifact directory to MLflow.

The trigger is deliberately profile-based:

```yaml
apiVersion: benchflow.io/v1alpha1
kind: MetricsProfile
metadata:
  name: detailed-tracing
spec:
  tracing:
    mode: detailed
    sample_ratio: 1.0
```

Tracing is **off by default**. The default `sample_ratio` is `0.1`, but it has no effect while `mode: off`.

The validated RHOAI smoke run produced 10 traces, 60 spans, two services, 10 complete multi-service trees, and 23 numeric metric series. See [MLflow run 35a854d975a7427fb17b7ef64075f8d5](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/377/runs/35a854d975a7427fb17b7ef64075f8d5?workspace=benchflow).

## Mental model

```text
MetricsProfile tracing configuration
        |
        v
BenchFlow platform setup
  OpenTelemetry Collector + Jaeger
        |
        v
BenchFlow deployment rendering
  vLLM + EPP + optional P/D routing proxy
        |
        | OTLP/gRPC spans on port 4317
        v
OpenTelemetry Collector
  drop metrics-scraping noise -> batch -> Jaeger
        |
        v
Jaeger query API on port 16686
        |
        | benchmark start/end window + release-scoped service names
        v
BenchFlow artifact collection
  services.json + traces.jsonl.gz + trace-summary.json
        |
        v
BenchFlow post-run reporting
  full_run_artifacts_report.html
  trace_distribution_report.html
        |
        v
MLflow artifact upload
```

The OpenTelemetry plane is a namespace-level shared service. Individual model deployments remain distinguishable because every instrumented component gets a service name prefixed by the BenchFlow release name.

## Public configuration surface

Tracing belongs to `MetricsProfile.spec.tracing`.

| Field | Accepted values | Default | Meaning |
|---|---:|---:|---|
| `mode` | `off`, `standard`, `detailed` | `off` | Enables tracing and controls whether vLLM detailed model-phase spans/attributes are requested. |
| `sample_ratio` | number from 0.0 through 1.0 | `0.1` | Ratio used by the parent-based trace-ID-ratio sampler. |

### Modes

- **`off`**: no tracing plane is required, no tracing flags or environment variables are injected, and trace collection returns `{"enabled": false}`.
- **`standard`**: exports ordinary component spans, but does not pass vLLM `--collect-detailed-traces=all`.
- **`detailed`**: exports ordinary spans and asks vLLM to emit detailed model-phase tracing with `--collect-detailed-traces=all`.

The sampler is always:

```text
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=<sample_ratio>
```

This exact sampler name matters for llm-d components. The smoke profile uses `sample_ratio: 1.0` to retain every trace during validation. A production profile should normally begin around `0.1`, then increase only for focused debugging or staging measurements.

### Current support boundary

Tracing requires a **BenchFlow-managed deployment**. An experiment using an existing target URL is rejected because BenchFlow cannot safely instrument an arbitrary external deployment.

Supported deployment paths are:

- **llm-d v0.9.0 or newer**, using the router-chart recipe layout;
- **RHOAI 3.5**, only through deployment profile `rhoai-distributed-default-tracing`.

RHOAI support is intentionally narrow. The special profile supplies an explicit OpenShift AI 3.5 default `EndpointPickerConfig`, allowing BenchFlow to inject EPP tracing configuration. Ordinary RHOAI deployment profiles are rejected when paired with a tracing-enabled metrics profile.

## Tracing plane installation

When any plan in the execution requires tracing, platform setup calls `ensure_tracing_plane`. It applies the resources in:

```text
src/benchflow/assets/setup/llmd/tracing.yaml
```

The plane contains:

- `benchflow-otel-collector` Deployment and Service;
- `benchflow-jaeger` Deployment and Service;
- an OpenTelemetry Collector ConfigMap.

Current component images:

- OpenTelemetry Collector Contrib `0.120.0`;
- Jaeger `2.15.0`.

The Collector accepts:

- OTLP/gRPC on port `4317`;
- OTLP/HTTP on port `4318`.

It filters spans generated by `/metrics` scraping, batches the remaining spans, and exports them to Jaeger over OTLP/gRPC. The filter removes spans matching `url.path=/metrics`, `http.route=/metrics`, `GET /metrics`, or the generic `GET` name used by this scraping path.

Jaeger's query API is exposed internally on port `16686`. The tracing plane is shared within the namespace and is removed when the platform is reset.

## Component instrumentation

### Common OpenTelemetry environment

Every instrumented component receives:

```text
OTEL_SERVICE_NAME=<release>-<role>
OTEL_EXPORTER_OTLP_ENDPOINT=http://benchflow-otel-collector.<namespace>.svc.cluster.local:4317
OTEL_TRACES_EXPORTER=otlp
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=<sample_ratio>
OTEL_RESOURCE_ATTRIBUTES=benchflow.release=...,benchflow.experiment=...,benchflow.model=...,benchflow.component.role=...
```

The explicit `OTEL_SERVICE_NAME` is essential. Without it, engine spans appear as `unknown_service`, making working instrumentation look broken and preventing reliable release-scoped collection.

### vLLM

BenchFlow removes any pre-existing tracing flags and injects:

```text
--otlp-traces-endpoint=http://benchflow-otel-collector.<namespace>.svc.cluster.local:4317
```

For `mode: detailed`, it also injects:

```text
--collect-detailed-traces=all
```

Typical service roles are `vllm-modelserver`, `vllm-decode`, or another release-specific role selected by the deployment topology.

### llm-d EPP

For the llm-d router chart, BenchFlow sets:

```yaml
router:
  tracing:
    enabled: true
    otelExporterEndpoint: http://benchflow-otel-collector.<namespace>.svc.cluster.local:4317
    sampling:
      sampler: parentbased_traceidratio
      samplerArg: "<sample_ratio>"
```

It also injects the release-scoped EPP service name and resource attributes.

### llm-d P/D routing proxy

The router-chart EPP tracing switch does not configure the P/D routing sidecar. BenchFlow patches that container separately with:

```text
--tracing=true
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_ENDPOINT=<collector endpoint>
OTEL_SERVICE_NAME=<release>-routing-proxy
```

Both the flag and exporter are necessary. Without `--tracing=true`, proxy spans are absent. Without `OTEL_TRACES_EXPORTER=otlp`, spans can be printed to stdout instead of reaching the Collector.

### RHOAI EPP

RHOAI tracing uses the explicit `rhoai-distributed-default-tracing` profile. It carries the OpenShift AI 3.5 default `EndpointPickerConfig` and the marker:

```yaml
options:
  tracing_provider: explicit-epp
```

That explicit scheduler template lets BenchFlow add the EPP `--tracing=true` flag and OTLP environment. This is currently the only supported RHOAI tracing path.

## Collection after the benchmark

Trace collection occurs during the normal BenchFlow artifact-collection stage, after the benchmark timestamps are known.

The collector:

1. requires benchmark start and end timestamps;
2. waits five seconds for batch span processors to flush final spans;
3. queries Jaeger `/api/services`;
4. keeps only service names beginning with `<release-name>-`;
5. queries `/api/traces` for every selected service within the benchmark window;
6. recursively splits a query window when a Jaeger response reaches the 1,000-trace limit;
7. deduplicates results by trace ID;
8. normalizes every span into a stable JSON representation;
9. counts traces containing more than one service as complete multi-service traces.

Collection failure does not hide the rest of the benchmark artifacts. BenchFlow writes a summary with `status: unavailable` and the error message, then continues artifact handling.

## Trace artifacts

All trace artifacts are stored below `traces/`.

| Artifact | Contents |
|---|---|
| `traces/services.json` | Release-scoped service names discovered in Jaeger. |
| `traces/traces.jsonl.gz` | One normalized span per gzip-compressed JSONL record. |
| `traces/trace-summary.json` | Collection status, backend, tracing mode, sample ratio, benchmark window, trace/span counts, service list, and complete-tree count. |
| `metadata.json` | The normal BenchFlow artifact metadata, including the embedded trace summary. |
| `reports/trace_distribution_report.html` | Standalone tracing distribution report generated after collection. |
| `reports/full_run_artifacts_report.html` | The ordinary benchmark post-run report. |

The normalized span records contain:

- `trace_id`;
- `span_id`;
- `parent_span_id`;
- `service_name`;
- `operation_name`;
- start and end times in Unix microseconds;
- `duration_microseconds`;
- OpenTelemetry status;
- span tags;
- span logs;
- process/resource tags.

These artifacts are uploaded to the same MLflow run as the benchmark outputs, metrics, logs, metadata, and rendered manifests.

## Metric provenance

The tracing report discovers numeric metric series from two sources.

### 1. Span durations

Every distinct `service_name + operation_name` pair becomes a duration series. BenchFlow reads `duration_microseconds` and converts it to milliseconds.

The report labels this provenance as:

```text
source: span duration · <operation_name>
```

This value is calculated from the span duration recorded by the tracing backend.

### 2. OpenTelemetry span attributes

Every finite numeric span attribute becomes a series. Numeric values stored inside JSON arrays are flattened; this supports attributes such as EPP top-score arrays.

The report labels this provenance as:

```text
source: span attribute · <attribute_name>
```

This means the value was emitted directly by component instrumentation, not derived by BenchFlow from span timestamps.

Conversions are:

- `gen_ai.latency.*`: seconds to milliseconds;
- span duration: microseconds to milliseconds;
- tokens, endpoint counts, scores, priorities, and request parameters: unchanged.

Booleans and non-numeric strings are ignored.

## Metrics observed in the validated RHOAI smoke run

The following inventory is evidence from run `35a854d975a7427fb17b7ef64075f8d5`. It describes what the tested vLLM and EPP versions emitted; it is not a promise that every version or topology emits every attribute.

### Detailed vLLM latency attributes

| Attribute | Report meaning | Unit |
|---|---|---:|
| `gen_ai.latency.e2e` | End-to-end request latency | ms |
| `gen_ai.latency.time_to_first_token` | Time to first token | ms |
| `gen_ai.latency.time_in_queue` | Queue wait | ms |
| `gen_ai.latency.time_in_model_prefill` | Model prefill phase | ms |
| `gen_ai.latency.time_in_model_decode` | Model decode phase | ms |
| `gen_ai.latency.time_in_model_inference` | Total model inference phase | ms |

### Token and request attributes

| Attribute | Meaning | Unit |
|---|---|---:|
| `gen_ai.usage.prompt_tokens` | Prompt-token count | tokens |
| `gen_ai.usage.completion_tokens` | Completion-token count | tokens |
| `gen_ai.request.max_tokens` | Requested maximum output | tokens |
| `gen_ai.request.n` | Requested completion count | count |
| `gen_ai.request.temperature` | Sampling temperature | value |
| `gen_ai.request.top_p` | Top-p threshold | value |
| `request_prio` | Request priority | priority |

### EPP scheduling attributes

| Attribute | Meaning | Unit |
|---|---|---:|
| `llm_d.epp.filter.candidate_endpoints` | Endpoints entering filtering | endpoints |
| `llm_d.epp.filter.filtered_endpoints` | Endpoints removed or represented by the filter stage | endpoints |
| `llm_d.epp.picker.candidate_endpoints` | Endpoints considered by the picker | endpoints |
| `llm_d.epp.picker.top_scores` | Numeric endpoint-selection scores, flattened from the emitted JSON array | score |

### Span-operation durations

The validated run contained 10 observations for each of these six operations:

| Service | Operation |
|---|---|
| EPP | `filter_endpoints` |
| EPP | `gateway.request` |
| EPP | `gateway.request_orchestration` |
| EPP | `pick_endpoints` |
| EPP | `run_scheduler_profile` |
| vLLM model server | `llm_request` |

Together, the 17 numeric attribute series and six operation-duration series produced the 23 report panels.

## Distribution report

Post-run report orchestration lives under `src/benchflow/reports/`. The benchmark report and tracing report are attempted independently, so one unavailable input does not suppress the other.

For every discovered numeric series, the tracing report shows:

- observation count;
- minimum and maximum;
- P25, P50, P75, P90, P95, and P99;
- arithmetic mean;
- sample standard deviation;
- coefficient of variation, $CV = \frac{s}{|\bar{x}|}$;
- metric unit;
- provenance: span duration or OpenTelemetry span attribute;
- normalized histogram overlaid with a Gaussian-kernel density estimate;
- empirical cumulative distribution function.

The kernel-density estimate is omitted for singleton or constant-valued series. The ECDF remains meaningful for those series.

The standalone report uses two metric panels per row and writes to:

```text
reports/trace_distribution_report.html
```

It is generated after trace collection and before the artifact directory is uploaded to MLflow.

## MLflow metadata

When tracing is enabled, the RunPlan adds:

```text
tracing_mode=<standard|detailed>
tracing_sample_ratio=<ratio>
```

These tags make traced runs discoverable independently of artifact inspection. The raw trace artifacts and both HTML reports remain under the run's MLflow artifacts.

## Smoke experiments

llm-d:

```text
experiments/smoke/qwen3-06b-llm-d-tracing-smoke.yaml
```

RHOAI 3.5:

```text
experiments/smoke/qwen3-06b-rhoai-distributed-default-tracing-smoke.yaml
```

The RHOAI experiment uses:

```yaml
deployment_profile: rhoai-distributed-default-tracing
benchmark_profile: aiperf-smoke
metrics_profile: detailed-tracing
```

During development, the tracing PR image is:

```text
ghcr.io/albertoperdomo2/benchflow:tracing
```

## Operational checks

A healthy traced run should show:

1. Collector and Jaeger Deployments ready in the BenchFlow namespace.
2. Release-scoped service names in `traces/services.json`.
3. `trace-summary.json` with `status: collected`.
4. A non-zero span count.
5. At least one multi-service trace when requests traverse EPP and vLLM.
6. `reports/trace_distribution_report.html` in MLflow.
7. Expected component roles in `OTEL_SERVICE_NAME`, rather than `unknown_service`.

If spans are missing:

- verify the metrics profile has `tracing.mode` other than `off`;
- verify the deployment is BenchFlow-managed;
- verify the platform/profile combination is supported;
- inspect vLLM arguments for `--otlp-traces-endpoint`;
- inspect detailed runs for `--collect-detailed-traces=all`;
- inspect EPP or proxy arguments for `--tracing=true`;
- inspect `OTEL_TRACES_EXPORTER`, `OTEL_EXPORTER_OTLP_ENDPOINT`, and `OTEL_SERVICE_NAME`;
- verify Collector and Jaeger readiness and services;
- inspect `trace-summary.json.error`;
- remember that a sampling ratio below 1.0 intentionally omits some root traces.

## Limitations and interpretation

- Trace metrics are event samples, not continuous Prometheus time series.
- The report describes the sampled requests captured in the benchmark window.
- A low sample ratio reduces report sample size and can produce incomplete-looking distributions.
- Attribute availability depends on component version, tracing mode, and request path.
- `detailed` enables vLLM model-phase tracing; it does not guarantee that every possible attribute exists.
- The current distribution report analyzes individual metric series. It does not yet render per-trace waterfall trees or calculate cross-span critical paths.
- Jaeger is an ephemeral collection backend for the run workflow; MLflow artifacts are the durable result.
- Prometheus queries in `detailed-tracing` remain separate from trace-derived distributions. GPU utilization, memory, CPU, running requests, and waiting requests come from Prometheus, not OpenTelemetry spans.

## Implementation locations

| Responsibility | Repository location |
|---|---|
| Tracing model and profile parsing | `src/benchflow/models.py`, `src/benchflow/loaders.py` |
| RunPlan validation and MLflow tags | `src/benchflow/plans.py` |
| Collector/Jaeger lifecycle and collection | `src/benchflow/tracing.py` |
| Collector and Jaeger resources | `src/benchflow/assets/setup/llmd/tracing.yaml` |
| llm-d instrumentation | `src/benchflow/deploy/llmd.py` |
| RHOAI deployment rendering | `src/benchflow/renderers/deployment.py` |
| Artifact collection | `src/benchflow/artifacts.py` |
| Post-run report orchestration | `src/benchflow/reports/post_run.py` |
| Trace distribution report | `src/benchflow/reports/tracing.py` |
| MLflow upload trigger | `src/benchflow/mlflow_upload.py` |

## Future reporting refactor

The `benchflow.reports` package now owns post-run report orchestration and trace rendering. Comparison-report parsing and rendering are still distributed across benchmark backends. The intended follow-up is to separate benchmark-result normalization from presentation, then move comparison-report orchestration and rendering into `benchflow.reports` as well.