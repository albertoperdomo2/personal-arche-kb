---
title: "llm-d-sc Metrics Accuracy Benchmark — Test Characterization"
date: 2026-08-27
type: research
experiment: llm-d-sc-pipeline-review
repo: https://github.com/llm-d-incubation/llm-d-semantic-classifier
head_commit: f6008c9
test_file: tests/metrics_accuracy.rs
model: cnuland/llm-d-sc-complexity (c5f55ef419d2)
---

# Metrics Accuracy Benchmark — Test Characterization

This document describes what `tests/metrics_accuracy.rs` measures, how to interpret the results, and what constitutes evidence of the bug and its fix.

## Background: the three request paths

The classification pipeline has three distinct paths a request can take, each with dramatically different latency:

| Path | What happens | Latency (measured) | Current metrics label |
|------|-------------|-------------------|----------------------|
| **True hit** | Result already in cache. No tokenization, no BERT forward, no ranking. Returns the cached `ClassificationResult`. | ~7.7 µs | `cache_hit` ✓ |
| **True miss** | Cache miss. Tokenize → BERT forward → anchor rank → store in cache → return. | ~226 ms | `cache_miss` ✓ |
| **Coalesced wait** | Cache miss, but another thread is already running the forward for the same key. Waits on a `Condvar` for the other thread's result. Experiences full forward latency. | ~398 ms | `cache_hit` ✗ |

The bug: coalesced waits are labeled `cache_hit` even though they experience forward-class latency (~50,000× slower than a true hit).

### Why this happens

In `src/classify.rs:283-305`, each call to `ServiceCore::classify()` creates a per-request `AtomicBool` called `forward_ran`. This flag is set to `true` inside the forward closure. After classification completes:

```
if forward_ran == true  → record as cache miss
if forward_ran == false → record as cache hit
```

In the single-flight path (`src/cache.rs:295-315`), only the **designated forwarder** thread runs the closure. All waiting threads receive the result via `Condvar` — their closures never execute, so `forward_ran` stays `false`, and they are counted as hits.

## Test M-001: Single-flight metrics proof

### What it does

1. Creates a `ServiceCore` wrapping a real `CandleClassifier` loaded from `artifacts/models/complexity`
2. Runs one warmup classification on an unrelated prompt (primes the model)
3. Launches **32 threads** simultaneously (synchronized by a `Barrier`) that all classify the **same prompt** on a **cold cache** (no prior result for this key)
4. Waits for all threads to complete
5. Reads the metrics snapshot and reports hits vs misses

### What to expect

Single-flight coalescing works correctly: only **1 forward** runs (the designated forwarder), the other 31 threads wait for its result.

But the metrics report:
- **31 hits, 1 miss** → 96.9% hit rate
- True breakdown: **0 hits, 1 miss, 31 coalesced waits** → 0% hit rate

The test also prints each thread's latency. All 32 threads show forward-class latency (~192 ms), confirming none of them experienced a true cache hit (~µs).

### How to read the output

```
Concurrent requests:   32       ← all 32 sent the same text
Forwards executed:     1        ← coalescing works (good)
Reported cache hits:   31       ← BUG: these are coalesced waits, not hits
Reported cache misses: 1        ← only the forwarder is counted as a miss
Reported hit rate:     96.9%    ← wildly inflated

Burst latencies:
  min:    191.9ms               ← all threads took ~192ms
  median: 192.0ms               ← a true hit would be ~7µs
  max:    192.1ms               ← no thread experienced hit-class latency
```

### Success criteria

**Before fix:** `cache_hits == 31` and `cache_misses == 1` (documents the bug).

**After fix:** `cache_hits == 0`, `cache_misses == 1`, `cache_coalesced == 31`.

## Test M-002: Three-path latency comparison

### What it does

Measures all three request paths on the same `ServiceCore` with a real model, then prints a side-by-side comparison table.

**Step 1 — True miss:** Classify a prompt that has never been seen. Measures full pipeline latency (tokenize + BERT forward + rank).

**Step 2 — True hit:** Classify the **same prompt** 200 times. All served from cache. Reports the mean per-request latency.

**Step 3 — Coalesced wait:** Launch **16 threads** simultaneously on a **different, new prompt** (cold key). One thread runs the forward; the other 15 wait. Reports median/min/max latency of the burst.

### What to expect

The table shows three rows with their latency, what the metrics call them, and what they actually are:

```
  Path               | Latency      | Metrics label   | Correct label
  -------------------+--------------+-----------------+----------------
  TRUE MISS          |   225.7ms    | cache miss      | cache miss
  TRUE HIT (mean)    |     7.7µs   | cache hit       | cache hit
  COALESCED (median) |   398.4ms   | cache hit  ← !! | coalesced wait
```

