---
title: Routing Ceph over 200G NICs on OCP with PCI passthrough VFs
date: 2026-07-24
type: learning
cluster: diadochos
---

# Routing Ceph over 200G NICs on OCP with PCI passthrough VFs

Ceph data traffic on the diadochos OCP cluster was migrated from the ~10 Gbps OVN overlay (management NIC) to the 200 Gbps ConnectX-7 NICs. This required a big-bang MON migration with ~15 minutes of CephFS downtime. Three earlier approaches failed, two of them breaking the cluster and requiring monmap repair.

## Goal

Route all Ceph data-path traffic (OSD replication, client-to-OSD, client-to-MDS) over the 200G fabric instead of the 10G management NIC used by OVN overlay. MON control-plane traffic on the management NIC is acceptable since it's only heartbeats and map updates.

## Environment

- **Cluster**: diadochos (`api.diadochos.ibm.rhperfscale.org:6443`), OCP 4.22, IBM Cloud
- **Nodes**: 3× `gx3d-160x1792x8h100` GPU nodes + 1 non-GPU node (gjfjh) with GPU label
- **Rook-Ceph**: v1.20.2 operator, Ceph v19.2.4 (Squid), deployed via Helm
- **Storage**: 21 NVMe OSDs (7 per GPU node), data pool `replicated.size: 1`
- **Workload**: CephFS RWX volumes for vLLM KV cache offloading (`TieringOffloadingSpec`)

### Network topology

| Network | Interface | Subnet | Speed | MTU | Used by |
|---------|-----------|--------|-------|-----|---------|
| OVN overlay (management) | `enp3s0` via `br-ex` | 10.243.65.0/24 | ~10 Gbps | ~1400 | Kubernetes pods, node communication |
| 200G fabric (data) | `enp163s0`–`enp233s0` | 10.0.0.0/16 – 10.7.0.0/16 | 200 Gbps | 9000 | PCI passthrough VFs from hypervisor |

### 200G NIC IPs (enp163s0)

| Node | Management IP | 200G IP | Ceph role |
|------|--------------|---------|-----------|
| fx7c8 | 10.243.65.5 | 10.0.0.6 | MON-e, 7 OSDs |
| mt46x | 10.243.65.7 | 10.0.0.7 | 7 OSDs |
| 6kl5z | 10.243.65.9 | 10.0.0.4 | MON-l, 7 OSDs |
| gjfjh | 10.243.65.15 | 10.0.0.8 | MON-k, MGR, MDS, operator |

### Key constraint: PCI passthrough VFs

The 200G NICs are **PCI passthrough virtual functions** from the IBM Cloud hypervisor, not physical NICs. This means:

- **SR-IOV is NOT available** — can't nest VFs inside VFs. `mlx5Gen Virtual Function` confirmed via `lspci`.
- **PCI passthrough traffic bypasses VPC security groups** — the hypervisor doesn't inspect traffic on passthrough devices. Cross-node 200G connectivity is wide open with no firewall.
- **NMState is NOT installed** — IP configuration is done at the OS level via NetworkManager, not via Kubernetes CRDs.

## Approaches tested

### 1. Multus ipvlan L2 — FAILED

Created a `NetworkAttachmentDefinition` to give Ceph pods a second interface on the 200G NIC via ipvlan L2 mode.

**Why it failed**: Host cannot ARP-resolve its own ipvlan children — fundamental Linux kernel limitation. The host sends ARP for the ipvlan child IP and gets zero responses. This breaks the CephFS kernel client → OSD path since the kernel client runs in the host network namespace and must reach OSD pods directly.

Also tested:
- **ipvlan L3 mode**: Same limitation at a different layer
- **Separate subnet (192.168.200.0/24)**: OVN hijacked routing — `ip route get 192.168.200.x` went through `br-ex` instead of `enp163s0`

### 2. Host networking without big-bang MON migration — FAILED (broke cluster)

Set `network.provider: host` + `public_network = 10.0.0.0/16` on the live cluster while MONs were still on the pod network (ClusterIP addresses).

**Why it failed**: OVN-networked pods have NO route to 10.0.0.0/16 (`ip route get 10.0.0.7` → "no route to host"). Newly created MGR/MDS daemons advertised 200G IPs and became unreachable from all OVN pods. Old MONs on ClusterIPs couldn't reach new MONs on 200G IPs. PGs went "unknown."

**Cluster damage**: MON quorum broken — mixed pod-network / host-network IPs couldn't communicate. Required monmap injection repair (see recovery procedure below).

