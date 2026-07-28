# RHOAI EPP requires isolated release Gateways

## Rule

For RHOAI distributed inference using the EndpointPicker, do not attach unrelated HTTPRoutes to the same Gateway listener as an `LLMInferenceService` route.

## Why

RHOAI known issue `INFERENG-6962` documents that Istio aggregates multiple HTTPRoutes on one wildcard listener into one generated VirtualService. The generated configuration omits the per-route `ExtProcPerRoute` override, so EndpointPicker is bypassed and requests fall back to round-robin routing.

The issue applies to any additional HTTPRoute, including a token endpoint, echo service, test route, or another LLMInferenceService route. A single experiment is not safe: independent BenchFlow users can submit releases concurrently.

## BenchFlow implementation

Each RHOAI `LLMInferenceService` release receives a deterministic, compact Gateway in `openshift-ingress` and emits an explicit `spec.router.gateway.refs` reference to it. The Gateway copies the bootstrap-managed `openshift-ai-inference` Gateway's class, Istio revision label, HTTPS listener, hostname, TLS configuration, and allowed-routes policy.

The Gateway name is `benchflow-<20-hex>`, derived from deployment namespace and release name. It deliberately avoids the human-readable release prefix.

Istio derives a Deployment label and Service name from `<gateway-name>-<gateway-class>`. Kubernetes labels have a 63-character limit. The original BenchFlow names such as `benchflow-cpu-offloading-m3-82fdd75610-gateway` made the generated label longer than 63 characters, so Istio could not create the Deployment or LoadBalancer Service. The Gateway then showed `Programmed=False`, `ServiceNotFound`, and `AddressNotUsable`.

Compact names resolve that failure while preserving the correct per-Gateway isolation needed for INFERENG-6962.

## Diadochos verification

On 2026-07-28, a disposable compact Gateway probe in `openshift-ingress` became `Accepted=True` and `Programmed=True` in roughly 35 seconds. Istio created:

- Deployment `benchflow-probe-7a58dccf52-openshift-ai-inference`
- LoadBalancer Service `benchflow-probe-7a58dccf52-openshift-ai-inference`

The probe and its controller-owned resources were deleted immediately after verification.

A hostname-less listener-section alternative is not viable: Gateway API allows only one listener for the `(443, HTTPS, empty hostname)` combination, so parallel BenchFlow releases fail validation.

## Scope and verification

This applies to all BenchFlow RHOAI `LLMInferenceService` modes, including default, approximate-prefix-cache, and precise-prefix-cache. Raw `InferenceService` deployments are outside this router/EPP path.

Before accepting the routing path on a RHOAI/Istio combination, launch two concurrent precise-prefix-cache releases and verify:

- Each generated HTTPRoute has only its release Gateway parent reference.
- The two HTTPRoutes attach to distinct Gateway objects and listeners.
- Istio retains an `ExtProcPerRoute` override for each route.
- EndpointPicker logs show per-request activity and prefix-cache scoring.
- Cleanup removes only the matching release Gateway.

## Sources

- [RHOAI 3.3 known issue INFERENG-6962](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.3/html/release_notes/known-issues_relnotes)
- [RHOAI gateway discovery supports explicit Gateway selection](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/deploy_models_using_distributed_inference_with_llm-d/gateway-discovery-for-llmd)