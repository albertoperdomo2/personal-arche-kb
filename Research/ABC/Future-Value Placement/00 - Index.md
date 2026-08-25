---
title: "COSTAR-KV future-value placement experiments"
date: "2026-08-25"
type: "research-index"
experiment: "COSTAR-KV future-value-aware admission, retention, and eviction"
status: "active"
---

# COSTAR-KV future-value placement experiments

## Research question

How much of the finite-CPU placement opportunity can a low-overhead online policy recover by valuing arriving and resident KV according to future reuse, request/prefix contribution, and reload cost?

This series follows the corrected offline retention result. It treats proactive movement as a later action and begins by separating victim-selection value from the ability to reject mirrored arrivals.

## Current conclusion

[[01 - Experiment 1 matched next-use admission decomposition|Experiment 1]] finds that, under the matched common-arrival C32 replay, **future-aware victim selection alone is sufficient to recover the complete measured movement result**. Forced-admit and bypass-capable next-use policies both make all 42 nonempty external targets CPU-complete and avoid all 12 native reads / 36.440 seconds of measured device service.

Bypass adds no movement or request-completeness benefit over forced admission in this experiment. It does reduce evictions by 75.0%, admissions by 48.3%, and total admission-plus-eviction churn by 58.8%. The earlier claim that rejection was required for the 12-read result is therefore corrected: rejection explains churn reduction, while victim selection explains the measured movement result under this replay.

## Experiment registry

| Experiment | Date | Status | Decision |
|---|---|---|---|
| [[01 - Experiment 1 matched next-use admission decomposition|01 — Matched next-use admission decomposition]] | 2026-08-25 | Conditionally valid diagnostic | Prioritize practical future-value victim policies; retain bypass as a churn optimization |

## Source corpus

- MLflow run: [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- Normalized database: `/private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite`
- SHA-256: `788adedcbc9850a64443681fbd8acf36ea62cdce950eeac6311f730f34cce687`
- Requests: 901 total; 42 with nonempty external targets
- CPU capacity: 131,072 KV chunks
- Native secondary reads: 12 jobs / 36.440469092 seconds measured service

## Next experiment

Benchmark practical forced-admit victim policies at exact capacity before adding richer admission machinery. Begin with LRU, ARC, aged LFU/LRFU, 2Q/LIRS-style reuse-distance policies, and the ATC'25 workload-aware KV baseline. Evaluate request completeness, native-read/service avoidance, churn, metadata, and runtime cost. Add explicit admission filters as a second axis because Experiment 1 shows their unique measured contribution is churn rather than read avoidance on C32.

## Related evidence

- [[../Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle|Finite CPU retention oracle]]
- [[../Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration|COSTAR Experiment 0 corpus calibration]]
- [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|Independent research audit and redirection]]