**Key discovery**: Rook operator does NOT update existing MON/OSD deployments with `hostNetwork: true` — it only applies to newly created deployments. All 21 existing OSD deployments had to be manually patched.

### 3. Host networking with management subnet only — FAILED

Set `public_network = 10.243.65.0/24` so Ceph binds to the management IP (reachable from OVN pods).

**Why it failed**: OCP's OVN firewall blocks pod-to-nodeIP traffic on non-standard ports. Confirmed with `nc -zv`:

| Port | Service | Cross-node from pod |
|------|---------|-------------------|
| 22 | SSH | Reachable |
| 6789 | MON v1 | Blocked |
| 3300 | MON v2 | Blocked |
| 6800+ | OSD/MDS | Blocked |

IBM Cloud VPC security groups add a second layer of blocking on the same ports.

### 4. Host networking with big-bang MON migration — SUCCEEDED

The key insight: since 200G PCI passthrough traffic bypasses all firewalls, and the CephFS kernel client runs in the host network namespace (which has direct access to 200G NICs), the only networking that matters is host-to-host on the 200G fabric. All Ceph daemons on `hostNetwork` bind to 200G IPs via `public_network = 10.0.0.0/16`, and the CephFS kernel client reaches them directly.

**Why big-bang**: Gradual MON migration is impossible because old MONs (OVN, 172.30.x.x ClusterIPs) can't reach new MONs (200G, 10.0.x.x). Quorum requires bidirectional communication. All MONs must switch at once.

## Migration procedure (what was actually done)

### Phase 0: Pre-flight (no disruption)

Patched control-plane pods with `hostNetwork: true` so they can reach 200G IPs. Safe — pods gain host namespace interfaces while continuing to work via current ClusterIP MON addresses.

```bash
# Operator
oc patch deploy -n rook-ceph rook-ceph-operator --type strategic \
  -p '{"spec":{"template":{"spec":{"hostNetwork":true,"dnsPolicy":"ClusterFirstWithHostNet"}}}}'

# Tools
oc patch deploy -n rook-ceph rook-ceph-tools --type strategic \
  -p '{"spec":{"template":{"spec":{"hostNetwork":true}}}}'

# CSI controller
oc patch driver.csi.ceph.io -n rook-ceph rook-ceph.cephfs.csi.ceph.com --type merge \
  -p '{"spec":{"controllerPlugin":{"hostNetwork":true}}}'
```

Updated `02-ceph-cluster.yaml`:
```yaml
network:
  provider: host
  addressRanges:
    public:
      - "10.0.0.0/16"
cephVersion:
  image: quay.io/ceph/ceph:v19.2.4  # Pin to avoid upgrade deadlock
```

### Phase 1: MON migration (service disruption begins)

1. Scale operator to 0:
   ```bash
   oc scale deploy -n rook-ceph rook-ceph-operator --replicas=0
   ```

2. Set public_network in Ceph config:
   ```bash
   oc exec -n rook-ceph deploy/rook-ceph-tools -- \
     ceph config set global public_network 10.0.0.0/16
   ```

3. Scale all MONs to 0:
   ```bash
   oc scale deploy -n rook-ceph rook-ceph-mon-h rook-ceph-mon-i --replicas=0
   oc scale deploy -n rook-ceph rook-ceph-mon-e --replicas=0
   ```

4. Monmap repair via privileged pod on fx7c8 (mount `/var/lib/rook/mon-e`):
   ```bash
   ceph-mon --extract-monmap /tmp/monmap --mon-data /var/lib/rook/mon-e/data
   monmaptool /tmp/monmap --rm h
   monmaptool /tmp/monmap --rm i
   monmaptool /tmp/monmap --rm e
   monmaptool /tmp/monmap --addv e [v2:10.0.0.6:3300/0,v1:10.0.0.6:6789/0]
   monmaptool /tmp/monmap --print  # verify
   ceph-mon --inject-monmap /tmp/monmap --mon-data /var/lib/rook/mon-e/data
   ```

5. Update ConfigMap `rook-ceph-mon-endpoints`:
   ```bash
   oc patch configmap -n rook-ceph rook-ceph-mon-endpoints --type merge \
     -p '{"data":{"data":"e=10.0.0.6:6789","csi-cluster-config-json":"[{\"clusterID\":\"rook-ceph\",\"monitors\":[\"10.0.0.6:6789\"]}]","mapping":"{\"node\":{\"e\":{\"Name\":\"diadochos-hqxzk-gpu-h100-fx7c8\",\"Hostname\":\"diadochos-hqxzk-gpu-h100-fx7c8\",\"Address\":\"10.0.0.6\"}}}"}}'
   ```

