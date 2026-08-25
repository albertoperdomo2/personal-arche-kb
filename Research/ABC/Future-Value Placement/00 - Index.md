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

This series follows the corrected offline retention result. It treats proactive movement as a later action and first separates future-aware victim selection, admission rejection, and the information actually available to a practical policy.

## Current conclusion

[[01 - Experiment 1 matched next-use admission decomposition|Experiment 1]] shows that, under the matched common-arrival C32 replay, clairvoyant future-aware victim selection alone recovers the complete measured movement result. Forced-admit and bypass-capable next-use policies both make all 42 nonempty external targets CPU-complete and avoid all 12 native reads / 36.440 seconds. Bypass changes churn, not the measured movement outcome: it cuts admission-plus-eviction churn by 58.8%.

[[02 - Experiment 2 practical forced-admit policy benchmark|Experiment 2]] finds that seven practical exact-key history policies fail to recover that oracle opportunity. None beats recorded placement's 30/42 complete external requests. LRU and FIFO are least harmful at 27/42 but each creates three net additional incomplete requests; LRFU has the best practical block-hit rate at 67.53% but still creates four net additional misses. This demonstrates that block-hit ratio is not the correct placement objective and strongly suggests that exact-key history becomes informative too late for many first-reuse decisions.

The program remains viable as **future-value, request/prefix-aware placement research**, but generic LRU/ARC/frequency/reuse-distance replacement is not the next live implementation.\n\n[[03 - Experiment 3 first-admission information audit|Experiment 3]] explains the failure: all 116,409 valuable ordinary arrivals are first-seen keys with no prior exact-key demand. Exact-key history is therefore unavailable at the decisive admission. Future demand is strongly request-structured—83.69% of target keys come from the dominant origin—and has a 398-second median horizon. The next candidate should retain coherent request/prefix bundles using source context, not predict isolated blocks.\n\n[[04 - Experiment 4 contextual request-bundle placement|Experiment 4]] shows that the available context is not yet sufficient: bundle FIFO/LRU and source-reuse priority remain worse than recorded placement. A post-hoc prompt-band policy recovers 8 recorded reads but creates 17 new misses; its combined variant reaches 67.53% block hits while completing only 21/42 requests.\n\n[[05 - Experiment 5 whole-bundle eviction diagnostic|Experiment 5]] shows that whole-source-request eviction does not repair this. It changes hundreds of victim events and sacrifices up to 7,319 occupied slots, but changes zero request-completeness outcomes. More C32-only threshold mining is stopped; the next experiment requires an independent or enriched trace with stable session/workflow identity.

## Experiment registry

| Experiment | Date | Status | Decision |
|---|---|---|---|
| [[01 - Experiment 1 matched next-use admission decomposition|01 — Matched next-use admission decomposition]] | 2026-08-25 | Conditionally valid diagnostic | Practical future-value victim ranking is the main opportunity; bypass remains a churn optimization |
| [[02 - Experiment 2 practical forced-admit policy benchmark|02 — Practical forced-admit policy benchmark]] | 2026-08-25 | Conditionally valid negative result | Do not implement the seven history-only policies live; audit information available at first admission |\n| [[03 - Experiment 3 first-admission information audit|03 — First-admission information audit]] | 2026-08-25 | Valid offline diagnostic | Kill exact-key history as primary signal; test request-bundle coherence and source context |\n| [[04 - Experiment 4 contextual request-bundle placement|04 — Contextual request-bundle placement]] | 2026-08-25 | Conditionally valid negative result | Reject tested context rules; block hits and gross avoided reads conceal request substitution |\n| [[05 - Experiment 5 whole-bundle eviction diagnostic|05 — Whole-bundle eviction diagnostic]] | 2026-08-25 | Valid negative result | Reject FIFO/LRU whole-source-request eviction; obtain independent/enriched trace |

## Source corpus

- MLflow run: [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow)
- Normalized database: `/private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite`
- SHA-256: `788adedcbc9850a64443681fbd8acf36ea62cdce950eeac6311f730f34cce687`
- Requests: 901 total; 42 with nonempty external targets
- CPU capacity: 131,072 KV chunks
- Native secondary reads: 12 jobs / 36.440469092 seconds measured service

## Next experiment

Run an offline first-admission information audit. Label which arrivals contribute to future complete requests and measure whether request identity, prefix position, prompt/decode origin, neighboring-prefix state, session/category context, queue position, time-to-demand, and reload cost are available early enough to rank them.

The goal is not to train a complex model immediately. It is to determine whether simple request/prefix-aware signals contain enough information to bridge the gap between recorded placement (30/42 complete) and clairvoyant placement (42/42), and exactly which missing trace fields must be instrumented before the next live policy.

## Related evidence

- [[../Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle|Finite CPU retention oracle]]
- [[../Reports/2026-08-25 - COSTAR Experiment 0 oracle corpus calibration|COSTAR Experiment 0 corpus calibration]]
- [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|Independent research audit and redirection]]