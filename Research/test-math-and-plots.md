# Temporary Markdown Rendering Test

> Throwaway article for testing Arche's LaTeX and Vega-Lite rendering. All measurements below are synthetic and must not be used as benchmark evidence.

## Equations

The illustrative CPU-copy time is $T_{CPU} \approx 75\ \text{s}$, where $T$ denotes elapsed time in seconds.

$$
T_{CPU} = \frac{32\ \text{GiB}}{455\ \text{MB/s}} \approx 75\ \text{s}
$$

For a synthetic benchmark, successful request throughput and request error rate can be written as:

$$
\begin{aligned}
\operatorname{Throughput} &= \frac{N_{\mathrm{successful}}}{\Delta t}, \\
\operatorname{ErrorRate} &= \frac{N_{\mathrm{failed}}}{N_{\mathrm{successful}} + N_{\mathrm{failed}}} \times 100\%.
\end{aligned}
$$

In prose, an illustrative error rate of $e = 5.9\%$ means roughly six failed requests per one hundred attempted requests.

## Charts

Figure 1 compares synthetic inference throughput across several illustrative model sizes.

*Provenance: Synthetic illustrative values created solely to test Markdown chart rendering; they are not measured results.*

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": {
    "text": "Figure 1 — Illustrative inference throughput by model size",
    "anchor": "start",
    "fontSize": 16
  },
  "width": "container",
  "height": 300,
  "data": {
    "values": [
      { "model_size_b": 7, "throughput_tokens_s": 8120 },
      { "model_size_b": 14, "throughput_tokens_s": 6240 },
      { "model_size_b": 32, "throughput_tokens_s": 4010 },
      { "model_size_b": 70, "throughput_tokens_s": 2180 }
    ]
  },
  "mark": {
    "type": "bar",
    "color": "#4C78A8",
    "cornerRadiusEnd": 3
  },
  "encoding": {
    "x": {
      "field": "model_size_b",
      "type": "ordinal",
      "title": "Model size (billions of parameters)",
      "sort": "ascending"
    },
    "y": {
      "field": "throughput_tokens_s",
      "type": "quantitative",
      "title": "Inference throughput (tokens/s)",
      "scale": { "zero": true }
    },
    "tooltip": [
      { "field": "model_size_b", "type": "ordinal", "title": "Model size (B parameters)" },
      { "field": "throughput_tokens_s", "type": "quantitative", "title": "Throughput (tokens/s)", "format": ",.0f" }
    ]
  },
  "config": {
    "axis": { "labelFontSize": 12, "titleFontSize": 13 },
    "view": { "stroke": null }
  }
}
```

As Figure 1 shows, the synthetic dataset deliberately assigns lower throughput to larger models.

Figure 2 shows an illustrative latency curve as request concurrency rises.

*Provenance: Synthetic illustrative values created solely to test Markdown chart rendering; they are not measured results.*

```vega-lite
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "title": {
    "text": "Figure 2 — Illustrative p95 latency versus concurrency",
    "anchor": "start",
    "fontSize": 16
  },
  "width": "container",
  "height": 300,
  "data": {
    "values": [
      { "concurrency_requests": 1, "p95_latency_ms": 420 },
      { "concurrency_requests": 8, "p95_latency_ms": 510 },
      { "concurrency_requests": 16, "p95_latency_ms": 690 },
      { "concurrency_requests": 32, "p95_latency_ms": 1040 },
      { "concurrency_requests": 64, "p95_latency_ms": 1880 },
      { "concurrency_requests": 128, "p95_latency_ms": 3620 }
    ]
  },
  "mark": {
    "type": "line",
    "color": "#E45756",
    "strokeWidth": 3,
    "point": { "filled": true, "size": 80 }
  },
  "encoding": {
    "x": {
      "field": "concurrency_requests",
      "type": "quantitative",
      "title": "Request concurrency (requests)",
      "scale": { "zero": true }
    },
    "y": {
      "field": "p95_latency_ms",
      "type": "quantitative",
      "title": "p95 end-to-end latency (ms)",
      "scale": { "zero": true }
    },
    "tooltip": [
      { "field": "concurrency_requests", "type": "quantitative", "title": "Concurrency (requests)" },
      { "field": "p95_latency_ms", "type": "quantitative", "title": "p95 latency (ms)", "format": ",.0f" }
    ]
  },
  "config": {
    "axis": { "grid": true, "labelFontSize": 12, "titleFontSize": 13 },
    "view": { "stroke": null }
  }
}
```

Figure 2 is intentionally shaped like a saturation curve: latency rises slowly at low concurrency and more sharply beyond 32 concurrent requests.