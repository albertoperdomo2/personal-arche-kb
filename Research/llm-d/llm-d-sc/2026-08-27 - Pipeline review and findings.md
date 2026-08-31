---
title: "llm-d-sc Pipeline Review and Findings"
date: 2026-08-27
type: research
experiment: llm-d-sc-pipeline-review
repo: https://github.com/llm-d-incubation/llm-d-semantic-classifier
head_commit: f6008c9
head_message: "Merge pull request #19 from llm-d-incubation/chore/codeowners-classifiers"
praxis_filter_repo: https://github.com/cnuland/llm-d-sc-praxis-filter
---

# llm-d-sc Pipeline Review and Findings

Code-level review of the llm-d-semantic-classifier pipeline at commit `f6008c9`, cross-referenced with the Praxis filter benchmarks and findings from `llm-d-sc-praxis-filter`.

## 1. Architecture overview

llm-d-sc is a lightweight Rust/Candle semantic classifier runtime. It takes a text prompt, embeds it with a resident sentence-transformer (BERT via Candle), scores it against labelled anchor sets using top-k mean cosine similarity, and returns ranked semantic signals over gRPC. It deliberately does **not** route — it produces classification evidence only (ADR-0001/AC-010). Routes are structurally unrepresentable on the wire: the proto schema carries labels, scores, and revision provenance, with no route field.

### Pipeline stages

```
gRPC request
  → Bounded handoff (off Tokio workers, queue cap 256)
    → Text normalization (trim)
      → Versioned cache lookup (blake3 fingerprint)
        → [HIT] Return cached ClassificationResult
        → [MISS] Single-flight coalescing (Condvar-based)
          → Tokenize (HuggingFace tokenizers, pure Rust)
            → BERT forward (Candle, CPU, memory-mapped safetensors)
              → Mean-pool + L2-normalize → embedding vector
                → Anchor ranking (cosine similarity, top-k mean)
                  → ClassificationResult (ranked labels + scores + revision fingerprint)
```

### Key source files

| Component | File | Purpose |
|-----------|------|---------|
| Cache | `src/cache.rs` | Exact-result cache (blake3 key, FIFO eviction, 50k cap) + SharedCache with single-flight |
| Pipeline core | `src/classify.rs` | `ServiceCore<R>` wrapping any `ClassifierRuntime`; `CandleClassifier` backend |
| Ranking | `src/ranker.rs` | Pure-math cosine similarity, `anchor_rank` (top-k mean), `cosine_rank` |
| Embedding | `src/embedding.rs` | BERT model load (memory-mapped safetensors), tokenize, forward, mean-pool, L2-normalize |
| gRPC surface | `src/grpc/classify.rs` | `ClassifyServiceImpl<R>`, telemetry hashing, signal validation |
| Handoff | `src/handoff.rs` | `InferenceExecutor`, bounded channel, dedicated thread pool |
| Taxonomy | `src/taxonomy.rs` | `ClassifierDefinition` deserialization, anchor embedding at load time |

## 2. Finding: single-flight waiters counted as cache hits

**Severity:** Metrics inaccuracy — misleading in production.
**Location:** `src/classify.rs:283-305`

### Mechanism

`ServiceCore::classify()` uses a per-request `AtomicBool` flag (`forward_ran`) to distinguish cache hits from misses. The flag is set to `true` inside the forward closure. After `SharedCache::classify_concurrent()` returns, the flag is checked:

```rust
// src/classify.rs:301-305
if forward_ran.load(Ordering::SeqCst) {
    metrics.record_cache_miss();
} else {
    metrics.record_cache_hit();
}
```

In the single-flight path (`src/cache.rs:295-315`), only the **designated forwarder's** closure runs. All other concurrent callers for the same key wait on the `Condvar` and receive the shared result — their closures are never invoked, so `forward_ran` stays `false`. They are counted as cache **hits**.

### Impact

If 100 identical requests arrive concurrently on a cold cache:
- The metrics report: **1 miss, 99 hits** (99% hit rate)
- The reality: **100 misses coalesced into 1 forward** (0% hit rate, excellent coalescing)

The hit rate metric is inflated. An operator seeing 99% hit rate would conclude caching is effective, when in reality the load pattern is heavy concurrent duplication on the same keys. This distinction matters because:

