---
title: "Phase 1 Naive Prefetch Implementation Guide"
date: "2026-08-14"
type: "implementation-guide"
experiment: "ABC"
status: "historical-rejected"
split: true
---

# Phase 1 — Naive Proactive Prefetching Implementation Guide

This historical guide documents the original post-miss same-request read-ahead implementation. The policy was rejected by the NVMe validation and superseded by the admission-time queued-request oracle design. It is split into two articles only to stay within the Arche article transport limit; no technical sections were discarded.

1. [[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide/01 - Reactive baseline through implementation|Part 1 — reactive baseline, toy design, and implementation]]
2. [[Methodology/02 - Phase 1 Naive Prefetch Implementation Guide/02 - Telemetry, run plan, and risks|Part 2 — telemetry, run plan, risks, scope, and related material]]

For the replacement design, see [[Methodology/04 - Phase 1 Queued-Request Oracle Prefetch Implementation Guide]]. For the concise conceptual comparison, see [[Methodology/05 - Initial versus Admission-Time Proactive Prefetching]].