6. Update Secret `rook-ceph-config`:
   ```bash
   MON_HOST=$(echo -n '[v2:10.0.0.6:3300,v1:10.0.0.6:6789]' | base64)
   MON_MEMBERS=$(echo -n 'e' | base64)
   oc patch secret -n rook-ceph rook-ceph-config --type merge \
     -p "{\"data\":{\"mon_host\":\"$MON_HOST\",\"mon_initial_members\":\"$MON_MEMBERS\"}}"
   ```

7. Patch MON-e deployment — add hostNetwork and fix bind addresses:
   ```bash
   # Get current args, replace --public-addr and --public-bind-addr with 200G IP
   # Remove: --public-addr=172.30.181.75 (old ClusterIP)
   # Remove: --public-bind-addr=$(ROOK_POD_IP) (resolves to management IP)
   # Add: --public-addr=10.0.0.6 --public-bind-addr=10.0.0.6
   oc patch deploy -n rook-ceph rook-ceph-mon-e --type strategic \
     -p '{"spec":{"template":{"spec":{"hostNetwork":true,"dnsPolicy":"ClusterFirstWithHostNet"}}}}'
   # Then JSON patch the container args (see gotcha #1 below)
   ```

8. Scale MON-e back up and verify:
   ```bash
   oc scale deploy -n rook-ceph rook-ceph-mon-e --replicas=1
   oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph -m 10.0.0.6 status
   ```

9. Delete old MON deployments and Services.

### Phase 2: OSD migration

```bash
for osd in 0 1 2 3 4 5 6 9 10 16 24 25 26 27 28 29 30 31 32 33 34; do
  oc patch deploy -n rook-ceph rook-ceph-osd-$osd --type strategic \
    -p '{"spec":{"template":{"spec":{"hostNetwork":true,"dnsPolicy":"ClusterFirstWithHostNet"}}}}'
done
```

OSDs automatically bind to 200G IPs via `public_network = 10.0.0.0/16`. No args patching needed — OSDs don't have `--public-addr` in their args (unlike MON/MGR).

### Phase 3: MGR and MDS migration

```bash
# MGR — CRITICAL: must also remove --public-addr from args
oc patch deploy -n rook-ceph rook-ceph-mgr-a rook-ceph-mgr-b --type strategic \
  -p '{"spec":{"template":{"spec":{"hostNetwork":true,"dnsPolicy":"ClusterFirstWithHostNet"}}}}'
# Then JSON patch each MGR to remove --public-addr=$(ROOK_POD_IP) from container args

# MDS
for mds in a b c d; do
  oc patch deploy -n rook-ceph rook-ceph-mds-kvcache-fs-$mds --type strategic \
    -p '{"spec":{"template":{"spec":{"hostNetwork":true,"dnsPolicy":"ClusterFirstWithHostNet"}}}}'
done
```

### Phase 4: Operator reconciliation

1. Apply the updated CephCluster CR:
   ```bash
   oc apply -f 02-ceph-cluster.yaml
   ```

2. Scale operator back up:
   ```bash
   oc scale deploy -n rook-ceph rook-ceph-operator --replicas=1
   ```

3. Operator detects single MON, creates 2 new MONs with hostNetwork on management IPs (see "Why MONs use management IPs" below).

