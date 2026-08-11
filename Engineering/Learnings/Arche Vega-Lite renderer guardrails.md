# Arche Vega-Lite renderer guardrails

## Symptom

A syntactically valid `vega-lite` fenced JSON block is displayed as source with:

> Unable to render chart. The spec is still available above.

This can happen before Vega-Lite runs because Arche applies its own safety and feature validator through `parseChartSpec`.

## Gotcha 1: comparison expressions containing `<` or `>`

Arche recursively checks every string in a spec against `/[<>]/`. Consequently, a valid Vega expression such as:

```json
{"condition":{"test":"datum.change >= 0","value":"#d62728"},"value":"#1f77b4"}
```

is rejected because the `test` string contains `>`.

Prefer either a field-predicate object, which contains no comparison-character string:

```json
{"condition":{"test":{"field":"change","gte":0},"value":"#d62728"},"value":"#1f77b4"}
```

or, more robustly, precompute a categorical field in `data.values` and encode color from that field.

## Gotcha 2: concatenation specs are not supported

Arche currently whitelists `layer` but not `vconcat`, `hconcat`, `concat`, `facet`, or `repeat` as top-level keys. It also requires a top-level `mark` or non-empty `layer`. A valid Vega-Lite dashboard built with `vconcat` therefore fails the guard.

Use one of these approaches:

- split each panel into its own numbered fenced figure;
- use supported `layer` composition, optionally with independent scales through `resolve`;
- use a single long-form dataset and one mark when the measures share a meaningful axis.

## Gotcha 3: `mermaid` fenced blocks do not render inline

The Arche Markdown renderer supports fenced `vega-lite` blocks and KaTeX math, but **not** fenced `mermaid` blocks. A fenced ` ```mermaid ` block inside a KB article or report renders as plain source code — the diagram does not appear. Mermaid is only available through the `diagram_create` tool, which produces a standalone diagram image outside the Markdown document.

For in-document flow/sequence content, either:

- express the progression as a fenced `vega-lite` chart (e.g., an ordered categorical/line chart for phase progressions); or
- render a true flowchart via `diagram_create` and reference the resulting image as an attachment.

## Validation checklist

Before publishing an Arche chart:

1. Parse the JSON.
2. Confirm the exact schema is `https://vega.github.io/schema/vega-lite/v5.json`.
3. Keep inline `data.values` at 1–1000 rows and at most 50 distinct columns.
4. Use a supported top-level `mark` or `layer`.
5. Avoid `<` and `>` in all strings, including expressions, titles, labels, and data.
6. Avoid URL-like strings and keys named `url`, `href`, or `src`.
7. Do not use concatenation/facet/repeat composition.
8. Test against Arche's `parseChartSpec`, not only generic Vega-Lite or JSON validation.
9. Do not use fenced `mermaid` blocks for in-document diagrams; use `vega-lite` or `diagram_create` instead.

## Source

Runtime validator: `apps/web/src/components/workspace/chat-panel/chart-output.ts`.

The workspace AGENTS guide lists the general fenced-chart constraints, but the runtime validator is authoritative and may support a slightly different key set.

Observed failure: ABC experiment definition (2026-08-11) — fenced `mermaid` block displayed as source; replaced with a `vega-lite` figure.
