---
title: "COSTAR-KV future-value placement experiments"
date: "2026-08-25"
type: "research-index"
experiment: "COSTAR-KV future-value-aware admission, retention, and eviction"
status: "active"
---

# COSTAR-KV future-value placement experiments

## Research question

How much of the finite-CPU placement opportunity can a low-overhead online policy recover by valuing arriving and resident KV according to future reuse, complete request/prefix contribution, and reload cost?

## Current conclusion

The equal-capacity placement opportunity is now validated in both pressure regimes.

- C32 recorded placement completes 30/42 external targets; matched next-use completes 42/42 and avoids 12/12 reads plus 36.440 seconds of device service.
- C64 recorded placement completes 577/789 external targets; matched next-use completes 716/789 and avoids 144/212 reads plus 540.369 seconds of device service.
- Forced-admit and bypass-capable next-use have the same movement result on both traces. Bypass reduces churn; future-aware victim ranking produces the measured request benefit.
- Exact-key history is unavailable for 100% of valuable C32 arrivals and 91.83% of valuable C64 arrivals. Generic LRU/ARC/frequency/reuse-distance policies therefore learn too late.
- Demand is request/prefix structured: the dominant originating request supplies 83.69% of C32 and 86.03% of C64 target keys.
- No practical policy tested so far beats recorded placement. Whole-bundle LRU gives a tiny C64 improvement over blockwise bundle LRU but remains 38 net misses worse than recorded.
- Prompt-size value bands reproduce descriptively across pressure regimes, but the frozen C32 hard-priority policy fails badly on C64. Prompt size can only be a soft feature, not a binary admission rule.

The next research step is a held-out **request/prefix expected-value ranking** experiment, not another block predictor or hard threshold.

## Experiment registry

| Experiment | Date | Status | Decision |
|---|---|---|---|
| [[01 - Experiment 1 matched next-use admission decomposition|01 — Matched next-use admission decomposition]] | 2026-08-25 | Conditionally valid diagnostic | Future-aware victim ranking produces the C32 movement result; bypass reduces churn |
| [[02 - Experiment 2 practical forced-admit policy benchmark|02 — Practical forced-admit policy benchmark]] | 2026-08-25 | Conditionally valid negative result | Do not implement the seven exact-key history policies live |
| [[03 - Experiment 3 first-admission information audit|03 — First-admission information audit]] | 2026-08-25 | Valid offline diagnostic | Exact-key history is absent at valuable C32 admissions; test request context |
| [[04 - Experiment 4 contextual request-bundle placement|04 — Contextual request-bundle placement]] | 2026-08-25 | Conditionally valid negative result | Reject tested C32 context rules; block hits and gross avoided reads conceal substitution |
| [[05 - Experiment 5 whole-bundle eviction diagnostic|05 — Whole-bundle eviction diagnostic]] | 2026-08-25 | Valid negative result | Whole-source-request FIFO/LRU granularity alone is insufficient |
| [[06 - Experiment 6 C64 independent pressure validation|06 — C64 independent pressure validation]] | 2026-08-25 | Valid trace; conditional policy replay | Opportunity and structure replicate; frozen practical rules fail held-out |

## Accepted corpora

### C32

- MLflow run: [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- Database: `/private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite`
- Database SHA-256: `788adedcbc9850a64443681fbd8acf36ea62cdce950eeac6311f730f34cce687`
- Events / requests: 2,241,218 / 901
- Native movement validation: 0/898 mismatches

### C64

- MLflow run: [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow)
- Trace: `/private/tmp/abc-oracle-validation-20260825/c64/oracle-trace-c64.complete.jsonl`
- Trace bytes: 6,959,277,072
- Trace SHA-256: `1167b512741bb97d2b76744cb238ede58fa0c6c2ef35ad7b3e9892c05b4ece3d`
- Database: `/private/tmp/abc-oracle-validation-20260825/c64-normalized.sqlite`
- Events / requests: 13,629,779 / 1,280
- Native movement validation: 0/1,263 mismatches

Both corpora use 131,072 two-MiB CPU KV chunks.

## Methodological correction

Initial offline tools selected the last resolved connector lookup. C64 exposed 34 requests with later decode-era resolved cycles and different working-set versions. Initial TTFT/placement targets now use the first resolved lookup. Native-read requests are evaluated at earliest native-load submission; no-read requests are evaluated at first resolution. C32 headlines are unchanged; C64 now reproduces native movement exactly.

## Next experiment

Construct a bundle-level value-ranking dataset across C32 and C64.

- Predict future complete-target contribution, time to reuse, reuse count, and reload cost.
- Use prompt size, source reuse, bundle size, prefix structure, age, and pressure as soft features.
- Train/calibrate on one execution and validate ranking and placement on the other.
- Blend expected value per byte with recency and a complete-target constraint.
- Require fewer net incomplete requests than recorded placement and LRU before live implementation.

If available features do not show held-out lift, add stable session/conversation/user identity plus agent tool/DAG events before continuing.

## Related evidence

- [[../Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle|Finite CPU retention oracle]]
- [[../Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration|COSTAR Experiment 0 corpus calibration]]
- [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|Independent research audit and redirection]]