---
title: "COSTAR offline oracle experiment series"
date: "2026-08-25"
type: "research-reports-index"
experiment: "ABC / COSTAR-KV offline oracle"
status: "active"
---

# COSTAR offline oracle experiment series

This series separates the scientific gates used to decide whether KV-cache prediction, movement, or retention has physically recoverable value. Each report records the model relaxation explicitly so optimistic evidence is not mistaken for a deployable result.

1. [[Reports/COSTAR Offline Oracle/01 - Experiment 0 normalized native replay|Gate 0 — normalized native replay]] — **passed:** 901 request and 1,435 transfer lifecycles close; capacity conserves exactly.
2. [[Reports/COSTAR Offline Oracle/02 - L0 single-request transfer feasibility|Gate 1 — L0 single-request feasibility]] — **passed as arithmetic:** the largest corrected 14.28 GiB external target needs about 5.71 seconds at 2.5 GiB/s; admission is far too late.
3. [[Reports/COSTAR Offline Oracle/03 - Global relaxed retention diagnosis|Gate 2 — global relaxed retention]] — **corrected diagnostic:** 116,409 unique externally reused keys fit in 88.81% of CPU capacity; ideal retention avoids all 12 native reads with zero proactive reads.
4. [[Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle|Gate 3 — finite-CPU retention diagnostic]] — **passed, interpretation corrected:** exact physical replay predicts native movement with 0/898 mismatches; next-use placement avoids 12/12 reads and 36.44 seconds. [[Future-Value Placement/01 - Experiment 1 matched next-use admission decomposition|The matched decomposition]] later showed forced admission obtains the same movement result; bypass reduces churn but is not required for read avoidance.

The accepted source corpus is MLflow run [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow).

Return to [[Reports/00 - Index|ABC experiment reports]].
## Current decision

[[Future-Value Placement/01 - Experiment 1 matched next-use admission decomposition|Experiment 1]] redirects the immediate question toward practical future-value **victim selection**: forced-admit next-use and bypass-capable next-use both recover the complete C32 movement result. Admission filtering remains a secondary churn/overhead axis. Do not build another block prefetcher yet.