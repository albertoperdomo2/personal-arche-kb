---
title: "Experiment and comparison report template"
date: "2026-07-29"
type: "template"
topic: "Performance, experiment, and comparison reports"
---

# Experiment and comparison report template

Use this template for performance reports, single-experiment reports, and multi-configuration comparison reports. Read the instructions before fetching artifacts, analyzing metrics, choosing plots, or writing the report.

## Instructions for the report author

### Naming and revision rules

Every report title and filename must begin with the report date:

```text
YYYY-MM-DD - <model, mechanism, or workload> - <comparison or experiment>.md
```

Examples:

```text
2026-07-29 - Qwen3.6-35B-A3B - KV-cache offload comparison.md
2026-07-29 - Llama-3.3-70B - concurrency scaling experiment.md
```

Do not overwrite an existing report unless the user explicitly asks for an update to that report.

If the desired filename already exists, create the next revision:

```text
2026-07-29 - Qwen3.6-35B-A3B - KV-cache offload comparison (revision 1).md
2026-07-29 - Qwen3.6-35B-A3B - KV-cache offload comparison (revision 2).md
```

Use a revision when rerunning the same comparison, replacing failed runs, or producing a report whose natural title collides with an existing report. Preserve the earlier report and link the revisions to one another.

If the report needs companion articles, create a folder with exactly the same base name as the report:

```text
2026-07-29 - Qwen3.6-35B-A3B - KV-cache offload comparison.md
2026-07-29 - Qwen3.6-35B-A3B - KV-cache offload comparison/
  01 - Request-level comparison.md
  02 - Session-level comparison.md
  03 - Mechanism and Prometheus comparison.md
  04 - Per-configuration diagnostics.md
```

### Frontmatter rules

Use plain YAML. Every value must be either:

- A double-quoted string
- A number

Do not use YAML arrays, nested objects, unquoted dates, booleans, `null`, aliases, tags, or multiline scalars. Use `"unknown"`, `"not applicable"`, or `"varies by configuration; see configuration table"` when a scalar value is unavailable or differs across runs.

Copy this frontmatter and replace every placeholder:

```yaml
---
title: "YYYY-MM-DD - <report title>"
date: "YYYY-MM-DD"
type: "experiment-report"
topic: "<research topic>"
experiment: "<experiment name>"
report_revision: 0
model: "<model name>"
model_revision: "<model revision or unknown>"
runtime_image: "<container image>"
vllm_version: "<version>"
gpu_type: "<GPU type>"
gpu_count: 0
tensor_parallelism: 0
pipeline_parallelism: 0
replicas: 0
gpu_memory_utilization: 0.0
max_model_len: 0
max_num_seqs: 0
concurrency: 0
cpu_bytes: 0
offload_spec: "<offload specification>"
secondary_tier: "<tier or varies by configuration>"
secondary_tier_threads: "<thread configuration or not applicable>"
shared_memory: "<size>"
workload: "<benchmark and profile>"
random_seed: 0
duration_seconds: 0
cache_cleaning_state: "<clean, warm, unknown, or varies>"
baseline: "<baseline configuration>"
configuration_count: 0
---
```

Use numeric zero only as a template placeholder. In a finished report, do not leave a zero that actually means “unknown”; replace it with the quoted string `"unknown"`.

### Analysis rules

Before writing:

1. Download and inspect every run's artifacts.
2. Verify configuration fingerprints, model/runtime versions, workload shape, seed, duration, cache state, node placement, and relevant storage or network topology.
3. Inventory request-level, session-level, native benchmark, Prometheus, log, and component-specific telemetry.
4. Classify each run as accepted, conditionally accepted, or rejected.
5. Declare one comparison baseline, normally `No offload` or the simplest stable configuration.
6. Declare the configuration order and color mapping once; reuse both throughout the report.
7. Preserve failed runs and show why they failed. Do not rank invalid or non-comparable runs.
8. Separate measured observations, inferences, limitations, and conclusions.
9. Never invent a missing metric or silently substitute a nearby one.

Use the finest source cadence and finest categorical grain available. Do not downsample to make a chart smaller. If the report or chart becomes too large, split it into linked companion articles.

### Configuration order and colors

Put the baseline first, followed by increasingly complex configurations. Use the same configuration order in every table, plot, legend, and prose comparison.

