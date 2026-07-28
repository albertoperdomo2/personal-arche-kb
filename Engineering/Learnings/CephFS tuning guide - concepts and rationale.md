---
title: "CephFS Tuning Guide: Concepts and Rationale"
date: 2026-07-28
type: learning
cluster: diadochos
topic: CephFS performance tuning
---

# CephFS Tuning Guide: Concepts and Rationale

This document explains every tuning change applied to the diadochos CephFS cluster from first principles. It is written for someone who knows nothing about CephFS — every concept (OSD, MDS, PG, MTU, mclock, etc.) is introduced before it is used.

## Table of Contents

1. [What is Ceph and CephFS?](#what-is-ceph-and-cephfs)
2. [The Ceph Architecture: Daemons and Components](#the-ceph-architecture)
3. [How Data Flows Through CephFS](#how-data-flows-through-cephfs)
4. [The Hardware: NICs, NVMe, and the Network Topology](#the-hardware)
5. [The Tuning Journey: From 2 GB/s to 23.5 GB/s](#the-tuning-journey)
6. [Change 1: Increasing pg_num (1 → 128)](#change-1-pg_num)
7. [Change 2: Mellanox NIC Fix — Moving Off VirtIO](#change-2-mellanox-nic-fix)
8. [Change 3: OSD Internal Tuning](#change-3-osd-tuning)
9. [Change 4: StorageClass Mount Options (rsize/wsize = 64 MB)](#change-4-storageclass)
10. [Change 5: 6-NIC OSD Distribution](#change-5-6-nic-distribution)
11. [Changes 6–7: I/O Depth (iodepth)](#changes-6-7-iodepth)
12. [Summary of All Ceph Config Settings](#summary-of-settings)
13. [Glossary](#glossary)

---

## What is Ceph and CephFS? {#what-is-ceph-and-cephfs}

**Ceph** is a distributed storage system. Instead of storing data on a single disk or a single NAS server, Ceph spreads data across many disks on many machines. This provides:

- **Scalability**: add more disks or machines to grow capacity and throughput
- **Redundancy**: data can be replicated so it survives disk or node failures
- **Flexibility**: Ceph provides block storage (like a virtual disk), object storage (like S3), and a POSIX filesystem (CephFS)

**CephFS** is Ceph's filesystem layer. It presents a standard POSIX filesystem (like ext4 or NFS) that multiple machines can mount simultaneously with read-write access (called **RWX** in Kubernetes). In our cluster, CephFS is used to share the **KV (key-value) cache** for vLLM inference across GPU nodes — one node writes cached inference data, and other nodes can read it, avoiding redundant computation.

**Rook** is the Kubernetes operator that deploys and manages Ceph inside an OpenShift/Kubernetes cluster. Instead of manually installing Ceph on each machine, Rook runs all Ceph daemons as Kubernetes pods and manages their lifecycle, configuration, and upgrades.

---

## The Ceph Architecture: Daemons and Components {#the-ceph-architecture}

Ceph runs several types of daemon processes, each with a specific role. Think of them as specialized workers in a warehouse:

### MON — Monitors (the management office)

**What they do**: Monitors maintain the **cluster map** — a set of data structures that describe where everything is: which OSDs exist, which ones are up or down, and how data is distributed. Every other daemon and every client asks the monitors "where should I find/put this data?"

**How many**: Always an odd number (typically 3 or 5) so they can use majority voting (**quorum**) to agree on the cluster state. If 1 out of 3 monitors fails, the other 2 form a majority and the cluster continues.

**Performance impact**: Minimal. Monitors handle only metadata and heartbeats — no actual data flows through them. They can safely run on slower networks.

**On diadochos**: 3 monitors (k, l, m), running on the GPU nodes.

### OSD — Object Storage Daemons (the warehouse workers)

**What they do**: OSDs are the workhorses of Ceph. Each OSD manages one physical disk (in our case, one NVMe SSD). When a client writes data, the OSD receives the data and writes it to its NVMe drive. When a client reads, the OSD fetches data from the drive and sends it over the network.

**How many**: One per physical disk. Our cluster has 3 nodes × 7 NVMe SSDs = **21 OSDs**.

**Performance impact**: Very high. The OSDs are the bottleneck for most workloads. Their network speed, disk speed, CPU allocation, memory, and internal threading configuration all directly affect throughput and latency.

**On diadochos**: 21 OSDs (osd.0 through osd.34, with some IDs skipped due to historical removals), each backed by a ~7 TB NVMe drive, 7 per GPU node.

### MDS — Metadata Servers (the filing cabinet)

**What they do**: MDS daemons exist only for CephFS (not for block or object storage). They handle **filesystem metadata** — directory listings, file permissions, file sizes, inode lookups. When you `ls` a directory or `stat` a file, the MDS answers. But the MDS does NOT handle actual file data — that goes directly between the client and the OSDs.

Think of it this way: the MDS knows *where* a file is and *what* it looks like (size, permissions, timestamps), but when you actually *read the bytes* of that file, you talk directly to the OSDs that store those bytes.

**How many**: At least 1 active, but more active MDS daemons allow CephFS to handle more metadata operations in parallel (important for workloads with many small files or frequent directory listings). **Standby** MDS daemons wait in reserve and take over if an active one fails. **Standby-replay** MDS daemons actively follow an active MDS's journal so they can take over even faster.

**On diadochos**: 2 active MDS + 2 hot standby-replay (`max_mds = 2`).

### MGR — Managers (the accountant)

**What they do**: Managers collect cluster statistics, expose the Ceph dashboard, handle alerts, and run management modules (like the Prometheus metrics exporter). They don't affect data-path performance.

**On diadochos**: 2 managers (a, b), one active and one standby.

### Summary diagram

```
  Client (GPU node running vLLM)
     │
     ├── metadata ops (ls, stat, open) ──→ MDS
     │                                       │
     │                                       ↓
     └── data ops (read, write) ──────────→ OSD ──→ NVMe SSD
                                             ↑
                                             │
                                  MON (tells everyone where data lives)
```

---

## How Data Flows Through CephFS {#how-data-flows-through-cephfs}

Understanding the data flow is essential to understanding why each tuning change matters.

### Writing a file

1. The **client** (the vLLM process on a GPU node) calls `write()` on an open CephFS file.
2. The **kernel CephFS driver** (running in the Linux kernel on that GPU node) breaks the write into **objects** (typically 4 MB each).
3. For each object, the kernel uses the **CRUSH algorithm** (Controlled Replication Under Scalable Hashing) to determine which OSD should store it. CRUSH is a deterministic hash — given the object name and the cluster map, any client can independently calculate which OSD holds any object, without asking anyone.
4. The kernel sends the object data over the network directly to the target OSD.
5. The OSD writes the data to its NVMe SSD and acknowledges back.
6. If the pool has **replication** (e.g., `size: 3`), the primary OSD also forwards the data to 2 other OSDs before acknowledging. In our cluster, we use `size: 1` (no replication) because the KV cache data is ephemeral — if it's lost, vLLM simply recomputes it.

### Reading a file

1. The client calls `read()`.
2. The kernel CephFS driver determines which OSD has each object (via CRUSH).
3. The kernel reads directly from the OSD over the network.
4. The OSD reads from its NVMe and sends the data back.

### Key insight: data never goes through MON or MDS

Monitors and MDS daemons are **not** in the data path. All actual file bytes travel directly between the client kernel and the OSDs. This is why OSD network speed and OSD internal performance dominate throughput.

---

## The Hardware: NICs, NVMe, and the Network Topology {#the-hardware}

### What is a NIC?

A **NIC** (Network Interface Card) is the physical network adapter in a server. It determines how fast data can travel to/from that machine. Our GPU nodes have two types:

| Type | Interface | Speed | MTU | Purpose |
|------|-----------|-------|-----|---------|
| VirtIO (management) | `enp3s0` via `br-ex` | ~1.5 Gbps effective | 1500 | Kubernetes pod networking, SSH, API server |
| Mellanox ConnectX-7 | `enp163s0`–`enp213s0` | 200 Gbps each | 9000 | High-speed storage and GPU-to-GPU communication |

Each GPU node has **6 usable Mellanox NICs** for Ceph data, each on a different subnet (10.0.x, 10.1.x, … 10.5.x). These are **PCI passthrough virtual functions** from the IBM Cloud hypervisor, using the `mlx5_core` Linux driver.

### What is MTU?

**MTU** (Maximum Transmission Unit) is the largest packet size (in bytes) that a network interface can send in a single frame.

- **MTU 1500** (default): standard Ethernet. Each packet carries up to ~1,460 bytes of payload.
- **MTU 9000** ("jumbo frames"): each packet carries up to ~8,960 bytes of payload — about 6× more data per packet.

Why does this matter? Every packet has a fixed overhead: headers, checksums, interrupts. With MTU 1500, to send 1 MB of data, you need ~700 packets. With MTU 9000, you need ~120 packets. Fewer packets means less CPU overhead per byte transferred, which translates to higher throughput and lower latency, especially at high data rates.

All 6 Mellanox NICs used for Ceph have **MTU 9000** (jumbo frames enabled).

### What is NVMe?

**NVMe** (Non-Volatile Memory Express) is a protocol for connecting solid-state drives (SSDs) directly to the CPU via the PCIe bus, bypassing the older SATA/SAS controller bottleneck. NVMe SSDs can deliver:

- Random reads: 500K–1M+ IOPS (I/O operations per second)
- Sequential reads/writes: 3–7 GB/s per drive

Each GPU node has 7 × ~7 TB NVMe SSDs, each managed by one OSD daemon.

---

## The Tuning Journey: From 2 GB/s to 23.5 GB/s {#the-tuning-journey}

The baseline performance of the cluster was **~2 GB/s** — far below what the hardware should deliver. Over 7 incremental changes, throughput was pushed to **23.5 GB/s read / 22.6 GB/s write**. That's a **~12× improvement**.

Two changes accounted for ~80% of the gains:
1. **Mellanox NIC fix** (~4–5× improvement): moved data traffic from the slow VirtIO management NIC to the 200 Gbps Mellanox NICs
2. **pg_num increase** (prerequisite): without enough Placement Groups, only a fraction of the OSDs were being utilized

The remaining changes were incremental but additive — each one unlocked another bottleneck.

---

## Change 1: Increasing pg_num (1 → 128) {#change-1-pg_num}

### What is a Placement Group (PG)?

A **Placement Group** is a logical shard of a Ceph storage pool. It is the fundamental unit of data distribution in Ceph.

When Ceph stores objects, it doesn't assign each object directly to an OSD. Instead, it uses a two-step process:

1. **Object → PG**: Each object is hashed (by its name) into a Placement Group. The hash function evenly distributes objects across all PGs.
2. **PG → OSD(s)**: Each PG is assigned to one or more OSDs (depending on the pool's replication factor) using the CRUSH algorithm.

Think of PGs as buckets in a hash table. Objects go into buckets, and buckets are assigned to warehouse workers (OSDs).

### Why pg_num = 1 was a disaster

With only 1 PG, **every single object in the pool mapped to the same PG**, and that PG was assigned to a single primary OSD. This meant:

- Only **1 out of 21 OSDs** was handling all I/O
- Only **1 NVMe drive** out of 21 was being read/written
- Only **1 network connection** was carrying all data traffic

It was like having a 21-lane highway but forcing all traffic through a single toll booth.

### Why 128?

The recommended formula for pg_num is:

$$\text{pg\_num} = \frac{\text{OSDs} \times 100}{\text{replication size}}$$

With 21 OSDs and replication size 1: $21 \times 100 / 1 = 2100$, rounded to the next power of 2 = 2048. However, for a workload with relatively few large files (KV cache blocks are typically megabytes), 128 PGs provides sufficient distribution across all 21 OSDs while keeping overhead low. The key was getting away from 1.

### What is pgp_num?

`pgp_num` (Placement Group for Placement purposes) controls how PGs are actually distributed to OSDs. It should always equal `pg_num`. When you increase `pg_num`, Ceph creates new PGs; when you increase `pgp_num` to match, Ceph redistributes objects to fill the new PGs.

### Impact

**~2 GB/s → ~2.3 GB/s**: The gain was small because at this point all traffic was still going through the slow VirtIO management NIC (~1.5 Gbps). Spreading I/O across 21 OSDs doesn't help much when the network is the bottleneck. But this change was a **prerequisite** — without it, the Mellanox NIC fix (change #2) would have no effect because all traffic would still go to a single OSD.

### How to verify

```bash
ceph osd pool get kvcache-fs-data0 pg_num   # → 128
ceph osd pool get kvcache-fs-data0 pgp_num  # → 128
```

---

## Change 2: Mellanox NIC Fix — Moving Off VirtIO {#change-2-mellanox-nic-fix}

### The problem

By default, all Ceph daemon pods in Kubernetes use the **OVN overlay network** — a software-defined virtual network that routes traffic through the node's management NIC (VirtIO, ~1.5 Gbps effective). This means all OSD read/write traffic, MDS metadata traffic, and MON heartbeats shared a ~1.5 Gbps pipe.

Meanwhile, each node had **6 × 200 Gbps Mellanox NICs sitting idle**. That's 1.2 Tbps of available bandwidth per node — 800× more than VirtIO.

### The fix

Two changes were needed:

**a) hostNetwork mode**: Instead of running Ceph daemon pods inside the Kubernetes overlay network, they were switched to `hostNetwork: true`. This makes the pod share the node's network namespace, giving it direct access to all physical NICs — including the Mellanox NICs.

**b) public_network configuration**: Ceph has a setting called `public_network` that tells daemons which IP subnet to bind to. By setting it to `10.0.0.0/16` (the Mellanox NIC subnet), all OSD, MDS, and MGR daemons bind to a Mellanox NIC IP instead of the VirtIO management IP.

Additionally, **per-OSD `public_addr`** was set to force each OSD to bind to a specific Mellanox NIC IP address. This is what enables change #5 (multi-NIC distribution) — without per-OSD addresses, all 7 OSDs on a node would share the first Mellanox NIC.

### Why this was hard

This was not a simple configuration flip. It required a **big-bang monitor migration** with ~15 minutes of CephFS downtime. Three earlier approaches failed:

1. **Multus ipvlan** — failed because the Linux kernel prevents a host from ARP-resolving its own ipvlan children
2. **Host networking without migration** — broke monitor quorum because old monitors (on VirtIO IPs) couldn't communicate with new daemons (on Mellanox IPs)
3. **Host networking on management subnet** — OCP's OVN firewall blocks Ceph ports on non-standard ports between nodes

The successful approach used monmap injection to forcibly change monitor addresses while the cluster was stopped. See [[Routing Ceph over 200G NICs on OCP with PCI passthrough VFs]] for the full procedure.

### Impact

**~2.3 GB/s → ~8–12 GB/s**: Moving from a shared 1.5 Gbps VirtIO pipe to dedicated 200 Gbps Mellanox NICs was the single largest improvement. At this point, the network was no longer the primary bottleneck.

---

## Change 3: OSD Internal Tuning {#change-3-osd-tuning}

With the network bottleneck removed, the next limiting factor was how fast each OSD could process I/O requests internally. Several OSD parameters were tuned:

### osd_op_num_shards_ssd (default 1 → 16)

**What it does**: Inside each OSD daemon, incoming I/O operations are distributed across **shards** — independent processing queues. Each shard processes operations in parallel with the others.

**Analogy**: Imagine a restaurant kitchen with 1 prep station vs. 16 prep stations. With 1, all orders queue up behind each other. With 16, up to 16 orders are being prepared simultaneously.

**Why 16**: NVMe SSDs have massive internal parallelism (hundreds of thousands of IOPS). A single OSD processing thread cannot saturate the NVMe. 16 shards allow the OSD to issue many operations to the NVMe concurrently.

### osd_op_num_threads_per_shard_ssd (default 1 → 4)

**What it does**: Each shard (from above) has one or more **worker threads**. More threads per shard means more CPU cores working on operations within each shard.

**Combined with osd_op_num_shards_ssd = 16**: This gives each OSD 16 × 4 = **64 operation threads**, up from the default of 1 × 1 = 1. That's 64× more CPU parallelism for processing I/O operations.

### osd_mclock_profile = high_client_ops

**What it does**: Ceph uses a scheduling algorithm called **mClock** to prioritize different types of work inside the OSD. The OSD has to do several things:

- **Client I/O**: reads and writes from application pods (what we care about for performance)
- **Recovery**: rebuilding data after an OSD fails and comes back
- **Scrubbing**: background integrity checks that verify data hasn't been corrupted

The `high_client_ops` profile tells mClock to prioritize client I/O over recovery and scrub operations. This means reads and writes get more CPU/disk time, and background maintenance gets less.

**Trade-off**: After a failure, data recovery will be slower. Since our pool uses `size: 1` (no replication), recovery is less relevant — there's nothing to recover from.

### osd_mclock_max_capacity_iops_ssd

**What it does**: Tells the mClock scheduler how fast the underlying NVMe drive is, so it can make accurate scheduling decisions. Each OSD had this value **individually benchmarked** — the values range from ~44K to ~79K IOPS depending on the drive.

**Why per-OSD**: Not all NVMe SSDs are identical even within the same server. Manufacturing variation, wear level, and firmware differences mean each drive has a slightly different peak IOPS. By measuring each one, mClock can make optimal scheduling decisions.

### objecter_inflight_ops (default 1024 → 4096)

**What it does**: This is a **client-side** setting (not OSD-side). It controls how many I/O operations a CephFS client (the kernel driver on the GPU node) is allowed to have in flight simultaneously — sent to OSDs but not yet acknowledged.

**Analogy**: Imagine ordering packages from 21 different warehouses. If you're limited to ordering 1024 packages total across all warehouses, some warehouses sit idle waiting for their turn. With 4096, more warehouses stay busy in parallel.

**Why 4096**: With 21 OSDs and high-bandwidth NICs, the client can easily fill many more than 1024 concurrent operations. 4096 allows the client to keep all OSDs busy.

### objecter_inflight_op_bytes (500 MB)

Related to the above — limits the total bytes in flight, not just the count. 500 MB allows roughly 125 concurrent 4 MB object writes, which is sufficient to keep the pipeline full.

### ms_tcp_rcvbuf (default OS value → 4 MB)

**What it does**: Sets the TCP receive buffer size for all Ceph messenger (ms) connections. The receive buffer is how much data the kernel can buffer in memory before the application reads it.

**Why 4 MB**: With 200 Gbps NICs, data arrives very fast. A small TCP buffer causes the receiver to signal the sender to slow down (TCP flow control), creating pauses. A 4 MB buffer gives enough headroom for the OSD to absorb bursts without throttling.

**Analogy**: A bigger mailbox. If your mailbox is too small, the mail carrier has to wait for you to empty it before delivering more.

### Impact

**~8–12 GB/s → moderate additional gain under concurrency**: These changes don't show dramatic improvement in simple sequential benchmarks, but they significantly increase throughput when multiple clients or threads access the filesystem simultaneously.

### How to verify

```bash
ceph config get osd osd_op_num_shards_ssd         # → 16
ceph config get osd osd_op_num_threads_per_shard_ssd  # → 4
ceph config get osd osd_mclock_profile             # → high_client_ops
ceph config get client objecter_inflight_ops       # → 4096
ceph config get global ms_tcp_rcvbuf               # → 4194304
```

---

## Change 4: StorageClass Mount Options (rsize/wsize = 64 MB) {#change-4-storageclass}

### What is a StorageClass?

In Kubernetes, a **StorageClass** defines how storage volumes are provisioned. It specifies which storage system to use (CephFS, in our case), how to configure it, and what mount options to apply when a pod mounts the volume.

### What are rsize and wsize?

When a CephFS client (the Linux kernel driver) reads or writes data, it doesn't send each byte individually. It batches data into chunks:

- **rsize** (read size): the maximum number of bytes the client requests in a single read RPC to the OSD
- **wsize** (write size): the maximum number of bytes the client sends in a single write RPC to the OSD

The default in the Linux kernel CephFS client is **64 KB** (65,536 bytes).

### Why 64 MB?

$$\text{throughput} = \frac{\text{chunk size}}{\text{latency per round trip}}$$

With 64 KB chunks and even a tiny 0.1 ms round-trip time:

$$\frac{64\text{ KB}}{0.1\text{ ms}} = 640\text{ MB/s}$$

That's a hard ceiling regardless of how fast the NIC or disk is. With 64 MB chunks:

$$\frac{64\text{ MB}}{0.1\text{ ms}} = 640\text{ GB/s}$$

The ceiling is now far above the hardware limits. Each round trip carries 1000× more data, so the same number of round trips transfers 1000× more data.

**Analogy**: Imagine carrying boxes from a warehouse to a truck. With a small box (64 KB), you make 1000 trips. With a huge crate (64 MB), you make 1 trip. The walking time (latency) is the same either way, but you move far more cargo per trip with bigger crates.

### What is noatime?

Every time a file is read, Linux normally updates the file's **access time** (atime) metadata. This generates a **write** for every **read** — the MDS must update the inode's atime field, and that update must be journaled and flushed.

`noatime` disables this. Since we don't care when KV cache files were last accessed, this eliminates unnecessary metadata writes and reduces MDS load.

`nodiratime` does the same for directory access times.

### StorageClasses on the cluster

Two StorageClasses exist, both now with 64 MB rsize/wsize:

| StorageClass | Mount Options |
|---|---|
| `rook-cephfs` | `noatime`, `nodiratime`, `wsize=67108864`, `rsize=67108864` |
| `rook-cephfs-fast` | `noatime`, `nodiratime`, `wsize=67108864`, `rsize=67108864` |

The `rook-cephfs-fast` StorageClass was created specifically for performance-sensitive workloads. Both now have identical mount options (the original `rook-cephfs` was updated to match).

### Impact

**~5–9% write improvement**: Modest but free — this is purely a client-side optimization with no downsides for sequential I/O workloads.

---

## Change 5: 6-NIC OSD Distribution {#change-5-6-nic-distribution}

### The problem

After change #2 (moving to Mellanox NICs), all 7 OSDs on each node shared a **single Mellanox NIC** (enp163s0, on subnet 10.0.0.0/16). Even though each NIC is 200 Gbps, 7 OSDs sharing one NIC means each OSD effectively gets ~28 Gbps — and the NIC becomes the bottleneck before the NVMe drives are saturated.

Meanwhile, each node has **6 Mellanox NICs** available, each on a different subnet.

### The fix

Each OSD was given its own **`public_addr`** — a per-OSD configuration that forces it to bind to a specific IP address on a specific NIC. The 7 OSDs on each node are distributed across the 6 NICs:

| NIC | Subnet | OSDs per node | Bandwidth |
|-----|--------|---------------|-----------|
| enp163s0 | 10.0.x.x | 2 | 200 Gbps |
| enp173s0 | 10.1.x.x | 1 | 200 Gbps |
| enp183s0 | 10.2.x.x | 1 | 200 Gbps |
| enp193s0 | 10.3.x.x | 1 | 200 Gbps |
| enp203s0 | 10.4.x.x | 1 | 200 Gbps |
| enp213s0 | 10.5.x.x | 1 | 200 Gbps |
| **Total** | | **7** | **1.2 Tbps** |

The first NIC gets 2 OSDs (because 7 doesn't divide evenly by 6), and all others get 1.

### Why this matters for CephFS specifically

The CephFS kernel client **pipelines** I/O across connections. When data is spread across 21 OSDs on 18 different NIC endpoints (6 per node × 3 nodes), the client can read/write to all of them simultaneously. More NIC endpoints = more parallel network streams = higher aggregate throughput.

### How the public_network supports this

The CephCluster CR's `public_network` was expanded to include all 6 subnets:

```yaml
addressRanges:
  public:
    - "10.243.65.0/24"   # management (for monitors)
    - "10.0.0.0/16"      # Mellanox NIC 1
    - "10.1.0.0/16"      # Mellanox NIC 2
    - "10.2.0.0/16"      # Mellanox NIC 3
    - "10.3.0.0/16"      # Mellanox NIC 4
    - "10.4.0.0/16"      # Mellanox NIC 5
    - "10.5.0.0/16"      # Mellanox NIC 6
```

Without listing all subnets, the OSD daemons wouldn't accept connections on those networks.

### Impact

**Reads: +79% (from ~7.3 GB/s to 13 GB/s). Writes: 12.7 GB/s (at iodepth=1)**. This was the second most impactful change after the initial Mellanox migration.

---

## Changes 6–7: I/O Depth (iodepth) {#changes-6-7-iodepth}

### What is I/O depth?

**I/O depth** (or queue depth) is a **client-side benchmark parameter**, not a Ceph cluster setting. It controls how many I/O operations the benchmark tool (fio, or benchflow in our case) submits simultaneously without waiting for previous ones to complete.

- **iodepth=1**: Submit 1 operation, wait for it to complete, submit the next. This measures **latency per operation** but leaves the storage system mostly idle between operations.
- **iodepth=4**: Submit 4 operations concurrently. The storage system has 4 operations to work on simultaneously, improving throughput.
- **iodepth=8**: Submit 8 operations concurrently. Even more parallelism.

### Why it matters

A single I/O operation takes time to:
1. Travel from client to OSD over the network (network latency)
2. Be processed by the OSD (OSD processing time)
3. Be read from / written to the NVMe drive (disk latency)
4. Travel back to the client (network latency again)

With iodepth=1, the client waits for this entire round trip before sending the next operation. The NVMe drive, the network, and the OSD are all idle during the transit time. Higher iodepth fills these gaps — while operation #1 is traveling back from the OSD, operations #2, #3, and #4 are already being processed.

### The progression

| iodepth | Read | Write | Notes |
|---------|------|-------|-------|
| 1 | 13 GB/s | 12.7 GB/s | After multi-NIC fix |
| 4 | 21 GB/s | 20.4 GB/s | +62% read, +61% write |
| 8 | 23.5 GB/s | 22.6 GB/s | +12% more, but P99 latency doubles |

**iodepth=8 saturates** — going higher shows diminishing throughput gains but increasing tail latency (P99 doubles), meaning the most unlucky operations take much longer. This is the point where the system's internal queues are full and adding more operations just increases wait time.

### Where iodepth is configured

iodepth is set in the benchmark workload definition (fio job file or benchflow configuration), not in the Ceph cluster. It cannot be verified from cluster config — it's a property of how the test or application submits I/O.

---

## Summary of All Ceph Config Settings {#summary-of-settings}

### Global settings

| Setting | Value | Purpose |
|---------|-------|---------|
| `public_network` | `10.243.65.0/24, 10.0.0.0/16 … 10.5.0.0/16` | Defines which subnets Ceph daemons bind to |
| `ms_tcp_rcvbuf` | 4194304 (4 MB) | TCP receive buffer — prevents flow-control stalls at high speed |

### OSD settings

| Setting | Value | Default | Purpose |
|---------|-------|---------|---------|
| `osd_op_num_shards_ssd` | 16 | 1 | Parallel processing queues per OSD |
| `osd_op_num_threads_per_shard_ssd` | 4 | 1 | Worker threads per queue |
| `osd_mclock_profile` | `high_client_ops` | `balanced` | Prioritize client I/O over background tasks |
| `osd_mclock_max_capacity_iops_ssd` | 44K–79K (per OSD) | auto | Measured NVMe IOPS capacity per drive |
| `osd_memory_target` | 8589934592 (8 GB) | 4 GB | OSD memory budget for caching and operation |
| `bdev_async_discard_threads` | 1 | 4 | Workaround for Squid v19 discard bug |
| `ms_async_op_threads` | 5 | 3 | Network messenger worker threads per OSD |
| Per-OSD `public_addr` | `v2:10.X.0.Y:0/0` | (auto) | Forces each OSD to bind to a specific Mellanox NIC |

### MDS settings

| Setting | Value | Default | Purpose |
|---------|-------|---------|---------|
| `mds_cache_memory_limit` | 4294967296 (4 GB) | 1 GB | Memory available for metadata caching |
| `max_mds` | 2 | 1 | Number of active MDS daemons |

### Client settings

| Setting | Value | Default | Purpose |
|---------|-------|---------|---------|
| `client_readahead_max_bytes` | 33554432 (32 MB) | 8 MB | Speculative read-ahead — prefetch data the client will likely need |
| `client_cache_size` | 65536 | 16384 | Number of inode/dentry entries cached client-side |
| `client_oc_size` | 1073741824 (1 GB) | 200 MB | Object cache size — client-side cache of object data |
| `objecter_inflight_ops` | 4096 | 1024 | Maximum concurrent I/O operations in flight |
| `objecter_inflight_op_bytes` | 524288000 (500 MB) | 100 MB | Maximum bytes in flight |

### Pool settings

| Setting | Value | Purpose |
|---------|-------|---------|
| `kvcache-fs-data0 size` | 1 | No replication (single copy) — KV cache is ephemeral |
| `kvcache-fs-data0 min_size` | 1 | Allow I/O even with only 1 copy |
| `kvcache-fs-data0 pg_num` | 128 | Distribute data across all 21 OSDs |
| `kvcache-fs-data0 pgp_num` | 128 | Must match pg_num |

### StorageClass mount options

| Option | Value | Purpose |
|--------|-------|---------|
| `rsize` | 67108864 (64 MB) | Maximum read chunk size per RPC |
| `wsize` | 67108864 (64 MB) | Maximum write chunk size per RPC |
| `noatime` | (flag) | Disable access time updates on reads |
| `nodiratime` | (flag) | Disable directory access time updates |

---

## Glossary {#glossary}

| Term | Definition |
|------|-----------|
| **CRUSH** | Controlled Replication Under Scalable Hashing. The algorithm Ceph uses to determine which OSD stores each piece of data. It is deterministic — every client computes the same answer independently. |
| **CephFS** | Ceph's POSIX-compliant distributed filesystem. Provides standard file/directory semantics backed by the Ceph object store. |
| **Daemon** | A background process that runs continuously. In Ceph: MON, OSD, MDS, and MGR are all daemons. |
| **hostNetwork** | A Kubernetes pod setting that makes the pod share the host machine's network stack instead of using the virtual overlay network. Required for Ceph to access Mellanox NICs. |
| **IOPS** | I/O Operations Per Second. A measure of how many read/write operations a storage device can process per second. NVMe SSDs achieve hundreds of thousands. |
| **iodepth** | The number of I/O operations submitted concurrently by a client without waiting for previous ones to complete. Higher iodepth = more parallelism = higher throughput (up to a saturation point). |
| **Jumbo frames** | Ethernet frames with MTU larger than the standard 1500 bytes. Typically MTU 9000. Reduces per-byte packet overhead. |
| **KV cache** | Key-Value cache. In the context of vLLM, this stores computed attention key/value tensors so they can be reused across requests, avoiding redundant GPU computation. |
| **mClock** | A resource scheduling algorithm used inside Ceph OSDs to divide disk/CPU time between client I/O, recovery, and scrubbing. |
| **MDS** | Metadata Server. Handles CephFS directory listings, file attributes, and namespace operations. Does not handle file data. |
| **MGR** | Manager. Handles Ceph monitoring, dashboards, and management modules. Not in the data path. |
| **mlx5_core** | The Linux kernel driver for Mellanox ConnectX-5/6/7 network adapters. |
| **MON** | Monitor. Maintains the cluster map (which OSDs exist, their status, data distribution rules). Uses quorum for fault tolerance. |
| **Monmap** | A data structure inside each monitor that lists all monitors' addresses. Changing a monitor's address requires modifying the monmap. |
| **MTU** | Maximum Transmission Unit. The largest packet a NIC can send in one frame. Larger = fewer packets = less overhead. |
| **NIC** | Network Interface Card. The hardware that connects a server to the network. |
| **NVMe** | Non-Volatile Memory Express. A high-performance storage protocol that connects SSDs directly to the CPU via PCIe. |
| **OSD** | Object Storage Daemon. One per physical disk. Handles actual data reads/writes. The workhorse of Ceph. |
| **PCI passthrough** | A virtualization technique where a physical device (like a NIC) is passed directly to a virtual machine, bypassing the hypervisor for near-native performance. |
| **PG** | Placement Group. A logical shard that maps objects to OSDs. More PGs = better I/O distribution across OSDs. |
| **Pool** | A logical partition of a Ceph cluster with its own replication settings and PG count. In our case, `kvcache-fs-data0` is the data pool for CephFS. |
| **Quorum** | Agreement among a majority of monitors. With 3 monitors, quorum requires 2. This ensures cluster state is consistent even if 1 monitor fails. |
| **Replication** | Storing multiple copies of data on different OSDs for fault tolerance. `size: 3` means 3 copies. We use `size: 1` (no replication) because KV cache data is ephemeral. |
| **Rook** | A Kubernetes operator that deploys and manages Ceph. It creates all the Ceph daemon pods, handles upgrades, and reconciles the desired state from CephCluster custom resources. |
| **RWX** | ReadWriteMany. A Kubernetes PersistentVolume access mode that allows multiple pods on multiple nodes to mount the volume simultaneously with read-write access. CephFS provides this natively. |
| **Scrubbing** | Ceph's background data integrity check. Each OSD periodically reads its stored data and verifies checksums to detect corruption. |
| **Shard** | An internal OSD processing queue. More shards allow the OSD to process more I/O operations in parallel. |
| **StorageClass** | A Kubernetes resource that defines how PersistentVolumes are provisioned and configured, including mount options. |
| **VirtIO** | A standardized interface for virtual devices in cloud environments. VirtIO NICs are slower (~1.5 Gbps effective) compared to physical NICs because traffic goes through the hypervisor's virtual switch. |

---

## Related

- [[CephFS performance tuning for KV cache offloading]] — original tuning notes
- [[Routing Ceph over 200G NICs on OCP with PCI passthrough VFs]] — detailed NIC migration procedure
- [[2026-07-27 - CephFS mount DeadlineExceeded on worker node due to stale monitor IP]] — incident caused by the public_network configuration