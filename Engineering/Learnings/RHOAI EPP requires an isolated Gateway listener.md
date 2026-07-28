# RHOAI EPP requires an isolated Gateway listener

## Rule

For RHOAI distributed inference using the EndpointPicker, do not attach unrelated HTTPRoutes to the same Gateway listener section as an `LLMInferenceService` route.

## Why

RHOAI known issue `INFERENG-6962` documents that Istio aggregates multiple HTTPRoutes on one wildcard listener into one generated VirtualService. The generated configuration omits the per-route `ExtProcPerRoute` override, so EndpointPicker is bypassed and requests fall back to round-robin routing.

The issue applies to any additional HTTPRoute, including a token endpoint, echo service, test route, or another LLMInferenceService route. A single experiment is not safe: independent BenchFlow users can submit releases concurrently.

## BenchFlow implementation

BenchFlow emits an explicit `spec.router.gateway.refs` entry for every RHOAI `LLMInferenceService`. The reference selects a deterministic, release-scoped `sectionName` on the bootstrap-managed `openshift-ai-inference` Gateway in `openshift-ingress`.

Each release listener:

- Copies the shared Gateway's working HTTPS TLS and allowed-routes policy.
- Omits the hostname. Gateway API rejects a duplicate hostname/port/protocol listener, while the hostname-less listener is a distinct valid section.
- Is appended with an atomic JSON Patch, without replacing concurrent release listeners.
- Is removed by a JSON Patch that first tests the listener name at the selected index, preventing concurrent cleanup from deleting another release's listener.

Bootstrap reuses an existing shared Gateway instead of applying its base single-listener manifest again, preventing later bootstrap runs from erasing active release listeners.

## Diadochos finding

A release-scoped Gateway object is not viable on Diadochos. The OpenShift GatewayClass accepted an arbitrary Gateway but left it `Programmed=False` with:

- `ServiceNotFound`: the LoadBalancer Service resource was missing.
- `AddressNotUsable`: no corresponding service DNS name could be resolved.

The existing shared Gateway has the only provisioned load-balancer Service. A disposable hostname-less listener appended to that Gateway was immediately `Accepted=True` and `Programmed=True`, using the existing load balancer, DNS, and TLS configuration. The disposable listener was removed after the test.

## Scope and verification

This applies to all BenchFlow RHOAI `LLMInferenceService` modes, including default, approximate-prefix-cache, and precise-prefix-cache. Raw `InferenceService` deployments are outside this router/EPP path.

The listener lifecycle and rendering are locally validated. Before accepting the routing path on a RHOAI/Istio combination, launch two concurrent precise-prefix-cache releases and verify:

- Each generated HTTPRoute selects only its release listener section.
- Istio retains an `ExtProcPerRoute` override for each route.
- EndpointPicker logs show per-request activity and prefix-cache scoring.
- Cleanup removes only the matching listener.

## Sources

- [RHOAI 3.3 known issue INFERENG-6962](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.3/html/release_notes/known-issues_relnotes)
- [RHOAI gateway discovery supports explicit Gateway selection](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/deploy_models_using_distributed_inference_with_llm-d/gateway-discovery-for-llmd_gateway-discovery)