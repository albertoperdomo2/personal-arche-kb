---
title: KV Cache Offloading — Calibration Protocol
date: 2026-07-25
type: research-protocol
topic: KV Cache Offloading
status: active
---

# Calibration Protocol: Standardized Offload Pressure Target

## Purpose

Establish a repeatable, model-specific calibration procedure so that every model enters the standardized offload matrix at a **comparable offload pressure level**. This ensures offload mechanisms are genuinely exercised and results are comparable across models.

---

## Pressure Target (Standardized Across All Models)

| Metric | Target | Rationale |
|--------|--------|-----------|
| **Mean GPU KV-cache occupancy** | 70–90% | High enough to force eviction/offload; low enough to avoid OOM |
| **Peak GPU KV-cache occupancy** | ≥95% | Confirms saturation events occur |
| **External KV transfer share** | >50% of prompt tokens | Proves offload path is active, not just cache-resident |
| **Mean waiting requests** | >20 | Scheduler backpressure present |
| **Peak waiting requests** | >50 | Tail pressure visible |
| **Completed sessions** | >30 in 1800s | Sufficient throughput for statistical validity |

**All four matrix cells must hit these targets.** If a cell fails, adjust its pressure parameters independently (see §4).

---

## Calibration Parameters (Per-Model, Tuned Once)

| Parameter | Role | Search Range | Notes |
|-----------|------|--------------|-------|
| `gpu-memory-utilization` (U) | Primary pressure knob | 0.55 → 0.95 step 0.05 | Lower = more headroom for KV; higher = more model weights in HBM |
| Concurrency (C) | Secondary pressure knob | 32, 64, 128, 256 | AgentX `concurrency` parameter |
| Context length | Tertiary knob | 32K, 64K, 131K | Only if U+C insufficient |
| TP / Replicas | **Fixed per model class** | See §5 | Do not tune during calibration |

---

## Calibration Procedure

### Step 1: Select Base Configuration
- Model: `<model-name>`
- TP / Replicas: per §5 table
- Workload: AgentX AIPerf WEKA trace (see Workload Definition)
- Seed: 42
- Duration: 1800s profiling window
- `/dev/shm`: 200 GiB
- Offload: **HBM-only (no offload)** — calibrate pressure on baseline first

### Step 2: Sweep Pressure Grid
Run short (300–600s) profiling sweeps:

| Sweep | U | C | Context | Duration |
|-------|---|---|---------|----------|
| 1 | 0.55 | 32 | 131K | 300s |
| 2 | 0.65 | 64 | 131K | 300s |
| 3 | 0.75 | 128 | 131K | 300s |
| 4 | 0.85 | 256 | 131K | 300s |
| 5+ | interpolate | interpolate | 131K | 300s |

**Stop criteria**: First config where **all pressure targets** (Table 1) are met in HBM-only mode.

### Step 3: Validate at Full Duration
Run the selected config at **full 1800s** with HBM-only. Confirm targets hold over full window. If not, adjust and re-run.

### Step 4: Lock Calibrated Config
Record in Model Methodology doc (§5 table):
- `calibrated_gpu_memory_utilization`
- `calibrated_concurrency`
- `calibrated_context_length`
- `calibration_run_id` (MLflow)
- `calibration_date`
- Observed pressure metrics at calibration

### Step 5: Run Full Matrix at Calibrated Pressure
Execute the 4-cell matrix (HBM-only, CPU64, CPU64+NVMe, CPU64+CephFS) **all at the calibrated U/C/context**. Each cell gets 3+ repetitions (different seeds: 42, 123, 456).

---

## Per-Cell Pressure Adjustment (If Needed)

If a specific offload cell fails pressure targets at the calibrated config:

| Cell | Adjustment Priority |
|------|---------------------|
| HBM-only | Should already pass (calibration baseline) |
| CPU64 | Increase C by 1 step (e.g., 64→128) |
| CPU64+NVMe | Increase C by 1 step |
| CPU64+CephFS | Increase C by 1 step; if still fails, log as "CephFS backpressure limits achievable pressure" |

**Document every adjustment** in the matrix report with rationale.

---

## Model Class Defaults (Fixed During Calibration)

| Model Class | Examples | TP | Replicas | GPUs | Notes |
|-------------|----------|----|----------|------|-------|
| 100B+ dense / MoE | Nemotron-3-Super-120B, Llama-3.1-405B | 4 | 1 | 4×H100 | FP8 |
| 30–70B dense / MoE | Qwen3.6-35B-A3B, Mixtral-8x22B | 2 | 1 | 2×H100 | FP8/BF16 |
| 7–15B dense | Llama-3.1-8B, Qwen2.5-14B | 1 | 1 | 1×H100 | BF16 |

**Do not change TP/replicas during calibration.** They are fixed by model class.

---

## Acceptance Criteria for Calibration

A calibration is **accepted** when:
1. HBM-only run at calibrated config hits all pressure targets (Table 1) over 1800s
2. MLflow run is logged with full metrics
3. Config recorded in Model Methodology doc
4. At least 2 team members review and sign off

A calibration is **rejected** if:
- OOM or CUDA errors occur
- Pressure targets not met after 5 sweep iterations
- Workload fails to send requests (client-side issue)

---

## Version History

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.0 | 2026-07-25 | Alberto Perdomo | Initial protocol |