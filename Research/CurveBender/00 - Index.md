---
title: CurveBender
date: 2026-09-29
type: research-index
topic: CurveBender
status: active
repository: /Users/aperdomo/workspace/redhat/curvebender-tools
---

# CurveBender

CurveBender is the joint IBM–Red Hat effort to make self-hosted inference the default for most internal AI workloads. The stated target from the 2026-09-28 introduction was to move roughly 80% of company inference traffic to self-hosted open models where quality and other constraints permit. The immediate business case is lower external inference spend; the engineering value is operating Red Hat and IBM's own serving stack at production scale and feeding the lessons back into llm-d, vLLM, gateways, evaluation, and eventually customer-facing solutions.

## Status

**Active, production-facing, and still being built.** The platform already serves production or pre-production workloads while core deployment, testing, rollout, observability, and operational practices are being defined.

As of the 2026-09-28 introduction:

- one production cluster was reported with about 544 active GPUs;
- two additional clusters were being expanded; the transcript mentions 128 H200s and 128 H100s but does not make the allocation between the two clusters clear;
- Lightwell was running a sustained background workload;
- Forge was preparing for executive pilots;
- a limited group of Red Hat and IBM users had access;
- the longer-term expectation was thousands to tens of thousands of GPUs across multiple clusters.

These figures are meeting snapshots, not a capacity inventory. Confirm them against the current project board or cluster data before using them in planning.

## Why the project exists

IBM and Red Hat incur substantial recurring costs for closed-model inference while also building and selling open-source inference systems. CurveBender aims to replace eligible external inference traffic with company-operated models on owned or leased accelerator capacity.

The project is also a large-scale dogfooding environment. Running real workloads exposes failures and gaps that smaller upstream tests often miss, including deployment fragility, flow control, CPU KV-cache offload failures, rollout behavior, API compatibility, functional coverage, and ambiguous utilization signals.

## Internal customers and workloads

Known or proposed consumers mentioned in the introduction:

- **Lightwell** — a sustained background workload for build verification and legacy-code migration tasks; it cares more about throughput than latency.
- **Forge** — an agentic executive-assistance platform for Slack and email summaries and drafting; latency and pilot reliability matter.
- **Support portal** — cited as an internal inference customer.
- **Global Engineering** — cited as a future consumer for coding-assistance traffic.
- **General Red Hat and IBM users** — a limited beta population using the served models for varied workloads.

The introduction also said code generation was not then an approved Lightwell use case, while later discussion named coding assistance as a future CurveBender consumer. The policy boundary and timing need confirmation.

## System scope

The repository and calls show an end-to-end scope from clients to accelerators:

- client and workload replay, including Forge and Lightwell-style traffic;
- OpenAI-compatible APIs and API compatibility;
- gateway and access control, including LiteLLM;
- llm-d routing and Endpoint Picker policies;
- prefill/decode disaggregation and flow control;
- vLLM model serving, KV transfer, prefix caching, and CPU/NVMe offload;
- cluster-specific deployment manifests and image builds;
- performance, quality, functional, regression, and chaos testing;
- rollout automation, observability, dashboards, SRE, and production readiness.

The local repository is [curvebender-tools](/Users/aperdomo/workspace/redhat/curvebender-tools). Its root README describes it as deployment tooling, manifests, benchmark harnesses, and reports for internal model serving across clusters.

## Current workstreams

| Workstream | Current state on 2026-09-29 | Immediate question |
| --- | --- | --- |
| One-million-token context | PCP canary and KV events were validated and a rollout was in progress. Prefill capacity was reported as sufficient; decode capacity remained unknown. | What is the effective decode KV capacity, and what configuration safely supports the target context? |
| HiCache / active-request CPU KV capacity | Reproduced on an SCC cluster without a node crash, but without an expected speedup; NIXL registration was much slower with it enabled. | Can it be validated on a Rocky H100 environment without reproducing the kernel panic? |
| Production rollout | DisaggregatedSet rollout behavior was degrading prefix/KV-cache hit rate during live traffic. | How should revisions be rolled out without losing cache effectiveness or service quality? |
| Deployment automation | Operations owners were meeting to define requirements; an initial direction was expected quickly. | What is the smallest safe automation path that can start now? |
| API compatibility | A Nemotron-specific missing-field issue was being reproduced on llm-d Rocky; an async LiteLLM pre-call hook was proposed. | Does the scoped body injection fix the failing API shape without affecting other models? |
| Evaluation and regression | CyberGym support was added to XGENTIC for Lightwell-style evaluation; regression integration was being planned. | Which task subset represents production use, and how is it added to release regression? |
| Model performance | Initial configurations existed for Granite 5 SFT and DeepSeek; Granite RL checkpoints and DeepSeek P/D disaggregation still needed work. | Which checkpoint and serving configuration should become the accepted golden configuration? |
| SRE and production readiness | No single visible plan; some contributors lacked cluster access. | Who owns the plan, access, on-call expectations, and operational acceptance criteria? |
| Functional and chaos testing | Identified as critical and largely undefined, especially unhappy paths and protocol/feature coverage. | What production-representative matrix covers APIs, routing, offload, failures, and recovery? |
| Observability | Custom dashboards exist and continue to evolve, but several signals were ambiguous or broken. | Which metrics express usable capacity, saturation, and user impact reliably? |

## Guardrails and operating assumptions