Two ratios are computed:

- **miss / hit:** How much slower a miss is than a hit. Expected: thousands to tens of thousands ×. Confirms the pipeline is working (cache hits are fast).
- **coalesced / hit:** How much slower a coalesced wait is than a true hit. If coalesced waits were truly "hits," this ratio should be ~1×. A measured ratio of 50,000× proves they are not hits.

### How to read the output

The key number is `coalesced / hit ratio`. Anything above ~5× proves the coalesced path has miss-class latency and should not be labeled a cache hit.

Measured: **51,890×** — a coalesced wait takes 398ms while a true hit takes 7.7µs. Both are called "cache hit" in the metrics.

### Success criteria

**Before fix:** The table shows a massive coalesced/hit ratio. Reported hit rate is >95%.

**After fix:** Same table, but the "Metrics label" column should show `coalesced` for the third row, and the reported metrics should break out hits/misses/coalesced separately.

## Test M-003: Mixed workload inflation

### What it does

Runs a realistic mixed workload in three phases, then compares the reported hit rate to the true hit rate.

**Phase 1 — 10 distinct prompts:** 10 unique texts, all cold misses. Establishes 10 cached entries.

**Phase 2 — 100 true hits:** Repeats one of the cached prompts 100 times. All served from cache. These are genuine cache hits.

**Phase 3 — 32-thread burst on a new key:** 32 threads classify a brand-new prompt simultaneously. 1 miss + 31 coalesced waits.

**Total: 142 requests** with a known ground truth.

### What to expect

| Metric | Reported | True |
|--------|----------|------|
| Hits | 131 | 100 |
| Misses | 11 | 11 |
| Coalesced | n/a | 31 |
| Hit rate | 92.3% | 70.4% |
| Inflation | +21.8pp | — |

The 31 coalesced waits from phase 3 inflate the reported hit count from 100 to 131, pushing the reported hit rate up by **21.8 percentage points**.

### How to read the output

The table shows reported vs true side by side. The `Inflation` row shows how many percentage points the reported hit rate exceeds the true hit rate.

In production with higher concurrency (hundreds or thousands of concurrent requests per key), this inflation would be even larger. The more concurrent traffic you have on the same prompts, the worse the inflation — which is exactly the scenario (gateway-level deployment) where accurate metrics matter most.

### Success criteria

**Before fix:** `cache_hits > 100` (inflated by coalesced waits). Inflation > 0.

**After fix:** `cache_hits == 100`, `cache_coalesced == 31`, `cache_misses == 11`. Inflation == 0.

## How to run

```bash
# Fetch the complexity model (~80MB, one-time download)
./hack/fetch-model --classifier complexity

# Run all three tests with output
cargo test --test metrics_accuracy -- --ignored --nocapture

# Run one test at a time
cargo test --test metrics_accuracy m001 -- --ignored --nocapture
cargo test --test metrics_accuracy m002 -- --ignored --nocapture
cargo test --test metrics_accuracy m003 -- --ignored --nocapture
```

Or run the binary directly (useful for piping output):

```bash
./target/debug/deps/metrics_accuracy-* --ignored --nocapture --test-threads=1
```

Tests take ~15-20 seconds each (model loading + BERT forwards). Total: ~40-60 seconds.

## Relationship to Praxis benchmarks

These tests are the in-process equivalent of Praxis filter benchmark B-1, which measured:

| Arm (Praxis B-1) | p50 | Equivalent here |
|-------------------|-----|-----------------|
| baseline (no classification) | 0.145 ms | n/a (no proxy layer) |
| classified-hit | 0.215 ms (+0.070ms) | True hit (~7.7 µs in-process) |
| classified-miss | 14.099 ms (+13.954ms) | True miss (~226 ms in-process) |

The in-process numbers are higher because they include the raw BERT forward without the Praxis proxy's own overhead dominating. The **ratio** between hit and miss is comparable: Praxis B-1 showed ~200× (through the proxy), our M-002 shows ~29,000× (in-process, no proxy overhead floor).

The coalesced path was never measured in Praxis B-1 — it would require a concurrent-burst arm with metrics inspection. That is what M-001/M-002/M-003 add.

## Not yet recorded as an issue

As of 2026-08-27, this metrics inaccuracy is not tracked in the project's GitHub issues. The 18 open issues include related topics (#9 FIFO eviction, #11 Prometheus export, #16 batched forwards) but none address the hit/miss counter accuracy under single-flight coalescing. The spec evidence for AC-007 (single-flight) proves coalescing works but does not examine the metrics labels. The spec evidence for AC-012 (metrics) proves `hits + misses == total` but not that requests are in the correct bucket.