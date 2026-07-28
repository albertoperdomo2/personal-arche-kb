# 2026-07-28 - Parallel MLflow uploads time out on benchmark results PVC

## Symptom

Parallel matrix children fail in the `upload-to-mlflow` TaskRun after their benchmark and artifact collection steps succeed:

```
error timed out waiting for remote reader pod benchflow-reader-cpu-offloading-<suffix>
```

The target-cluster event reveals the underlying storage conflict:

```
FailedAttachVolume: Multi-Attach error for volume "pvc-bcf7bf46-71b3-4b96-b8d3-b20740cc5293"
Volume is already used by pod(s) ...
```

This was observed for `cpu-offloading-matrix-1d8b7c` in Diadochos on 2026-07-28. Children `m1`, `m2`, and `m4` completed the benchmark and collection stages but failed while copying remote artifacts before upload. The MLflow API was not reached.

## Environment

- Cluster: Diadochos target cluster with the BenchFlow management cluster on aperdomo-lab.
- Original shared claim: `benchflow/benchmark-results`, 20 GiB, `ReadWriteOnce`, storage class `ibmc-vpc-block-10iops-tier`.
- Workload: concurrent RHOAI `cpu-offloading` matrix children.
- Execution image: `ghcr.io/albertoperdomo2/benchflow:manual-9215117`.

## Timeline

- Benchmark and collection TaskRuns completed successfully for the affected children.
- Their management-cluster `upload-to-mlflow` TaskRuns created temporary `benchflow-reader-*` pods in Diadochos.
- The readers attempted to mount the same RWO claim while benchmark or another reader pod held it on a different node.
- Kubelet reported `FailedAttachVolume Multi-Attach`; the management task deleted the reader after its wait timeout and failed before MLflow upload.

## Root Cause

BenchFlow's remote artifact handoff uses a single shared `benchmark-results` PVC. That is safe only while one target-cluster execution accesses it at a time. On Diadochos the claim was IBM VPC block storage with `ReadWriteOnce`, so it could not be mounted concurrently across nodes. Parallel matrix children made the shared-results design race during benchmark, collection, and reader-pod artifact retrieval.

## Resolution

On 2026-07-28 the old data was intentionally discarded and `benchflow/benchmark-results` was replaced with:

- Storage class: `rook-cephfs`
- Access mode: `ReadWriteMany`
- Capacity: `100Gi`

The volume root was initialized as `container_file_t:s0:c14,c27` with mode `1777`. A normal `benchflow-runner` pod successfully created and removed a test directory, confirming that arbitrary OpenShift UIDs can use it concurrently. The temporary SCC and service account used solely to initialize permissions were deleted afterwards.

Future target-cluster bootstrap reconciliations must include the same immutable claim settings:

```bash
bflow bootstrap \
  --target-kubeconfig /path/to/diadochos/kubeconfig \
  --cluster-name psap-h100-diadochos \
  --results-storage-class rook-cephfs \
  --results-access-mode ReadWriteMany \
  --results-size 100Gi
```

## Prevention / Runbook

Parallel target-cluster execution requires one of these storage contracts:

- Allocate an isolated RWO results PVC per execution and pass that claim through benchmark, collection, and reader stages.
- Use a known healthy RWX results storage class for the shared results workspace.
- Keep target-cluster execution sequential whenever the configured results PVC is RWO.

The durable product fix is per-execution artifact storage, because it preserves parallelism without making correctness dependent on a cluster-wide RWX claim.

## Related

- [[Local NVMe hostPath requires explicit ready-node placement]]
- Matrix run `cpu-offloading-matrix-1d8b7c`