For the standard KV-cache offload comparison, use:

| Order | Configuration | Color |
|---:|---|---|
| 1 | No offload | `#1f77b4` |
| 2 | CPU-only offload | `#ff7f0e` |
| 3 | CPU + NVMe | `#2ca02c` |
| 4 | CPU + CephFS | `#d62728` |

For other experiments, assign colors from Category10 in configuration order and record the mapping near the start of the report. When color represents a different semantic dimension—such as read versus write—use a stable, clearly labeled mapping for that dimension instead.

### Figure rules

Prefer cross-configuration comparison figures. Keep configurations in one plot when Vega-Lite supports the view and it remains readable.

Split a figure when:

- Inline data or native samples make the article too large.
- The combined legend or number of series becomes unreadable.
- Independent run timelines cannot be compared honestly on one axis.
- Per-configuration diagnostics require different metrics or scales.

When splitting, prefer comparison-first companion articles. Use one figure per configuration only when a combined comparison would obscure the result.

Every figure must have:

1. A numbered, descriptive title: `Figure 1 — ...`
2. A sentence immediately before it explaining what it shows and naming the source artifact or metric and source cadence.
3. Axes labeled with quantity and unit.
4. A white background.
5. Consistent configuration order, legend order, scales, and colors.
6. A short paragraph immediately after it interpreting the result and stating what it does not prove.

Use grouped bars for categorical comparisons, lines for time series, stacked bars or areas for composition, points for relationships, rules for thresholds, and error bars or bands when repetitions provide uncertainty. Bar-chart y-axes must begin at zero.

Use images when an existing artifact, system diagram, or screenshot materially clarifies the result. Give each image a numbered caption, provenance, and interpretation. Do not add decorative images.

Proactively add a supported plot when it materially improves the analysis, even if the user did not name it. The report should be feature-rich and plot-rich, but not a sequence of unexplained charts.

### Metric coverage

Include a metric when it is available and relevant. Explicitly state when important telemetry is missing.

#### Cross-configuration outcome metrics

- Request throughput in requests/s
- Output-token throughput in tokens/s
- Total output tokens and output tokens per request
- Requests sent, completed, errored, cancelled, and truncated
- TTFT, inter-token latency, and end-to-end latency
- Percentiles or distributions when native samples are available
- Relative and absolute deltas versus the declared baseline

#### Session-level metrics

- Sessions sent and completed
- Sessions active at cutoff, cancelled, or errored
- Session throughput
- Session latency and completion-time distributions
- Branches or child requests sent, completed, errored, and truncated
- Branch fan-out and per-session completion where available

#### Validity and saturation metrics

- Running and waiting requests
- Queue depth and queue-growth onset
- Errors, warnings, retries, restarts, OOMs, and preemptions
- Functional failure or incomplete-session evidence
- Configuration drift and cache-cleaning state
- Benchmark cutoff behavior

#### Mechanism-specific Prometheus metrics

Choose these independently for each configuration; a metric does not need to appear for configurations where it is irrelevant.

- **CPU KV offload:** GPU KV-cache occupancy, CPU memory pressure, CPU-to-GPU and GPU-to-CPU KV-transfer bandwidth and latency, external KV reuse, local computation, scheduler pressure, and store refusals.
- **NVMe:** read/write bandwidth, read/write IOPS, operation latency, queue depth, busy time/utilization, capacity, filesystem growth, and alignment with KV transfers and request progress.
- **CephFS:** pool/client read/write bandwidth, IOPS, operation latency, PVC usage, MDS/OSD/filesystem health, mount health, slow operations, throttling, and store refusals.
- **Networked/distributed mechanisms:** transmitted/received bandwidth, packet or retransmission evidence, endpoint health, and relevant service latency.
- **Other mechanisms:** inspect the available telemetry and add the resource, saturation, capacity, and health metrics needed to explain the mechanism.

Relate mechanism metrics to request outcomes on aligned elapsed-time axes when possible. Do not overlay incompatible units merely to save space.

---

# YYYY-MM-DD - <Model / mechanism / workload> - <comparison or experiment>

## Benchmark overview

In one short paragraph, state:

