# Ceph Orphaned OSDs After Node Disruption

When a Rook-Ceph node comes back from a disruption, the OSD prepare job may re-create OSDs with fresh IDs on the same NVMe devices. The old OSD entries remain in CRUSH and their K8s deployments crash-loop with `no disk found with OSD ID <N>`.

## Symptoms
- OSD pods in `Init:CrashLoopBackOff` / `Init:Error`
- Init container `activate` logs: `no disk found with OSD ID <N>`
- `ceph osd tree` shows duplicate entries for the same host — old OSDs `down`, reweight `0`
- `ceph osd df` shows old OSDs with `0 B` size/data
- New replacement OSDs running fine on the same node with different IDs
- CephCluster `HEALTH_WARN`: degraded data redundancy

## Diagnosis
```bash
# 1. Find crash-looping OSD pods
oc get pods -n rook-ceph | grep -E 'CrashLoop|Init:Error'

# 2. Confirm the error in init container logs
oc logs <osd-pod> -n rook-ceph -c activate --previous 2>&1 | tail -10
# → "no disk found with OSD ID <N>"

# 3. Check OSD tree for down/orphaned entries
TOOLS_POD=$(oc get pod -n rook-ceph -l app=rook-ceph-tools -o name | head -1)
oc exec -n rook-ceph $TOOLS_POD -- ceph osd tree

# 4. Verify orphaned OSDs have 0 data (CRITICAL — do not purge OSDs with data)
oc exec -n rook-ceph $TOOLS_POD -- ceph osd df
```

## Resolution
```bash
TOOLS_POD=$(oc get pod -n rook-ceph -l app=rook-ceph-tools -o name | head -1)

# 1. Purge each orphaned OSD (only after confirming 0 data, reweight 0, status down)
for osd_id in <list of orphaned IDs>; do
  oc exec -n rook-ceph $TOOLS_POD -- ceph osd purge $osd_id --yes-i-really-mean-it
done

# 2. Delete the orphaned K8s deployments
oc delete deployment -n rook-ceph rook-ceph-osd-<id1> rook-ceph-osd-<id2> ...

# 3. Verify recovery
oc exec -n rook-ceph $TOOLS_POD -- ceph status
# All remaining OSDs should be up/in, backfill progressing toward HEALTH_OK
```

## Safety Checks
- **Never purge an OSD that has data** — always verify `0 B` in `ceph osd df` first
- **Never purge an OSD that is `up`** — only purge `down` OSDs
- **Confirm replacement OSDs exist** — the same physical devices should have new running OSDs

## Related
- [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]
- [[2026-07-17 - IBM Cloud H100 node failed cannot_start_capacity on diadochos]]