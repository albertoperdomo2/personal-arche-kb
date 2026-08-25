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
2. [[Reports/COSTAR Offline Oracle/02 - L0 single-request transfer feasibility|Gate 1 — L0 single-request feasibility]] — **passed as arithmetic:** the largest 15.12 GiB target needs about 6.05 seconds at 2.5 GiB/s; admission is far too late.
3. [[Reports/COSTAR Offline Oracle/03 - Global relaxed retention diagnosis|Gate 2 — global relaxed retention]] — **diagnostic:** infinite retention prevents 889/891 deferrals with zero proactive reads, but requires 2.21× real CPU capacity.
4. **Gate 3 — finite-CPU retention and eviction oracle:** in progress. Enforce 131,072 slots and compare native/LRU behavior with clairvoyant next-use placement before adding proactive reads.

The accepted source corpus is MLflow run [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow).

Return to [[Reports/00 - Index|ABC experiment reports]].