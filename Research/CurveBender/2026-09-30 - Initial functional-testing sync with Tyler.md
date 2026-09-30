---
title: Initial functional-testing sync with Tyler
date: 2026-09-30
type: meeting-note
topic: CurveBender
status: active
participants:
  - Alberto Perdomo
  - Tyler
date_confidence: inferred-from-conversation-context
---

# Initial functional-testing sync with Tyler

## Context

Tyler and Alberto aligned on ownership, implementation direction, execution environments, and longer-term operation of the CurveBender test suite.

The meeting date was not stated in the supplied transcript. This note uses 2026-09-30 from the surrounding conversation context; correct it if the sync occurred on another date.

## Decisions

### Functional tests are the immediate priority

Both contributors should focus first on function tests. System tests are the next high-impact area because they prove that deployed manifests produced the intended live stack. Static validation remains useful, but Tyler considers it lower priority while many deployment assets are still changing rapidly.

This matches the direction attributed to Maroon and Rob: find client-visible failures quickly, then confirm the deployed system matches its declaration.

### Start with a small Python package and CLI

The initial implementation should be plain Python and remain easy to run:

- from a developer laptop against supplied endpoints;
- inside a pod in a cluster;
- from CI or an external scheduler.

The first version should not be a Kubernetes controller. A controller would introduce CRDs, reconciliation logic, and operational work that does not help deliver protocol coverage quickly.

The code should remain modular enough to migrate into or integrate with a more durable llm-d project later.

### Keep future upstream homes open

No single existing project was known to solve the whole problem. Two possible longer-term homes were discussed:

- **llm-d-benchmark**, which already plans and creates deployments, runs benchmarks, and provides smoke tests used in llm-d CI;
- **llm-d Lens**, an incubation project intended to connect deployment, testing, infrastructure checks, dashboards, and agent integrations.

The immediate repository should solve CurveBender's needs without tightly coupling itself to CurveBender manifests or building infrastructure that would make migration difficult.

### Cluster access is unresolved

Cluster roles, environments, and permitted actions were unclear. Tyler expected a Friday meeting with Carlos, Daryl, and cluster maintainers to clarify:

- which clusters are staging and production;
- who can access each cluster;
- what tests and changes are allowed;
- where models are hosted;
- how scarce accelerator capacity should be shared.

The transcript names the maintainers as Chris Malight and Mike Desmone; spellings should be confirmed.

This is the largest dependency for production and system testing. It does not block endpoint-only functional work.

Alberto reported access to internal Red Hat H100 and H200 clusters and can deploy llm-d there for initial validation while CurveBender-specific access is resolved.

### Scheduling, alerts, and result storage are required design seams

The long-term suite is expected to support scheduled and rollout-triggered execution, including possibilities such as:

- hourly or every few hours;
- nightly;
- immediately after a rollout.

Alerting should be pluggable rather than hard-coded. Slack is the first obvious target, with room for email or other notifiers later.

Result storage must work under uneven cluster permissions:

1. always write portable flat-file artifacts that can be copied from a container;
2. optionally persist to a PVC where permitted;
3. later support an external object store or messaging pipeline.

Alberto emphasized that each run should preserve the deployment manifests used at that time alongside its results. This is necessary to connect a failure to the exact deployment rather than to the repository's later state.

## Ownership and coordination

- Tyler planned to break the umbrella work into smaller GitHub issues and expose the initial directory structure.
- Alberto planned to inspect the functional-test repository and begin the function-test foundation.
- Work should be announced before starting so contributors do not duplicate or overwrite each other's changes.
- Detailed design discussion should continue in the issues.
- The initial split is by practical work rather than rigid categories, with both contributors centered on functional coverage.

## Implementation consequences

The first implementation should have these boundaries:

### Core library

- profile loading and validation;
- logical ingress definitions;
- reusable OpenAI and Anthropic raw HTTP/SSE clients;
- case registry and applicability rules;
- normalized assertion and result models;
- credential redaction.

### CLI

A CLI should accept a profile, endpoint overrides, selected cases or client contracts, output directory, and safety markers. It should return a nonzero exit status when a release-blocking expected case fails.

### Execution adapters

Keep execution outside the test semantics:

- local process;
- Kubernetes Job/Pod wrapper;
- CI job;
- future llm-d-benchmark or Lens adapter.

The core suite should not create a controller or own scheduling.

### Output bundle

Each run should emit a self-contained directory containing:

- machine-readable per-case results;
- JUnit XML;
- human-readable summary;
- redacted request/response evidence for failures;
- capability profile;
- component and SDK versions;
- endpoint names without credentials;
- rendered manifests or an immutable manifest bundle reference;
- Git revisions, image digests, timestamps, and run ID;
- logs from the test process;
- checksum manifest for the bundle.

The bundle must still be useful when no PVC or external store is available.

### Extension interfaces

Define small interfaces for:

- result sinks: local files first, then PVC-mounted directories or object storage;
- notifications: no-op first, then Slack and other targets;
- environment metadata collection;
- optional deployment/system-test adapters.

Implementing every backend is not required for the first protocol cases, but the core should not print results directly in a way that prevents these adapters.

## Immediate work sequence

1. Inspect the functional-test repository and its proposed layout.
2. Land the Python package skeleton, CLI, profile schema, and local artifact writer.
3. Implement the smallest release-blocking cases from [[2026-09-30 - Functional API compatibility matrix]]:
   - model discovery;
   - Chat Completions non-streaming;
   - Chat Completions SSE and terminal usage;
   - one forced tool call;
   - Anthropic Messages non-streaming and streaming;
   - request ID and core error envelopes.
4. Validate against Alberto's accessible Red Hat H100/H200 llm-d deployment.
5. After the cluster-access meeting, define CurveBender staging and production profiles and the allowed system tests.
6. Add scheduled execution, notification, and persistent result sinks after the case runner and artifacts are stable.

## Open questions

1. What is the exact repository URL and current issue/directory structure for `curvebender-functional-tests`?
2. Which tests are safe against production, and which require staging or an isolated namespace?
3. Which cluster and access model will be the first official CurveBender validation target?
4. Is llm-d-benchmark or llm-d Lens the preferred eventual home, and what extension API should this repository match?
5. Which scheduler owns nightly and post-rollout runs?
6. Where should durable artifacts live when PVC creation is unavailable?
7. Which alert channel and severity policy should be enabled first?
8. What is the minimum manifest/config fingerprint required to reproduce a failed run?

## Related documents

- [[2026-09-30 - Functional test strategy review]]
- [[2026-09-30 - Functional API compatibility matrix]]

## Source record

- User-provided full transcript of Alberto Perdomo's initial sync with Tyler.
- The transcript contains automated or informal name renderings. The identity of Speaker 1 as Tyler was supplied explicitly by Alberto.