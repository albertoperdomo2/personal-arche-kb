---
title: KV Cache Offloading — Per-Model Methodology Template
date: 2026-07-25
type: research-methodology
topic: KV Cache Offloading
status: template
---

# Per-Model Methodology: `<MODEL_NAME>` KV Cache Offloading

> **Instructions**: Copy this template to `<Model-Name>/00 - Methodology.md` and fill in all sections. One per model under test.

---

## 1. Model Identity

| Field | Value |
|-------|-------|
| **Model** | `<e.g., Nemotron-3-Super-120B>` |
| **HuggingFace ID** | `<e.g., nvidia/Nemotron-3-Super-120B-FP8>` |
| **Precision** | `<FP8 / BF16 / FP16>` |
| **Parameter Count** | `<e.g., 120B>` |
| **Architecture** | `<dense / MoE / MLA>` |
| **Context Window (native)** | `<e.g., 131072>` |
| **Model Class** | `<100B+ dense/MoE \| 30–70B \| 7–15B>` |

---

## 2. Fixed Infrastructure (Per Model Class)

| Parameter | Value | Source |
|-----------|-------|--------|
| **Tensor Parallel (TP)** | `<4 / 2 / 1>` | Model class default |
| **Pipeline Parallel (PP)** | `1` | Fixed |
| **Replicas** | `1` | Fixed |
| **GPUs** | `<4×H100 / 2×H100 / 1×H100>` | TP × Replicas |
| **GPU Memory** | `<80 GiB × N>` | Per GPU |
| **vLLM Version** | `<e.g., v0.24.0>` | Pin exact |
| **Image Digest** | `<sha256:...>` | Immutable |
| **RHOAI / Platform** | `<e.g., RHOAI 3.5 EA2>` | Cluster |

---

## 3. Calibration Record

| Field | Value |
|-------|-------|
| **Calibration Date** | `<YYYY-MM-DD>` |
| **Calibrated `gpu-memory-utilization`** | `<e.g., 0.75>` |
| **Calibrated Concurrency (C)** | `<e.g., 128>` |
| **Calibrated Context Length** | `<e.g., 131072>` |
| **Calibration Run ID (MLflow)** | `<run-uuid>` |
| **Calibration Run URL** | `<mlflow-url>` |
| **Pressure Metrics at Calibration** | |
| &nbsp;&nbsp;Mean KV Occupancy | `<XX%>` |
| &nbsp;&nbsp;Peak KV Occupancy | `<XX%>` |
| &nbsp;&nbsp;External KV Share | `<XX%>` |
| &nbsp;&nbsp;Mean Waiting Requests | `<XX>` |
| &nbsp;&nbsp;Peak Waiting Requests | `<XX>` |
| &nbsp;&nbsp;Completed Sessions (1800s) | `<XX>` |
| **Calibration Notes** | `<any deviations, retries, observations>` |
| **Reviewed By** | `<name, date>` |

---

## 4. Standardized Matrix Configuration (Locked Post-Calibration)

All four cells use the **exact same** calibrated parameters:

| Parameter | Value |
|-----------|-------|
| `gpu-memory-utilization` | `<calibrated value>` |
| `max-num-seqs` / Concurrency | `<calibrated value>` |
| `max-model-len` | `<calibrated value>` |
| `/dev/shm` | `200 GiB` |
| Seed(s) | `42, 123, 456` (3 reps minimum) |
| Duration | `1800s` profiling window |
| Workload | AgentX AIPerf WEKA (see Workload Definition) |
| Offload Spec | `TieringOffloadingSpec` (CPU64 + optional secondary) |

### Cell Definitions

| Cell | Tier Config | Secondary Storage | Notes |
|------|-------------|-------------------|-------|
| **A: HBM-only** | No offload | None | Baseline |
| **B: CPU 64 GiB** | `TieringOffloadingSpec`, 64 GiB CPU | None | Primary offload |
| **C: CPU 64 GiB + NVMe** | `TieringOffloadingSpec`, 64 GiB CPU | hostPath NVMe `/mnt/nvme-kv-cache` | Local fast tier |
| **D: CPU 64 GiB + CephFS** | `TieringOffloadingSpec`, 64 GiB CPU | PVC `vllm-kv-cache`, tuned threads | Distributed tier |

---

## 5. Repetition & Randomization Plan

| Repetition | Seed | Purpose |
|------------|------|---------|
| 1 | 42 | Primary (matches historical) |
| 2 | 123 | Variance estimate |
| 3 | 456 | Variance estimate |
| 4+ | 789, 999 | If variance >10% on throughput |

**Randomization**: Run order randomized across cells and reps to avoid temporal bias.

---

## 6. Observability Requirements (Per Run)

| Metric Category | Source | Cadence | Required |
|-----------------|--------|---------|----------|
| Request throughput / latency | AIPerf | Per-request | ✅ |
| TTFT / ITL / E2E distributions | AIPerf | Per-request | ✅ |
| Prompt-token source (external/hit/compute) | vLLM Prometheus | 15s | ✅ |
| GPU KV-cache occupancy | vLLM Prometheus | 15s | ✅ |
| Scheduler running/waiting | vLLM Prometheus | 15s | ✅ |
| CPU tier working set (RSS) | cAdvisor / node-exporter | 15s | ✅ |
| NVMe read/write bytes & latency | node-exporter (nvme) | 15s | Cell C only |
| CephFS PVC usage | kubelet volume stats | 15s | Cell D only |
| CephFS pool/MDS/OSD ops & bytes | Ceph exporter | 15s | Cell D only (best effort) |
| Tier read/write latency & hit/miss | vLLM offload counters | 15s | ✅ (if available) |
| Model pod restarts / OOMs / tracebacks | Kubernetes events | Event | ✅ |

**Missing telemetry = documented limitation in report.**

---

## 7. Acceptance Criteria for Matrix Runs

A matrix run is **valid** if:
- [ ] All 4 cells complete 1800s without pod restart / OOM / traceback
- [ ] All 4 cells hit pressure targets (Calibration Protocol Table 1)
- [ ] At least 3 repetitions per cell with different seeds
- [ ] All required observability streams present (Table 6)
- [ ] MLflow runs logged for every cell × repetition

A cell is **conditionally valid** if:
- Pressure targets missed but documented with reason (e.g., "CephFS backpressure limits achievable concurrency")
- Minor telemetry gaps noted

A cell is **rejected** if:
- Pod restart, OOM, or traceback occurs
- Workload client fails to send requests
- Critical telemetry entirely missing

---

## 8. Analysis Deliverables (Per Matrix)

Each model's matrix produces:

1. **Primary Report** — `YYYY-MM-DD — <Model> standardized offload matrix.md`
   - Executive summary with headline table
   - Figures 1–7 (throughput, latency, prompt-source, scheduler, KV occupancy, disposition, tier observability)
   - Deployment audit table
   - Interpretation & hypotheses
   - Run registry with MLflow links
   - Next experiment recommendation

2. **Comparison Appendices** (linked from primary)
   - Prompt tokens by source — comparison
   - Running/waiting requests — comparison
   - KV transfer & storage — comparison

3. **Raw Data Archive** — MLflow run IDs + exported CSVs/Parquet

---

## 9. Version History

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.0 | 2026-07-25 | Alberto Perdomo | Template created |

---

## 10. Sign-Off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Calibration Lead | | | |
| Experiment Owner | | | |
| Reviewer | | | |