- What model and mechanism are being tested.
- Which configurations are compared, in their canonical order.
- The stable parameters shared by all runs.
- The parameters intentionally varied.
- The workload/profile, concurrency, duration, and repetition count.

Example:

> This benchmark compares KV-cache offloading for `<model>` across No offload, CPU-only, CPU + NVMe, and CPU + CephFS. All runs use `<runtime>`, `<GPU/TP topology>`, `<GPU-memory utilization>`, `<max model length>`, `<workload>`, concurrency `<N>`, duration `<N seconds>`, and seed `<N>`. Only the offload tier and its associated storage settings vary.

## Executive summary

In a small number of paragraphs, answer:

- Were the runs valid and comparable?
- Were there errors, warnings, failed runs, or important observability gaps?
- Which configuration won on the primary objective?
- Which configuration lost?
- Was there practical or statistical parity?
- What is the main mechanism explaining the result?
- What is the decision or conclusion?

Do not declare parity from rounded equality alone. If there is one run per configuration, say that the values were equal or near-equal in this run and that variance is unknown.

## Validity verdict

**Verdict: Valid / Conditionally valid / Invalid / Inconclusive**

Give the concrete reason. If only some runs are valid, list the disposition of every configuration. Do not rank rejected or non-comparable runs.

## Main conclusions

- **Winner:** <configuration and metric evidence, or no defensible winner>
- **Loser:** <configuration and metric evidence, or no defensible loser>
- **Parity:** <where parity exists, how it is defined, and its uncertainty>
- **Issue:** <errors, warnings, failures, saturation, or missing telemetry>
- **Mechanism:** <measured explanation>
- **Decision:** <what should be used or tested next>

## Configuration map

Declare the order and color mapping used throughout the report.

| Order | Configuration | What changes | Color |
|---:|---|---|---|
| 1 | <Baseline> | <baseline behavior> | `#1f77b4` |
| 2 | <Configuration 2> | <difference from baseline> | `#ff7f0e` |
| 3 | <Configuration 3> | <difference from baseline> | `#2ca02c` |
| 4 | <Configuration 4> | <difference from baseline> | `#d62728` |

## Headline results

Put this table before detailed plots. Include a direct MLflow link for every configuration.

| Configuration | MLflow | Disposition | Requests/s | Δ requests/s vs baseline | Output tokens/s | Mean TTFT (s) | Mean E2E (s) | Completed sessions | Error/cancel rate | Main note |
|---|---|---|---:|---:|---:|---:|---:|---:|---:|---|
| <Baseline> | [MLflow](<direct run URL>) | Accepted |  |  |  |  |  |  |  |  |
| <Configuration 2> | [MLflow](<direct run URL>) | Accepted / Rejected |  |  |  |  |  |  |  |  |

Add columns such as output tokens per request, p95 latency, branch completion, or mechanism-specific throughput when they are central to the experiment. Use `N/A` for invalid deltas and explain why.

## Experiment and deployment fingerprint

Summarize the stable and varying parameters without duplicating every manifest field.

| Dimension | Stable value or per-configuration difference |
|---|---|
| Model and revision |  |
| Runtime image and version |  |
| GPU topology, TP, PP, replicas |  |
| GPU memory utilization |  |
| Maximum model length and sequences |  |
| CPU tier and secondary tier |  |
| Secondary-tier threads |  |
| Shared memory and mounts |  |
| Workload, concurrency, duration, seed |  |
| Cache cleaning and warmup |  |
| Node placement and topology |  |

## Comparison evidence

Start with cross-configuration outcomes before mechanism-specific diagnostics.

Recommended order:

1. Request throughput
2. Output-token throughput and output-token distribution
3. TTFT and end-to-end latency
4. Request disposition
5. Session and branch outcomes
6. Queueing and saturation
7. Prompt-token source or reuse mechanism
8. Configuration-specific resource metrics

For every figure, use this prose/figure/prose pattern:

> Figure `<N>` compares `<metric>` across configurations. Provenance: `<artifact or Prometheus metric>`, sampled at `<native cadence>` with `<aggregation, if any>`.

Replace the following example with validated inline data, then change the fence from `json` to `vega-lite`:

