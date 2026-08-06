---
title: "RHOAI DSC NotReady: dual controller ownerRef on odh-model-controller ServiceAccount"
date: 2026-08-06
type: incident
cluster: diadochos
platform: RHOAI 3.5.0
---

# RHOAI DSC NotReady: dual controller ownerRef on odh-model-controller SA

## Symptom

`DataScienceCluster` phase `Not Ready`. The `KserveReady` condition shows `DeployFailed`:

```
deploy: applying owned resources: failure deploying resource redhat-ods-applications/odh-model-controller:
  apply failed /v1, Kind=ServiceAccount: unable to patch /v1, Kind=ServiceAccount
  redhat-ods-applications/odh-model-controller: ServiceAccount "odh-model-controller" is invalid:
  metadata.ownerReferences: Invalid value: [...]:
  Only one reference can have Controller set to true.
  Found "true" in references for ModelController/default-modelcontroller and Kserve/default-kserve
```

The `odh-model-controller` ServiceAccount, Deployment, and all KServe controller deployments (`kserve-controller-manager`, `llmisvc-controller-manager`) were missing from `redhat-ods-applications`.

## Root cause

The cluster was upgraded from RHOAI 3.5.0-ea.2 to 3.5.0 GA. The EA operator created a separate `ModelController` CR (`components.platform.opendatahub.io/v1alpha1`) with annotation `platform.opendatahub.io/version: 3.5.0-ea.2`. The GA operator does **not** use a separate `ModelController` CR — the model controller deployment is owned directly by the `Kserve` module.

The stale EA `ModelController` CR persisted after the upgrade. When the GA Kserve reconciler tried to create the `odh-model-controller` ServiceAccount, it built an ownerReferences list containing both `Kserve` (controller: true) and the stale `ModelController` (controller: true). Kubernetes rejects multiple controller ownerReferences, so the SA creation failed, blocking all downstream deployments.

## Resolution

1. Deleted the stale `ModelController` CR: `oc delete modelcontroller default-modelcontroller`
2. Deleted the stuck `Kserve` CR (removal of `platform.opendatahub.io/finalizer` was required because the finalizer handler was stuck): `oc patch kserve default-kserve --type=json -p='[{"op":"remove","path":"/metadata/finalizers"}]'`
3. The DSC controller recreated a fresh `Kserve` CR with a new UID.
4. The GA operator deployed all resources cleanly — `odh-model-controller`, `kserve-controller-manager`, `llmisvc-controller-manager`, `model-serving-api` all came up 1/1.
5. DSC phase transitioned to `Ready`.

**Note**: Deleting only the `ModelController` CR was not sufficient. The Kserve component reconciler was stuck in an error backoff state and did not re-reconcile even after operator pod restarts and forced DSC annotation changes. Deleting the Kserve CR itself was necessary to get a clean reconciliation.

## Key observations

- The `ModelController` CR is an EA-era artifact. The GA 3.5.0 operator does not recreate it — confirming that the model controller functionality was folded into the Kserve module for GA.
- The Kserve component reconciler did not recover from the deploy failure on its own — no re-reconciliation was observed for over 1.5 hours, even after operator pod restarts.
- The `platform.opendatahub.io/finalizer` on the Kserve CR blocked deletion indefinitely and had to be manually removed.

## Prevention

After upgrading RHOAI from EA to GA, check for orphaned component CRs that the GA operator no longer manages:

```bash
oc get modelcontroller
```

If a `ModelController` CR exists with an EA version annotation, delete it before or immediately after the GA upgrade.