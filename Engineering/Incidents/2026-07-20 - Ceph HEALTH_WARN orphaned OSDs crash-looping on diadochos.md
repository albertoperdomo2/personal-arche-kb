# 2026-07-20 — Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos

## Symptom
CephCluster `rook-ceph` in namespace `rook-ceph` reporting `HEALTH_WARN` with message:
```
Degraded data redundancy: 189457/653691 objects degraded (28.983%), 1 pg degraded, 1 pg undersized
```
CephCluster phase stuck at `Progressing` ("Processing OSD 31"). Seven OSD pods (35–41) on node `diadochos-hqxzk-gpu-h100-mt46x` in `Init:CrashLoopBackOff` / `Init:Error` with init container error:
```
no disk found with OSD ID 35
```
(same pattern for OSDs 36–41).

## Environment
- Cluster: `diadochos` (`api.diadochos.ibm.rhperfscale.org:6443`)
- Node: `diadochos-hqxzk-gpu-h100-mt46x` (8×H100 GPU node, IBM Cloud eu-de-2)
- Component: Rook Ceph v19.2.4 (squid), rook-ceph namespace
- Storage: 3 nodes × 7 NVMe (`^nvme[1-7]n1$`), 21 OSDs expected per healthy state
- When: Observed 2026-07-20; previous health was `HEALTH_ERR`, transitioned to `HEALTH_WARN` on 2026-07-19

## Timeline
1. Node `mt46x` likely experienced a disruption (possibly related to the [[2026-07-17 - IBM Cloud H100 node failed cannot_start_capacity on diadochos|2026-07-17 capacity incident]]).
2. When the node came back, Rook's OSD prepare job re-scanned the NVMe disks and created new OSDs 0–6 on the same physical devices that previously held OSDs 35–41.
3. OSDs 0–6 started successfully and began backfilling data.
4. Old OSD deployments (35–41) remained in Kubernetes and kept trying to activate, but their `ceph-volume lvm list` no longer contained their OSD IDs → `Init:Error` → `CrashLoopBackOff`.
5. CRUSH map still listed OSDs 35–41 under host `mt46x` (down, reweight 0, 0 B data), doubling the host's CRUSH weight to ~97 TiB vs the expected ~49 TiB.

## Root Cause
NVMe disks on `mt46x` were re-prepared by Rook with fresh OSD IDs (0–6), replacing the previous IDs (35–41). The old OSD entries were never cleaned up — they remained in the CRUSH map and their Kubernetes deployments kept crash-looping because the backing devices no longer mapped to those OSD IDs.

## Resolution
1. Verified OSDs 35–41 had reweight 0, 0 B data, status `down` — confirmed safe to purge.
2. Purged all 7 orphaned OSDs from Ceph:
```bash
TOOLS_POD=$(oc get pod -n rook-ceph -l app=rook-ceph-tools -o name | head -1)
for osd_id in 35 36 37 38 39 40 41; do
  oc exec -n rook-ceph $TOOLS_POD -- ceph osd purge $osd_id --yes-i-really-mean-it
done
```
3. Deleted the orphaned Kubernetes deployments:
```bash
oc delete deployment -n rook-ceph \
  rook-ceph-osd-35 rook-ceph-osd-36 rook-ceph-osd-37 \
  rook-ceph-osd-38 rook-ceph-osd-39 rook-ceph-osd-40 rook-ceph-osd-41
```
4. Post-purge: OSD tree clean (21 OSDs, all up/in), CRUSH weight corrected to ~147 TiB, backfill continuing at ~155 MiB/s. Cluster expected to reach `HEALTH_OK` once backfill completes (~20–30 min).

## Prevention / Runbook
- After a node disruption that triggers OSD re-preparation, check for orphaned OSD deployments:
  ```bash
  oc get pods -n rook-ceph | grep -E 'CrashLoop|Init:Error'
  ```
- Compare `ceph osd tree` entries against running OSD pods — any `down`/reweight-0 OSDs with crash-looping pods are candidates for purge.
- Verify with `ceph osd df` that the OSDs have 0 B data before purging.
- Consider monitoring for OSD count mismatches between `ceph osd tree` and healthy running pods.

## Related
- Prior incident: [[2026-07-17 - IBM Cloud H100 node failed cannot_start_capacity on diadochos]]
- Cluster: diadochos (IBM Cloud eu-de-2, H100 GPU nodes)