- True cache hits cost ~0.070 ms (`src/cache.rs:108` documents 632ns internally; B-1 from Praxis benchmarks confirms +0.070ms at p50 end-to-end)
- Coalesced waits cost the full forward duration: 14–53 ms depending on token count (B-3 from Praxis benchmarks)

The metrics conflate two paths with a **200× latency difference**.

### Compounding issue: failed forwards counted as hits

If the designated forwarder's forward **fails**, the error is propagated to all waiters (`src/cache.rs:317-331`). Their `forward_ran` is still `false`, so failed requests are recorded as cache hits. The metrics say "hit" for requests that returned an error.

### Suggested fix

Add a third counter (`record_cache_coalesce()`) or at minimum stop counting coalesced waiters as hits. The `SharedCache` could return an enum `{ Hit, Miss, Coalesced }` instead of relying on the forward-ran flag.

## 3. Finding: double text normalization

**Severity:** Minor inefficiency.
**Location:** `src/classify.rs:265` and `src/classify.rs:583`

`ServiceCore::classify()` normalizes the input text (trim + allocate new String) and passes the normalized version into the forward closure. `CandleClassifier::classify()` then trims and allocates again:

```rust
// ServiceCore::classify (line 265)
let normalized = input.text.trim().to_string();
// ... builds ClassificationInput { text: normalized, ... }

// CandleClassifier::classify (line 583) — called by the forward closure
let normalized = input.text.trim().to_string();  // already trimmed
self.real_forward(&normalized)
```

Trimming an already-trimmed string is idempotent but allocates a redundant `String` on every cache miss. On a path where the codebase documents 632ns cache hits and uses bounded executors to avoid Tokio thread contention, this is inconsistent with the optimization posture.

## 4. Finding: FIFO eviction is a deliberate, data-driven choice

**Not a flaw.** The comment at `src/cache.rs:107-113` explains:

> Deliberately FIFO rather than LRU. FIFO bounds memory, which is the actual defect, and costs one push and one pop per insert. LRU would retain hot keys better but needs recency bookkeeping on every HIT, and a hit is currently 632 nanoseconds, so that bookkeeping is a real fraction of it.

This was measured. LRU's per-hit overhead would be a significant fraction of the 632ns hit latency. FIFO was chosen because memory bounding (the actual risk) is the priority, not hot-key retention.

## 5. Assessed and rejected: caching embeddings for semantic cache lookup

### Why it was considered

The classifier computes an embedding (BERT forward) and then discards it after anchor ranking. Could caching the embedding enable a semantic cache — matching semantically similar (but textually different) prompts to avoid redundant classifications?

### Why it doesn't work

The BERT forward is the expensive step (~14–53 ms depending on token count, per B-3). The anchor ranking step after it is trivially cheap (~40–50 cosine similarities). A semantic cache lookup **also requires the embedding** (you must embed the new prompt to compare it against cached embeddings), so the expensive BERT forward runs either way. The only savings would be skipping the ranking — but the ranking is ~40 cosine similarities, while a semantic cache with more than ~40 entries would cost **more** than just doing the ranking.

The math: you'd add complexity (ANN index, similarity threshold, cache management) to save microseconds on the cheap step while the expensive step is unchanged.

### Where embedding exposure does make sense

Exposing the embedding in the gRPC response as an optional field — not for the classifier's own caching, but for a downstream consumer (e.g., the router's semantic cache of **LLM responses**). The savings there are enormous: skipping a full LLM inference (seconds, expensive GPU), not 40 cosine similarities. The classifier already computes the embedding; one BERT forward could serve both classification and the router's cache lookup. This is a different feature with a different justification.

## 6. Cross-reference: Praxis filter benchmarks

Key performance data from `llm-d-sc-praxis-filter/bench/BENCHMARKS.md` (measured 2026-08-24/25):

### B-1: Filter overhead

| Path | p50 latency | Throughput (c=1) |
|------|-------------|-----------------|
| Baseline (no classification) | 0.145 ms | 6149.8 req/s |
| Cache hit | 0.215 ms (+0.070 ms) | 4301.4 req/s |
| Cache miss | 14.099 ms (+13.954 ms) | 70.4 req/s |

### B-3: Prompt-length sensitivity (cache miss, c=1)

