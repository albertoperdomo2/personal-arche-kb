# 2026-07-17 — IBM Cloud H100 node failed, cannot_start_capacity on diadochos

## Symptom
Node `diadochos-hqxzk-gpu-h100-mt46x` went **NotReady** on the diadochos OCP cluster. Kubelet stopped posting node status at `2026-07-17T08:19:09Z`. All node conditions flipped to `Unknown` with message `Kubelet stopped posting node status`. The node was completely network-unreachable — 100% packet loss from sibling nodes, `oc adm node-logs` returned `dial tcp 10.243.65.7:10250: i/o timeout`.

Ceph (Rook-Ceph) reported **HEALTH_WARN**:
- `1/3 mons down, quorum c,e` — `mon.d` out of quorum
- `1 host (7 osds) down` — host `diadochos-hqxzk-gpu-h100-mt46x` with OSDs 35–41
- `Degraded data redundancy: 17265/51795 objects degraded (33.333%), 3 pgs degraded, 3 pgs undersized`
- 3 PGs stuck `active+undersized+degraded` for 2h+

Machine API showed `Phase: Running` (stale cache), but IBM Cloud API reported instance status as **`failed`**.

## Environment
- Cluster: **diadochos** (`api.diadochos.ibm.rhperfscale.org`, OCP 4.22, `v1.35.5`)
- Node: `diadochos-hqxzk-gpu-h100-mt46x` — IBM Cloud VPC instance `gx3d-160x1792x8h100` (8x NVIDIA H100 80GB HBM3), zone `eu-de-2`
- Instance ID: `02c7_e6e3837b-a31c-416a-a16e-01b35e0cf37f`
- Storage: Rook-Ceph in `rook-ceph` namespace (not ODF), 21 OSDs across 3 GPU nodes, CephFS provisioned
- MachineSet: `diadochos-hqxzk-gpu-h100` — 3 desired, 2 ready
- OS: RHCOS 9.8.20260617-0, kernel 5.14.0-687.15.1.el9_8.x86_64
- When it started: ~2026-07-17 08:19 UTC

## Timeline
- **08:19 UTC** — Last kubelet heartbeat from `mt46x`. Node conditions go `Unknown`.
- **08:20 UTC** — Node controller applies `unreachable` taints (`NoExecute`, `NoSchedule`).
- **~08:20 UTC** — Ceph detects `mon.d` out of quorum, 7 OSDs down. PGs go `active+undersized+degraded`.
- **~09:50 UTC** — Rook operator begins cycling `mon.d` pod — repeated create/delete attempts fail (pod stays Pending, no schedulable node).
- **~10:30 UTC** — 7 new OSD pods (35–41) created by operator, all stuck Pending.
- **~11:00 UTC** — Investigation begins. Node confirmed unreachable via ping from sibling node `6kl5z`. IBM Cloud API reveals instance status is `failed`, not `running` as Machine API reported.
- **12:09 UTC** — `ibmcloud is instance-stop` issued. Instance reaches `stopped` state within seconds.
- **12:10 UTC** — `ibmcloud is instance-start` issued. Instance goes to `starting`.
- **12:15 UTC** — Instance transitions back to `failed` with error: `cannot_start_capacity — Can't start instance because resource capacity is unavailable.`
- **12:21 UTC** — Second stop/start cycle attempted. Same result: `failed` with `cannot_start_capacity`.

### 2026-07-18 retry
- **09:38 UTC** — Third stop/start cycle attempted. Instance stopped successfully, then `instance-start` issued. Instance entered `starting` state with no immediate error.
- **~09:42 UTC** — Instance still in `starting` state after ~4 minutes — appeared promising.
- **~09:44 UTC** — Instance transitioned back to `failed` with `cannot_start_capacity`. H100 capacity in `eu-de-2` still unavailable after ~26 hours.

### 2026-07-20 retry — RESOLVED
- **10:20 UTC** — Fourth stop/start cycle attempted. Instance stopped, then `instance-start` issued.
- **~10:21 UTC** — Instance entered `starting` state with no immediate error.
- **~10:22 UTC** — Instance reached `running` state. H100 capacity in `eu-de-2` became available after ~3 days.
- **~10:24 UTC** — Node `diadochos-hqxzk-gpu-h100-mt46x` transitioned to `Ready` in OCP.
- **~10:25 UTC** — Ceph mons back in quorum (`c,d,e`). OSDs on the recovered node still re-peering; PGs degraded but recovering.

## Root Cause
The IBM Cloud VPC instance entered a `failed` state at the hypervisor level. The exact cause of the initial failure is unknown (could be hardware fault, hypervisor issue, or resource exhaustion on the host).

When attempting to restart, IBM Cloud returned `cannot_start_capacity`: there was no available H100 GPU capacity in zone `eu-de-2` to place the instance. The instance has **no reservation**, so it competes for on-demand capacity. Capacity was unavailable for ~3 days (2026-07-17 to 2026-07-20).

The OCP Machine API cached the instance status as `running` and did not detect the failure — no `MachineHealthCheck` is configured for the GPU worker MachineSet.

## Resolution
**RESOLVED** on 2026-07-20 ~10:22 UTC. Fourth stop/start cycle succeeded after H100 capacity became available in `eu-de-2`. Node is `Ready`, Ceph mons in quorum, OSDs recovering.

Total downtime: ~3 days (2026-07-17 08:19 to 2026-07-20 ~10:24 UTC).

## Prevention / Runbook
1. **Add a MachineHealthCheck** for the GPU MachineSet to detect and auto-remediate NotReady nodes:
   ```yaml
   apiVersion: machine.openshift.io/v1beta1
   kind: MachineHealthCheck
   metadata:
     name: gpu-h100-health
     namespace: openshift-machine-api
   spec:
     selector:
       matchLabels:
         machine.openshift.io/cluster-api-machineset: diadochos-hqxzk-gpu-h100
     unhealthyConditions:
       - type: Ready
         status: Unknown
         timeout: 300s
     maxUnhealthy: 1
   ```
2. **Use IBM Cloud reservations** for GPU instances to guarantee capacity on restart.
3. **Monitor IBM Cloud instance status** directly — don't rely solely on Machine API phase, which can cache stale state.

## Related
- IBM Cloud docs: [instance-status-messages#cannot-start-capacity](https://cloud.ibm.com/docs/vpc?topic=vpc-instance-status-messages#cannot-start-capacity)
- Cluster: diadochos (Performance-Scale account, `eu-de` region)
- Rook-Ceph namespace: `rook-ceph`