# RHOAI EPP requires an isolated Gateway listener

## Rule

For RHOAI distributed inference using the EndpointPicker, do not attach unrelated HTTPRoutes to the same Gateway listener as an `LLMInferenceService` route.

## Why

RHOAI known issue `INFERENG-6962` documents that Istio aggregates multiple HTTPRoutes on one wildcard listener into one generated VirtualService. The generated configuration omits the per-route `ExtProcPerRoute` override, so EndpointPicker is bypassed and requests fall back to round-robin routing.

The issue applies to any additional HTTPRoute, including a token endpoint, echo service, test route, or another LLMInferenceService route.

## BenchFlow implementation

BenchFlow previously emitted `router.gateway: {}`, which selected the shared `openshift-ai-inference` Gateway. This is unsafe whenever another route can coexist, including independent BenchFlow users and concurrent matrix children.

BenchFlow now gives every RHOAI `LLMInferenceService` release a deterministic, release-scoped Gateway in `openshift-ingress` and emits an explicit `spec.router.gateway.refs` reference. The deployment applies and waits for that Gateway before the LLMInferenceService; cleanup removes it by the same derived name even when the service no longer exists.

The release Gateway copies the bootstrap-managed Gateway's class, Istio revision label, HTTPS listener, hostname, TLS configuration, and allowed-routes policy. This is intentional:

- The RHOAI workaround is to use a separate Gateway, not to require a new public hostname.
- Diadochos' `data-science-gateway-service-tls` certificate has only internal service DNS SANs, so a per-release external hostname would be invalid without new cluster TLS infrastructure.
- A distinct Gateway object and listener preserves the existing endpoint/TLS path while isolating BenchFlow-generated routes.

This is applied to all BenchFlow RHOAI `LLMInferenceService` modes, not only precise-prefix-cache and not only matrix runs. Raw `InferenceService` deployments are outside this routing path.

## Verification Required

Validate each RHOAI/Istio combination with two concurrent precise-prefix-cache releases:

- Each generated HTTPRoute has only its release Gateway parent reference.
- The routes attach to distinct Gateway objects/listeners.
- Istio retains an `ExtProcPerRoute` override for each route.
- EndpointPicker logs show per-request activity and prefix-cache scoring.
- Cleanup removes only the matching release Gateway.

## Sources

- [RHOAI 3.3 known issue INFERENG-6962](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.3/html/release_notes/known-issues_relnotes)
- [RHOAI gateway discovery supports explicit Gateway selection](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/deploy_models_using_distributed_inference_with_llm-d/gateway-discovery-for-llmd_gateway-discovery)