4. Wait for 3-MON quorum. If operator deadlocks on version upgrade (see gotcha #4), manually create the 3rd MON deployment from a template of the 2nd.

### Phase 5: Verification

```bash
# Cluster health
ceph -m 10.0.0.6 status  # → HEALTH_OK

# MON addresses
ceph mon dump  # → e on 10.0.0.6, k/l on 10.243.65.x

# OSD addresses — all should be 10.0.0.x
ceph osd dump | grep "^osd\." | grep -oP 'v2:\K[0-9.]+' | sort -u
# → 10.0.0.4, 10.0.0.6, 10.0.0.7

# MGR address
ceph mgr dump | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['active_addrs'])"
# → 10.0.0.8 (200G)

# MDS addresses
ceph fs dump | grep mds\. | grep -oP 'v2:\K[0-9.]+'
# → all 10.0.0.8

# Traffic verification: write 2GB, check NIC byte counters
MGMT_TX_BEFORE=$(cat /sys/class/net/enp3s0/statistics/tx_bytes)
NIC200_TX_BEFORE=$(cat /sys/class/net/enp163s0/statistics/tx_bytes)
dd if=/dev/zero of=/mnt/cephfs/testfile bs=1M count=2048 conv=fdatasync
MGMT_TX_AFTER=$(cat /sys/class/net/enp3s0/statistics/tx_bytes)
NIC200_TX_AFTER=$(cat /sys/class/net/enp163s0/statistics/tx_bytes)
echo "Management: $(( (MGMT_TX_AFTER - MGMT_TX_BEFORE) / 1048576 )) MB"   # → 0 MB
echo "200G:       $(( (NIC200_TX_AFTER - NIC200_TX_BEFORE) / 1048576 )) MB" # → 2063 MB
```

## Measured performance

| Metric | Value |
|--------|-------|
| Sequential write (dd, 4 GB) | 1.3 GB/s (10.4 Gbps) |
| Sequential read (dd, 4 GB, cache dropped) | 1.7 GB/s (13.6 Gbps) |
| Management NIC traffic during I/O | 0 MB |
| 200G NIC traffic during I/O | 2063 MB (for 2 GB write) |

These numbers are for data pool `replicated.size: 1` (single OSD, no replication). With replication, write throughput would be split across replica targets.

## Why MONs use management IPs (and why that's fine)

The Rook operator determines MON addresses from the Kubernetes Node object's `status.addresses[].InternalIP`, which is always the management NIC IP (10.243.65.x). The 200G IPs are configured at the OS level via NetworkManager but are not reflected in the Kubernetes Node status — the cloud provider only registers the management IP.

MON-e was manually patched to 10.0.0.6 via monmap repair. Operator-created MONs (k, l) use management IPs (10.243.65.15, 10.243.65.9). This is acceptable because:

- MON traffic is only heartbeats (~100 bytes/sec) and OSD map updates (a few KB when topology changes)
- No data flows through MONs — all I/O goes client → OSD directly
- With hostNetwork, each host has both management and 200G interfaces in the same network namespace, so cross-network connectivity works

## Gotchas and hidden dependencies

### 1. `--public-addr=$(ROOK_POD_IP)` in MON and MGR args

The Rook operator injects `--public-addr=$(ROOK_POD_IP)` into MON and MGR deployment container args. With `hostNetwork`, `ROOK_POD_IP` resolves to the node's primary IP (management NIC), which **overrides** `public_network` from the Ceph config database. The daemon binds to the management IP instead of the 200G IP.

**Symptoms**: MON logs show `public addrs [v2:10.0.0.6:3300] at bind addrs [v2:10.243.65.5:3300]`. MGR shows active address on 10.243.65.x instead of 10.0.0.x.

**Fix for MON**: Replace `--public-addr=$(ROOK_POD_IP)` with `--public-addr=10.0.0.6` (the specific 200G IP for that node). Also replace `--public-bind-addr=$(ROOK_POD_IP)` if present.

**Fix for MGR**: Remove `--public-addr=$(ROOK_POD_IP)` entirely — the MGR doesn't need it and will correctly use `public_network` to pick the 200G IP. Don't hardcode an IP because MGRs can be rescheduled to different nodes.

**Note**: OSDs do NOT have `--public-addr` in their args — they rely solely on `public_network` from the config database and correctly bind to 200G IPs automatically.

### 2. `rook-ceph-config` Secret

Contains `mon_host` and `mon_initial_members` used by **ALL daemons** (OSDs, MGR, MDS) to find MONs at startup. The init container `cephx-keyring-update` reads this secret and tries to connect to the listed MON addresses.

**Symptom**: After MON address change, OSD pods sit at 0/1 Ready for 5 minutes, then crash. Init container logs show repeated connection attempts to old ClusterIP addresses.

**Fix**: Update the secret immediately after changing MON addresses:
```bash
MON_HOST=$(echo -n '[v2:10.0.0.6:3300,v1:10.0.0.6:6789]' | base64)
MON_MEMBERS=$(echo -n 'e' | base64)
oc patch secret -n rook-ceph rook-ceph-config --type merge \
  -p "{\"data\":{\"mon_host\":\"$MON_HOST\",\"mon_initial_members\":\"$MON_MEMBERS\"}}"
```
Then restart all daemon pods.

### 3. `rook-ceph-mon-endpoints` ConfigMap

Contains three critical fields:
- `data`: MON endpoint list (e.g., `e=10.0.0.6:6789`)
- `csi-cluster-config-json`: CSI driver monitor list for PVC provisioning
- `mapping`: Node-to-MON placement mapping with addresses

All three must be consistent. If the CSI config still points to old MON addresses, PVC creation fails silently.

### 4. Operator version upgrade deadlock

If the Ceph image tag (`quay.io/ceph/ceph:v19`) resolves to a newer version than running daemons, the operator enters a version reconciliation loop:

1. Detects mixed versions (e.g., 23 daemons on 19.2.4, 6 on 19.2.5)
2. Tries to rolling-upgrade MONs first (operator processes MONs before creating new ones)
3. Can't stop MON-e for upgrade because <3 MONs exist → quorum would be lost
4. Never creates the 3rd MON → deadlock

**Fix**: Pin the Ceph image to the exact running version in the CephCluster CR:
```yaml
cephVersion:
  image: quay.io/ceph/ceph:v19.2.4  # NOT v19
```

### 5. `--setuser-match-path` in MON deployment args

When creating a MON deployment from a template (e.g., copying MON-k's deployment to create MON-l), the `--setuser-match-path=/var/lib/ceph/mon/ceph-k/store.db` arg contains a hardcoded path with the source MON's ID. The new MON's data directory is `ceph-l`, not `ceph-k`.

**Symptom**: MON exits immediately with `unable to stat setuser_match_path /var/lib/ceph/mon/ceph-k/store.db: No such file or directory`.

**Fix**: Replace `ceph-k` with `ceph-l` in the `--setuser-match-path` arg, all `volumeMount.mountPath` entries, and the `chown` init container args.

### 6. VPC security groups (belt and suspenders)

Although PCI passthrough traffic bypasses VPC security groups, we opened Ceph ports (3300, 6789, 6800-7300) on the `diadochos-hqxzk-sg-cluster-wide` security group scoped to same-SG sources. This provides a fallback path if any management-NIC MON traffic needs to traverse the VPC firewall.

```bash
for port in 3300 6789; do
  ibmcloud is sg-rulec $SG inbound tcp --port-min $port --port-max $port --remote $SG
done
ibmcloud is sg-rulec $SG inbound tcp --port-min 6800 --port-max 7300 --remote $SG
```

## Monmap repair procedure (used for cluster recovery)

When MON quorum is permanently broken (e.g., MONs on mixed pod-network / host-network IPs that can't reach each other):

1. Scale down operator and all MONs
2. Create a privileged pod on the surviving MON's node, mounting its data directory
3. Extract monmap: `ceph-mon --extract-monmap /tmp/monmap --mon-data /var/lib/rook/mon-e/data`
4. Remove broken entries: `monmaptool /tmp/monmap --rm <broken-mon>`
5. Add corrected entry: `monmaptool /tmp/monmap --addv e [v2:10.0.0.6:3300/0,v1:10.0.0.6:6789/0]`
6. Inject: `ceph-mon --inject-monmap /tmp/monmap --mon-data /var/lib/rook/mon-e/data`
7. Update configmap, secret, patch deployment, scale back up
8. Remove stale `public_network` from Ceph config if reverting: `ceph config rm global public_network`

This procedure was used twice during this work — once to recover from the failed approach #2, and once for the successful migration.

## Rollback procedure

If the 200G migration needs to be reverted:

1. Scale operator to 0
2. `ceph config rm global public_network`
3. Monmap repair: change MON address back to a ClusterIP
4. Update configmap and secret with ClusterIP addresses
5. Remove `hostNetwork` from all deployments
6. Scale everything back up

This was successfully tested during the earlier failed attempt recovery.

## Final cluster state (2026-07-24)

```
HEALTH_OK
  mon: 3 daemons, quorum e,k,l
  mgr: a(active), standbys: b
  mds: 2/2 daemons up, 2 hot standby
  osd: 21 osds: 21 up, 21 in
  objects: 111.46k objects, 432 GiB
  pgs: 3 active+clean
  All daemons: ceph version 19.2.4 squid (stable)
```

| Daemon | Address | Network |
|--------|---------|---------|
| MON-e | 10.0.0.6 | 200G |
| MON-k | 10.243.65.15 | Management |
| MON-l | 10.243.65.9 | Management |
| MGR-a (active) | 10.0.0.8 | 200G |
| MGR-b (standby) | 10.0.0.4 | 200G |
| MDS (all 4) | 10.0.0.8 | 200G |
| OSDs (all 21) | 10.0.0.4/6/7 | 200G |

## Related

- [[CephFS performance tuning for KV cache offloading]]
- [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]
- [[Ceph orphaned OSDs after node disruption]]
- Research: [[KV Cache Offloading]]
- Runbook: `clusters/psap-diadochos-h100/rook-ceph/RUNBOOK.md`