---
title: "llm-d-sc — Semantic Classifier Code Review & Pipeline Analysis"
date: 2026-08-27
type: research
experiment: llm-d-sc-pipeline-review
---

# llm-d-sc — Index

Durable record of analysis, findings, and research on the [llm-d-semantic-classifier](https://github.com/llm-d-incubation/llm-d-semantic-classifier) project.

## Research question

Where are the real improvement opportunities in llm-d-sc — in the classification pipeline mechanics, or in the anchor/taxonomy quality?

## Current status

**Initial code review and benchmark analysis complete.** The pipeline mechanics are well-engineered; the primary gap is classification accuracy on independently-authored prompts (anchor quality), not pipeline performance.

## Key documents

- [[2026-08-27 - Pipeline review and findings]] — code-level pipeline review against HEAD `f6008c9`, cross-referenced with Praxis filter benchmarks from `llm-d-sc-praxis-filter/bench/BENCHMARKS.md` and `FINDINGS.md`.

## Working conclusion

1. The classification pipeline (cache → single-flight → tokenize → BERT forward → anchor rank) is structurally sound and deliberately optimized.
2. The single confirmed pipeline flaw is a **metrics inaccuracy**: single-flight coalesced waiters are counted as cache hits, inflating the hit rate and masking concurrent-load patterns.
3. The real improvement lever is **anchor quality** — classification accuracy drops from 97.5% to 68.8% on independently-authored prompts (F-7 from Praxis findings), and boundary-case accuracy is 37.5%.
4. The architectural boundary ("we produce evidence, we don't route") is enforced structurally at the proto schema level, not just by convention.

## Open threads

- Whether the metrics fix (single-flight waiters counted as hits) warrants a contribution PR.
- Anchor quality improvement for the complexity classifier — framing/generalization bias toward technical system-design prompts.
- Potential for exposing the embedding vector in the gRPC response for downstream reuse (semantic caching of LLM responses, not classifier responses).
- Abstention logic — `ABSTAIN` status exists on the wire but nothing triggers it; threshold semantics are a design question (whose policy?).