| Tokens | p50 | Throughput |
|--------|-----|-----------|
| 32 | 14.727 ms | 67.0 req/s |
| 64 | 18.658 ms | 53.1 req/s |
| 128 | 38.612 ms | 25.8 req/s |
| 256 | 52.968 ms | 18.7 req/s |
| 512 | 53.092 ms | 18.6 req/s |

Note: latency plateaus at ~256 tokens, indicating the BERT model's max sequence length is reached and longer inputs are truncated by the tokenizer.

### B-6: Failure modes

| Mode | p50 | Behavior |
|------|-----|----------|
| Classifier down (connection refused) | 0.186 ms | Fail-open, fast reject |
| Classifier slow (TCP blackhole) | 104.039 ms | Fail-open, but **560× latency increase** — "fail-open in name only" |
| Queue exhausted (c=300) | 430.194 ms | Graceful degradation |
| Fail-closed (configured) | 0.107 ms | 503, fastest path |

### B-7: In-cluster topology (c=1)

| Path | p50 |
|------|-----|
| Cache hit | 0.115 ms (+0.060 ms over baseline) |
| Cache miss | 21.035 ms (+20.976 ms over baseline) |

## 7. Cross-reference: classification accuracy (Praxis FINDINGS.md)

### F-7: Anchor generalization gap

The complexity classifier's accuracy drops from **97.5%** on its own held-out set to **68.8%** on independently-authored prompts. The cause is anchor bias: anchors skew toward "technical system-design" framing rather than broad complexity. A genuinely hard planning document lands semantically nearer the MEDIUM anchors than the COMPLEX ones.

This is fixable without retraining — the anchor system was designed for exactly this (swap the JSON, no model change).

### B-4: Routing accuracy on held-out prompts

| Metric | Value |
|--------|-------|
| Overall routing accuracy | 77.3% |
| Boundary-case accuracy | 37.5% |
| COMPLEX → small model (quality risk) | 16/32 (50%) |
| REASONING → small model (quality risk) | 12/32 (37.5%) |

Misroute errors are asymmetric: under-routing (quality risk) is more common and more costly than over-routing (wasted capacity). The Praxis findings conclude: "the label→cluster mapping is the safety knob, not the classifier."

### Model-affinity analysis

| Metric | Value |
|--------|-------|
| Observed model-selection agreement | 78.6% |
| Under-routing / quality risk | 11.9% |
| Over-routing / wasted capacity | 5.9% |
| Neither model sufficient | 16.9% |

**Caveat:** Large-model quality judgment has a 65.2% position-bias flip rate (unstable evaluator signal). Under-routing figures depend on this unstable signal.

## 8. Architectural observation: the evidence/routing boundary

The meeting notes (2026-08-25 sig-semantic-classifier call) surfaced a key design tension: the boundary between "semantic evidence" and "routing decision."

Today, this boundary is enforced **structurally** — the proto schema has no route field. Routes are literally unrepresentable on the wire. The classifier cannot tell you which model to use even if it wanted to.

The risk is gradual erosion through reasonable-sounding features: abstention thresholds (whose policy?), composite scores (whose composition logic?), model awareness ("it might know models at some point" — per project maintainer). Each step is individually defensible; the accumulation turns the classifier into an opinionated router without anyone intending it.

The design heuristic: when someone proposes a new response field, ask whether it describes **what the prompt is** (belongs in the classifier) or **what should happen to it** (belongs in the consumer).

## 9. Summary: where the real improvement levers are

| Area | Status | Impact |
|------|--------|--------|
| Pipeline mechanics (cache, single-flight, ranking) | Sound | Low — already well-optimized |
| Metrics accuracy (single-flight counted as hits) | Confirmed flaw | Medium — misleading in production |
| Double normalization | Confirmed inefficiency | Low — one redundant String allocation per forward |
| Anchor quality / generalization (F-7) | Primary gap | **High** — accuracy drops 97.5% → 68.8% on real prompts |
| Boundary-case classification | Related to anchors | **High** — 37.5% accuracy at decision boundaries |
| Embedding exposure for downstream reuse | Not implemented | Medium — one BERT forward could serve classification + semantic cache |
| Abstention logic | Stubbed, not implemented | Medium — design question about policy ownership |

The pipeline is not the bottleneck. The anchors are.