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
- Prompt-size value bands reproduce descriptively across pressure regimes, but the frozen C32 hard-priority policy fails badly on C64.
- Experiment 7's soft static ranker fails the held-out capacity-weighted gate: C32→C64 request-context AUC is 0.674, yet byte-weighted AUC is 0.472 and top-25%-byte future-reference lift is only 0.788×.
- Experiment 8's age-, prompt-, request-, and coarse-prefix-conditioned hazards also fail. C32→C64 request+prefix hazard recovers 54.30% of 30-second oracle utility versus 68.52% for always admit; C64→C32 it collapses to always admit and recovers 0.58% versus 99.47% for keeping the victim.
- Experiment 9 finds exact, complete AgentX-to-vLLM identity joins and abundant repeated conversations. Structural lineage alone still ranks below random by bytes, but combined bundle+lineage context crosses the held-out gate weakly: byte-weighted AUC 0.548/0.567 and top-25%-capacity value lift 1.049×/1.320×. Same-corpus trace identity provides only a 1.15–1.37× upper bound.

The next research step is a **lineage-conditioned, horizon- and victim-aware offline utility gate**. Do not implement a live policy from the weak aggregate-ranking result.

## Experiment registry

| Experiment | Date | Status | Decision |
|---|---|---|---|
| [[01 - Experiment 1 matched next-use admission decomposition|01 — Matched next-use admission decomposition]] | 2026-08-25 | Conditionally valid diagnostic | Future-aware victim ranking produces the C32 movement result; bypass reduces churn |
| [[02 - Experiment 2 practical forced-admit policy benchmark|02 — Practical forced-admit policy benchmark]] | 2026-08-25 | Conditionally valid negative result | Do not implement the seven exact-key history policies live |
| [[03 - Experiment 3 first-admission information audit|03 — First-admission information audit]] | 2026-08-25 | Valid offline diagnostic | Exact-key history is absent at valuable C32 admissions; test request context |
| [[04 - Experiment 4 contextual request-bundle placement|04 — Contextual request-bundle placement]] | 2026-08-25 | Conditionally valid negative result | Reject tested C32 context rules; block hits and gross avoided reads conceal substitution |
| [[05 - Experiment 5 whole-bundle eviction diagnostic|05 — Whole-bundle eviction diagnostic]] | 2026-08-25 | Valid negative result | Whole-source-request FIFO/LRU granularity alone is insufficient |
| [[06 - Experiment 6 C64 independent pressure validation|06 — C64 independent pressure validation]] | 2026-08-25 | Valid trace; conditional policy replay | Opportunity and structure replicate; frozen practical rules fail held-out |
| [[07 - Experiment 7 held-out soft bundle-value ranking|07 — Held-out soft bundle-value ranking]] | 2026-08-25 | Valid negative result | Static context predicts some reused bundles but fails byte-weighted C32→C64 value ranking; do not replay live |
| [[08 - Experiment 8 held-out age-conditioned reuse hazard|08 — Held-out age-conditioned reuse hazard]] | 2026-08-25 | Valid negative result | Age and coarse prefix context collapse to pressure-specific global actions; instrument activity identity next |
| [[09 - Experiment 9 AgentX identity coverage and held-out lineage information|09 — AgentX identity coverage and held-out lineage information]] | 2026-08-25 | Valid positive-but-weak information result | Exact lineage is available offline; require deadline/victim-aware net utility before policy replay |

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

Run one more offline discriminating experiment before cache replay.

- Add the exact joined lineage features to Experiment 8's age- and deadline-conditioned arrival-versus-LRU-victim comparison.
- Freeze the policy bidirectionally between C32 and C64; do not use raw trace ID as a generalizable feature.
- Measure horizon-discounted substitution utility, candidate/victim decisions, complete-request impact, and avoided native service.
- Require improvement over the stronger simple action at the same horizon in both directions. Aggregate AUC alone is insufficient.
- If the gate fails, stop predictor elaboration on the current fields and investigate explicit application intent, tool/suspension events, route-to-data, or deterministic session placement.
- Before any live policy, propagate only the minimal privacy-safe structural fields required by the successful offline ablation into the request context and oracle trace.

## Related evidence

- [[../Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle|Finite CPU retention oracle]]
- [[../Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration|COSTAR Experiment 0 corpus calibration]]
- [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|Independent research audit and redirection]]