```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "background": "white",
  "width": 760,
  "height": 340,
  "title": "Figure <N> — <descriptive title>",
  "data": {
    "values": [
      {"configuration": "<Baseline>", "value": 0},
      {"configuration": "<Configuration 2>", "value": 0}
    ]
  },
  "mark": {"type": "bar"},
  "encoding": {
    "x": {
      "field": "configuration",
      "type": "nominal",
      "title": "Configuration",
      "sort": ["<Baseline>", "<Configuration 2>"]
    },
    "y": {
      "field": "value",
      "type": "quantitative",
      "title": "<Metric (unit)>",
      "scale": {"zero": true}
    },
    "color": {
      "field": "configuration",
      "type": "nominal",
      "title": "Configuration",
      "scale": {
        "domain": ["<Baseline>", "<Configuration 2>"],
        "range": ["#1f77b4", "#ff7f0e"]
      }
    }
  }
}
```

Interpretation: `<measured observation>`. `<Bounded inference>`. This figure does not establish `<limitation>`.

## Request-level results

Cover request throughput, output-token throughput, total output, output-token samples/distributions, completion disposition, TTFT, inter-token latency, end-to-end latency, and request errors. Prefer comparison figures with consistent order and colors.

## Session- and branch-level results

Cover completed and incomplete sessions, session throughput, session latency, active sessions at cutoff, branches/children, branch errors, and truncation. Explain fixed-duration cutoff behavior separately from functional failure.

## Saturation, errors, and validity evidence

Show the evidence behind the validity verdict: running/waiting requests, queue growth, saturation, failures, warnings, retries, preemptions, OOMs, restarts, and configuration drift. Include failed runs; do not bury them in an appendix.

## Mechanism and Prometheus evidence

Add only the resource metrics that characterize each mechanism, but inspect all available metrics before deciding. Explain why every included metric matters.

For time series:

- Use elapsed time when run start times differ.
- Preserve native samples.
- Align comparable panels to the same elapsed-time domain.
- Use the declared configuration colors.
- Mark important events such as warning onset, cache saturation, or queue growth when supported.

If the section is too large, summarize the conclusions here and link:

- [[<report base name>/01 - Request-level comparison]]
- [[<report base name>/02 - Session-level comparison]]
- [[<report base name>/03 - Mechanism and Prometheus comparison]]
- [[<report base name>/04 - Per-configuration diagnostics]]

## Conclusions

### Established by the data

- <Measured conclusion>
- <Measured conclusion>

### Uncertainty and limitations

- <Missing repetition, telemetry, configuration control, or other limitation>

### Decision

State the practical recommendation, or state that no recommendation is defensible.

## Next experiment

Specify the smallest experiment that resolves the most important uncertainty:

- Controls and stable parameters
- Configurations
- Repetitions and seeds
- Concurrency or pressure steps
- Cache cleaning and warmup
- Required telemetry
- Acceptance and rejection criteria

## Run registry and provenance

| Configuration | Child/deployment | MLflow run ID and link | Artifact sources | Disposition |
|---|---|---|---|---|
| <Baseline> |  | [<run ID>](<direct MLflow URL>) |  | Accepted |
| <Configuration 2> |  | [<run ID>](<direct MLflow URL>) |  | Accepted / Rejected |

State:

- Artifact download date
- Native metric cadence
- Any aggregation or derived formulas
- Missing artifacts or inaccessible metrics
- Exact source files used for tables and figures

## Final publication checklist

- The filename and H1 begin with the date.
- No existing report was overwritten without explicit authorization.
- A collision uses the next `(revision N)` suffix.
- Frontmatter contains only double-quoted strings and numbers.
- The benchmark overview is brief and identifies stable and varying parameters.
- The executive summary states issues, winners, losers, parity, and conclusions.
- The headline table contains direct MLflow links and configuration labels.
- Failed or invalid runs are visible and are not ranked.
- Request-, session-, branch-, latency-, error-, and saturation-level evidence is included when available.
- Mechanism-specific Prometheus metrics are included only where relevant.
- Every figure has a number, provenance, units, explanation, and interpretation.
- Configuration order, legend order, and colors are consistent.
- Native metric cadence is preserved; large content is split, not downsampled.
- Missing telemetry and limitations are explicit.
- New diagnostic plots were added when they materially improve the analysis.
- The report is prose-led and evidence-rich, not an unexplained sequence of plots.
- The experiment/model index is updated with the report and run registry.