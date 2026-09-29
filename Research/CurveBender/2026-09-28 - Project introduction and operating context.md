---
title: CurveBender project introduction and operating context
date: 2026-09-28
type: meeting-note
topic: CurveBender
source: project introduction transcript
status: captured
---

# CurveBender project introduction and operating context

## Meeting purpose

Ashish Kamra convened the call to reduce confusion across several busy channels, understand the project and current ownership, and identify where Alberto Perdomo and Boaz Ben Shabat could contribute.

The central message was that CurveBender is a high-priority, production-facing program that needs owners for broad problem areas. It is moving faster than its processes, test coverage, and documentation.

## Origin and strategic goal

Maroon Ayoub described three related internal efforts:

1. **Lightwell** demonstrated that self-hosted open models could support useful internal work. Its workloads included build verification and legacy-code migration, with GLM models served through llm-d and vLLM on H100s.
2. **Forge** built an end-to-end agentic platform for executive workflows such as Slack summaries, email summaries, and drafting. The model is only one part of that pipeline.
3. **CurveBender** formalized the shared inference layer underneath such applications.

The stated goal was to serve roughly 80% of company inference traffic on self-hosted models, keeping closed frontier models for cases where quality or other constraints require them. Cost reduction is the immediate driver, but the project also creates a production environment for improving the open-source stack and learning how to deliver the same capabilities to customers.

## Deployment and capacity snapshot

The call described three clusters:

- one active production cluster with approximately 544 active GPUs;
- two clusters being expanded, involving H100 and H200 capacity;
- an expectation that aggregate capacity could eventually reach thousands or tens of thousands of GPUs across multiple clusters.

The exact H100/H200 allocation across the two additional clusters was unclear in the transcript and must be confirmed before reuse.

The production environment already served Lightwell, Forge preparation, and a limited Red Hat/IBM beta. The economics differ from per-token APIs: fixed accelerator capacity is economical only when utilization stays above an appropriate threshold.

## Organizational framing

CurveBender was described as formally IBM-led with Red Hat support, with both companies expected to converge their internal self-hosting plans. At the time of the call, gateway choices still differed: IBM used LightLLM while Red Hat used a different gateway stack over the same inference backend.

For Red Hat, the investment was described as substantial because the platform was expected to become a main internal inference backend. Improvements to llm-d and vLLM are valuable outcomes, but the immediate target is a reliable internal service.

## Maturity

The platform was simultaneously:

- considered production;
- serving closed-beta users;
- preparing Forge pilots for executives;
- running Lightwell continuously;
- still establishing rollout, testing, and operational practices.

Maroon characterized the team as chasing current production needs and wanting to become at least one week ahead. The system had a working model and basic production service, but every domain needed hardening.

Production operation had already exposed issues and generated upstream work in vLLM and llm-d. CPU offload and large-model configurations had produced deadlocks and other failures that were not evident from existing upstream or downstream test coverage.

## Testing gaps

### Performance testing

The project needs production-representative performance testing across real workload shapes and hardware. Existing test clusters may not transfer cleanly to production topology, but testing directly in production creates scheduling, priority, and safety questions.

Questions raised:

- What is the performance ceiling for a given deployment?
- Which workload represents actual use?
- Which metrics define success?
- Can tests run in low-priority windows or with explicit scheduling controls?
- How should Red Hat performance engineers collaborate with IBM owners?

### Quality and evaluation

An IBM-led evaluation stream was already active. The call suggested performance and quality testing could eventually share one coherent pipeline, but the interface and acceptance criteria were not defined.

### Functional testing

Functional coverage was described as critical and largely absent. Candidate areas included:

- OpenAI-compatible protocol and field coverage;
- model-specific API behavior;
- tool calling and agent-client behavior;
- routing correctness;
- KV offload behavior;
- prefill/decode disaggregation;
- failure and recovery behavior.

The participants emphasized unhappy paths. Red Hat performance and scale tools such as Kraken were named as possible foundations for chaos tests, but existing chaos work had not yet covered vLLM and llm-d.

### Downstream quality engineering gap

Ashish said the removal of a dedicated QE organization left engineering teams responsible for quality without consistent QE practices. Basic questions about test plans, coverage, and pass percentages did not have clear answers. The group agreed CurveBender should define what its production environment needs regardless of downstream product gaps.

## Observability snapshot

Maroon showed custom dashboards developed in response to live issues. The meeting snapshot included:

- about 80 requests per second;
- approximately 80–90 decode tokens per second, though the transcript does not state whether this was aggregate or normalized;
- poor P99 TTFT;
- flow-control views;
- GPU and CPU prefix-cache views;
- a dedicated deadlock dashboard;
- a summarized dashboard that was broken at the time.

A key lesson was that average KV-cache utilization does not by itself describe usable capacity. Prompt lengths were described as about 70k tokens at P50 and above 200k at P99, while the prefill context limit was around 265k. A worker might show only about 60% KV use and still be unable to admit another average request.

An experimental monitoring agent received a snapshot of system state every ten minutes, used the served GLM model to evaluate it, and posted noteworthy alerts to Slack.

## Participation guidance

Maroon advised new contributors to expect bottom-up coordination and incomplete definitions. The most useful contribution would be to own an important problem area, establish the questions and methods, and drive it forward. Isolated contributions would be less likely to have visible impact.

Suggested leadership areas included:

- streamlined, production-representative performance testing;
- functional and protocol coverage;
- chaos and unhappy-path testing;
- production-safe test execution;
- clearer metrics and dashboards.

Boaz planned to begin with the kernel-panic issue. Alberto was asked to ramp up around existing downstream-release and upstream commitments; no single deliverable was assigned during the call.

## Decisions and working conclusions

- CurveBender is a primary project, not a side task.
- The first customer is the company itself; external productization may follow.
- The team must improve the entire path from client and gateway to routing, model servers, KV-cache management, and accelerators.
- Functional and unhappy-path testing are major unmet needs.
- Production observability exists but is still evolving and includes ambiguous or broken signals.
- New contributors should take ownership of problem areas rather than wait for fully specified tickets.

## Follow-ups

- Add participants to the relevant Slack channels and meetings.
- Complete cluster onboarding and determine appropriate access boundaries.
- Share evaluation details and connect interested contributors with the evaluation owners.
- Identify Alberto's first owned area.
- Investigate the kernel panic without unsafe experimentation on production.
- Define a functional coverage matrix and a performance-testing strategy.
- Determine whether and how chaos tooling can be extended to vLLM and llm-d.

## Uncertainties from transcription

- “Hisparce” in later discussion is likely **HiCache**, but the exact feature or branch must be confirmed.
- Cluster names were transcribed inconsistently as RITS, RIX, and “rates.”
- The two additional clusters' H100/H200 allocation was not intelligible.
- Several names and gateway terms may be misspelled.
- The transcript contains a possible policy tension between “code generation not approved” and a future plan for Global Engineering coding-assistance traffic.

## Source

User-provided transcript titled “curvebender sync,” dated 2026-09-28. The recording ended at 00:39:25 and was marked as computer-generated and possibly inaccurate.