- The primary production cluster is a real service environment. Features capable of crashing a node must be validated in a representative non-production environment before any production test.
- The production namespace has restricted access. Cluster onboarding and carefully scoped access are prerequisites for several workstreams.
- Work is being coordinated while the platform is live, so priorities and ownership may change quickly.
- Dashboard metrics need interpretation. For example, average GPU KV-cache utilization can look low while long requests prevent another request from fitting.
- The project needs owners for problem areas, not only contributors picking up isolated tickets. The 2026-09-28 advice was to treat meaningful involvement as a primary focus.
- The autogenerated transcripts contain recognition and speaker-attribution errors. Technical names below are normalized only where the repository or context makes the correction clear.

## Alberto's initial involvement

Alberto was asked to ramp up while finishing existing downstream release and upstream commitments. The clearest areas inviting leadership were:

- production-representative performance testing;
- functional and protocol coverage;
- unhappy-path and chaos testing;
- defining tests that combine quality with performance;
- helping determine how to run tests safely on production-like hardware when smaller environments are not representative.

No single owned deliverable was assigned in the introduction. Confirm current ownership on the project board before acting on the list above.

## Project map

### Durable notes

- [[2026-09-28 - Project introduction and operating context]]
- [[2026-09-29 - Daily sync - long context rollout and readiness]]

### Repository anchors

- `README.md` — repository scope and navigation.
- `deployments/` — model- and cluster-specific serving snapshots.
- `deployments/common/observability/` — generic Grafana dashboards and PodMonitors.
- `endpoint-validation/` — accuracy evaluation against OpenAI-compatible endpoints.
- `client-harnesses/forge-breifing/` — Forge transcript replay and load generation.
- `tuning/pd-flow-control/` — tested admission, fairness, and priority-reserve guidance.
- `rollouts-disagg-set/` — manual P/D rollout under traffic.
- `argocd-poc/` — deployment automation proof of concept.
- `reports/` — self-contained benchmark campaign reports.

### Confirmed repository snapshots

- Commit `c382f57` (2026-09-29) updated the root README and project map.
- Commit `6a45fdf` (2026-09-29), “GLM53 PCP Canary,” added the GLM 5.3 PCP canary, CPU KV offload, KV-event routing, PodMonitor, and supporting image changes.
- `deployments/glm-5.3/rits-roce-h100/README.md` describes a live RITS RoCE H100 snapshot captured on 2026-09-21. Treat it as a snapshot, not a portable template or a statement of today's exact production state.

## People and areas mentioned

These are meeting roles inferred from the calls, not a formal ownership directory.

- **Maroon Ayoub** — technical context, serving deployment, dashboards, llm-d/vLLM improvements, and testing gaps.
- **Ashish Kamra** — Red Hat coordination, staffing, performance/scale, quality and chaos-testing framing.
- **Carlos** — daily coordination and project-board tracking; surname not captured.
- **Rob** — PCP, HiCache/KV behavior, and rollout reliability.
- **Praveen** — reproduction and model-performance work; transcript spelling varies.
- **Mihal/Michal** — evaluation, XGENTIC, CyberGym, and regression coordination; exact spelling needs confirmation.
- **Rachel** — API compatibility reproduction and proposed request-body hook.
- **Chris and Michael** — operations, deployment, and production-readiness coordination.
- **Priya** — resource prioritization referenced for upcoming Granite 5 work.
- **Boaz Ben Shabat** — ramping into the project, initially focusing on the kernel-panic issue.
- **Michey Mehta** — project-stage, observability, and functional-testing questions.
- **Ramesh Doddaiah** — evaluation and quality-metric interest.
- **Alberto Perdomo** — ramping into performance and functional-testing work.

## Glossary

- **PCP** — prefill context parallelism, used to distribute long-context prefill work.
- **P/D** — disaggregated prefill and decode serving.
- **EPP** — Endpoint Picker / llm-d routing component.
- **NIXL** — transfer path used for KV movement between serving roles.
- **HiCache** — tiered KV-cache mechanism discussed as making CPU and GPU memory act more like a combined active-request pool. The transcript rendered the name as “Hisparce”; confirm the exact implementation name in the relevant issue or PR.
- **KV events** — cache-state events used by precise prefix-cache routing.
- **TTFT** — time to first token.
- **ITL/TPOT** — inter-token latency / time per output token.
- **RITS Rocky** — production environment name used in the calls; automated transcription also rendered it as RIX, RITS, or “rates.”
- **llm-d Rocky** — non-production Rocky-based environment proposed for safer reproduction.
- **SCC** — separate cluster used for reproduction and performance work.

## Open questions

1. What is the authoritative project charter, target date, and success metric for the “80%” goal?
2. What are the exact cluster names, current capacities, owners, and production boundaries?
3. Which model and checkpoint is primary in production today?
4. What traffic is approved for self-hosting, especially code generation and coding assistance?
5. What is the current gateway split between IBM LightLLM and Red Hat's gateway stack, and what convergence is planned?
6. What are the release gates for functionality, quality, performance, reliability, and security?
7. Which project board and Slack channels are canonical?
8. What access does Alberto need for dashboards, non-production clusters, logs, and test execution?
9. Which workstream will Alberto own first?
10. What exact failure produced the HiCache-related kernel panic, and where is its incident record?

## Source record

- User-provided “curvebender sync” transcript, 2026-09-28, 39:25. Computer-generated and explicitly marked as possibly inaccurate.
- User-provided daily-sync transcript, 2026-09-29. Computer-generated speaker labels and names are incomplete.
- Local repository `/Users/aperdomo/workspace/redhat/curvebender-tools`, branch `main`, inspected at `c382f57` on 2026-09-29.