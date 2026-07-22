# NVIDIA GPU Hardware Specifications

> Reference sheet for AI Systems Performance Engineering. All values sourced from official NVIDIA datasheets and verified technical documentation. Last updated: July 2026.

---

## Quick Comparison Table

| GPU | Architecture | VRAM | Mem BW (TB/s) | FP8 Sparse (TFLOPS) | FP4 Sparse (TFLOPS) | TDP | NVLink BW | Form Factor |
|-----|-------------|------|---------------|---------------------|---------------------|-----|-----------|-------------|
| **H100 SXM5** | Hopper | 80 GB HBM3 | 3.35 | 3,958 | — | 700W | 900 GB/s | SXM5 |
| **H100 PCIe** | Hopper | 80 GB HBM2e | 2.0 | 3,026 | — | 350W | 600 GB/s | PCIe |
| **H100 NVL** | Hopper | 94 GB HBM3 | 3.9 | 3,958 | — | 400W | 600 GB/s | NVL Bridge |
| **H200 SXM** | Hopper | 141 GB HBM3e | 4.8 | 3,958 | — | 700W | 900 GB/s | SXM5 |
| **B200** | Blackwell | 192 GB HBM3e | 8.0 | 9,000 (dense) | 18,000 (dense) | 1,000W | 1.8 TB/s | SXM6 |
| **GB200 (2×B200)** | Blackwell | 372 GB HBM3e | 16.0 | 20 PFLOPS | 40 PFLOPS | — | 3.6 TB/s | Superchip |
| **GB200 NVL72** | Blackwell | ~13.5 TB HBM3e | — | 720 PFLOPS | 1,440 PFLOPS | — | 130 TB/s | Rack |
| **B300** | Blackwell Ultra | 288 GB HBM3e | 8.0 | — | 30,000 (dense) | 1,400W | 1.8 TB/s | SXM6 |
| **GB300 NVL72** | Blackwell Ultra | ~21 TB HBM3e | — | — | — | — | — | Rack |

> **Reading the table:** FP8/FP4 values are Tensor Core throughput **with 2:4 structured sparsity** unless marked "dense". Dense values are exactly 1/2 of sparse values. See [Notes: Sparsity vs Dense](#notes).

---

## H100 Variants

### H100 SXM5 — Primary Data Center Variant

| Spec | Value |
|------|-------|
| Architecture | Hopper (GH200) |
| Process | TSMC 4N |
| Transistors | 80 billion |
| Die Size | 814 mm² |
| CUDA Cores | 16,896 |
| Tensor Cores | 528 (4th generation) |
| Streaming Multiprocessors | 132 |
| Base Clock | 1.06 GHz |
| Boost Clock | 1.98 GHz |
| VRAM | 80 GB HBM3 (96 GB on later HBM3 variant) |
| Memory Bandwidth | 3.35 TB/s |
| Memory Bus | 5,120-bit |
| L2 Cache | 50 MB |

#### Compute Performance (H100 SXM5)

| Precision | Dense | With 2:4 Sparsity |
|-----------|-------|-------------------|
| FP64 | 34 TFLOPS | — |
| FP64 Tensor Core | 67 TFLOPS | — |
| FP32 | 67 TFLOPS | — |
| TF32 Tensor Core | 989 TFLOPS | 1,979 TFLOPS |
| BF16 Tensor Core | 989 TFLOPS | 1,979 TFLOPS |
| FP16 Tensor Core | 989 TFLOPS | 1,979 TFLOPS |
| FP8 Tensor Core | 1,979 TFLOPS | 3,958 TFLOPS |
| INT8 Tensor Core | 1,979 TOPS | 3,958 TOPS |

#### Connectivity & Power

| Feature | Value |
|---------|-------|
| TDP | Up to 700W (configurable) |
| NVLink | 900 GB/s bidirectional (NVLink 4.0) |
| PCIe | Gen5, 128 GB/s (x16) |
| MIG | Up to 7 instances @ 10 GB each |
| NVSwitch | Yes (in DGX/HGX configurations) |

---

### H100 PCIe — Secondary Variant

| Spec | Value |
|------|-------|
| Architecture | Hopper |
| CUDA Cores | 16,896 (some sources report 14,592) |
| VRAM | 80 GB HBM2e |
| Memory Bandwidth | 2.0 TB/s |
| Form Factor | PCIe dual-slot, air-cooled |

#### Compute Performance (H100 PCIe)

| Precision | Dense | With 2:4 Sparsity |
|-----------|-------|-------------------|
| FP64 | 26 TFLOPS | — |
| FP64 Tensor Core | 51 TFLOPS | — |
| FP32 | 51 TFLOPS | — |
| TF32 Tensor Core | 378 TFLOPS | 756 TFLOPS |
| BF16 Tensor Core | 756 TFLOPS | 1,513 TFLOPS |
| FP16 Tensor Core | 756 TFLOPS | 1,513 TFLOPS |
| FP8 Tensor Core | 1,513 TFLOPS | 3,026 TFLOPS |
| INT8 Tensor Core | 1,513 TOPS | 3,026 TOPS |

#### Connectivity & Power

| Feature | Value |
|---------|-------|
| TDP | 300–350W (configurable) |
| NVLink | 600 GB/s |
| PCIe | Gen5 |
| MIG | Up to 7 instances |

> **Note:** The PCIe variant trades peak performance for air-cooled deployment and standard PCIe slot compatibility. HBM2e (vs HBM3) contributes to the lower memory bandwidth.

---

### H100 NVL — Dual-GPU Bridge Variant

| Spec | Value |
|------|-------|
| Architecture | Hopper |
| VRAM | 94 GB HBM3 per GPU |
| Memory Bandwidth | 3.9 TB/s per GPU |
| TDP | 350–400W per GPU |
| NVLink | 600 GB/s |
| MIG | Up to 7 MIGs @ 12 GB each |

> **Note:** H100 NVL pairs two H100 GPUs via a high-speed NVLink bridge, designed for large-model inference where memory capacity is the bottleneck. Total system memory: 188 GB HBM3.

---

## H200 SXM — Hopper with More Memory

| Spec | Value |
|------|-------|
| Architecture | Hopper (same compute die as H100) |
| CUDA Cores | 16,896 |
| Tensor Cores | 528 (4th generation) |
| VRAM | 141 GB HBM3e |
| Memory Bandwidth | 4.8 TB/s |
| TDP | 700W |
| Form Factor | SXM5 |

#### Compute Performance (H200 SXM)

Identical to H100 SXM5 — the compute engines are the same.

| Precision | Dense | With 2:4 Sparsity |
|-----------|-------|-------------------|
| FP8 Tensor Core | 1,979 TFLOPS | 3,958 TFLOPS |
| BF16/FP16 Tensor Core | 989 TFLOPS | 1,979 TFLOPS |
| TF32 Tensor Core | 989 TFLOPS | 1,979 TFLOPS |

#### Key Advantages over H100 SXM

- **76% more memory capacity** (141 GB vs 80 GB)
- **43% more memory bandwidth** (4.8 TB/s vs 3.35 TB/s)
- Same compute performance — purely a memory upgrade
- Critical for large model inference (e.g., Llama 70B+ fits entirely in HBM)

---

## B200 — Blackwell Standalone GPU

| Spec | Value |
|------|-------|
| Architecture | Blackwell |
| Process | TSMC 4NP |
| Transistors | 208 billion (dual-die design) |
| Die | Two GB100 dies connected by 10 TB/s NV-HBI interconnect |
| Total Die Size | ~1,620 mm² (two dies) |
| CUDA Cores | 18,944 (per die; some sources vary) |
| Tensor Cores | 592 (5th generation) |
| SMs | 148 per die |
| Boost Clock | 1.98 GHz |
| VRAM | 192 GB HBM3e |
| Memory Bandwidth | 8.0 TB/s (some sources report 7.7 TB/s) |
| Memory Bus | 8,192-bit |
| L2 Cache | 64 MB |
| TDP | 1,000W |
| NVLink | 1.8 TB/s bidirectional (NVLink 5.0) |
| PCIe | Gen6 |
| Form Factor | SXM6 |

#### Compute Performance (B200)

| Precision | Dense | With 2:4 Sparsity |
|-----------|-------|-------------------|
| FP64 | 37 TFLOPS | — |
| FP32 | 75 TFLOPS | — |
| BF16 Tensor Core | 2,250 TFLOPS | 4,500 TFLOPS |
| FP16 Tensor Core | 2,250 TFLOPS | 4,500 TFLOPS |
| FP8 Tensor Core | 4,500 TFLOPS | 9,000 TFLOPS |
| FP4 Tensor Core | 9,000 TFLOPS | 18,000 TFLOPS |

> **Note:** FP4 is a new capability introduced with Blackwell. Dense values are exactly 1/2 of sparse values. The dual-die NV-HBI interconnect runs at 10 TB/s and is transparent to software.

---

## GB200 Grace Blackwell Superchip

| Spec | Value |
|------|-------|
| Configuration | 1 Grace CPU + 2 Blackwell GPUs |
| CPU | 72 Arm Neoverse V2 cores |
| CPU Memory | Up to 480 GB LPDDR5X |
| CPU Bandwidth | Up to 512 GB/s |
| GPU Memory | Up to 372 GB HBM3e (186 GB per GPU) |
| GPU Bandwidth | 16 TB/s (combined) |
| NVLink-C2C (CPU↔GPU) | 900 GB/s |
| NVLink (GPU↔GPU) | 3.6 TB/s |

#### System Compute Performance (GB200 — 2×B200 GPUs)

| Precision | Sparse | Dense |
|-----------|--------|-------|
| FP4 Tensor Core | 40 PFLOPS | 20 PFLOPS |
| FP8 Tensor Core | 20 PFLOPS | 10 PFLOPS |
| FP16/BF16 Tensor Core | 10 PFLOPS | 5 PFLOPS |
| TF32 Tensor Core | 5 PFLOPS | 2.5 PFLOPS |
| FP32 | 160 TFLOPS | — |
| FP64 | 80 TFLOPS | — |
| FP64 Tensor Core | 80 TFLOPS | — |

> **Note:** PFLOPS = petaflops = 1,000 TFLOPS. GB200 combines Grace CPU compute with Blackwell GPU compute. CPU-only TFLOPS are modest; the value is in the unified memory architecture and CPU↔GPU NVLink-C2C bandwidth.

---

## GB200 NVL72 — Rack-Scale System

| Spec | Value |
|------|-------|
| Configuration | 36 Grace CPUs + 72 Blackwell GPUs |
| Total GPU Memory | ~13.5 TB HBM3e (up to 13.4 TB effective) |
| Total CPU Memory | ~17 TB LPDDR5X |
| Aggregate NVLink Bandwidth | 130 TB/s |
| Cooling | 100% liquid-cooled |
| Form Factor | Full rack (OCP-compatible) |

#### Aggregate Compute Performance (GB200 NVL72)

| Precision | Sparse | Dense |
|-----------|--------|-------|
| NVFP4 Tensor Core | 1,440 PFLOPS | 720 PFLOPS |
| FP8 Tensor Core | 720 PFLOPS | 360 PFLOPS |
| FP16/BF16 Tensor Core | 360 PFLOPS | 180 PFLOPS |
| TF32 Tensor Core | 180 PFLOPS | 90 PFLOPS |
| FP32 | 5,760 TFLOPS | — |
| FP64 | 2,880 TFLOPS | — |

> **Note:** NVL72 is a disaggregated rack-scale system. All 72 GPUs are connected via NVLink switch fabric, presenting as a single logical GPU pool to software. This is the platform for training and serving frontier-scale models.

---

## GB300 NVL72 — Blackwell Ultra Rack-Scale

| Spec | Value |
|------|-------|
| Configuration | 36 Grace CPUs + 72 Blackwell Ultra (B300) GPUs |
| Per-GPU VRAM | 288 GB HBM3e |
| Total GPU Memory | ~21 TB HBM3e |
| NVLink | 1.8 TB/s per GPU (NVLink 5) |
| Networking | Up to 800 Gb/s via ConnectX-8 SuperNICs |
| Cooling | Liquid-cooled |
| Form Factor | Full rack |

#### Key Specs

- **~50× overall increase** in AI factory output vs Hopper platforms (NVIDIA claimed)
- Higher per-GPU memory (288 GB vs 192 GB on B200) enables larger models without disaggregation
- Same NVLink 5 fabric as GB200 NVL72 but with upgraded GPU silicon

> **Status:** GB300 is "Blackwell Ultra" generation. Specific per-GPU TFLOPS figures follow the B300 spec below.

---

## B300 — Blackwell Ultra Standalone GPU (Reference)

| Spec | Value |
|------|-------|
| Architecture | Blackwell Ultra |
| Transistors | 208 billion |
| CUDA Cores | 20,480 |
| Tensor Cores | 640 (5th generation) |
| VRAM | 288 GB HBM3e |
| Memory Bandwidth | 8.0 TB/s |
| NVFP4 Dense Performance | 15 PFLOPS (50% over B200's 10 PFLOPS dense) |
| TDP | Up to 1,400W |
| NVLink | 1.8 TB/s bidirectional (NVLink 5.0) |
| Form Factor | SXM6 |

> **Note:** B300 figures are preliminary/estimated based on available NVIDIA disclosures. Final specs may differ. B300 improves over B200 primarily through higher memory capacity (288 GB vs 192 GB) and ~50% higher FP4 throughput.

---

## Calculation Reference

### Memory Bandwidth Utilization

$$\text{BW Utilization (\%)} = \frac{\text{Operational Intensity (FLOPs/Byte)} \times \text{Achieved TFLOPS}}{\text{Peak Bandwidth (TB/s)} \times 1000} \times 100$$

**Roofline model shortcut:**

$$\text{Arithmetic Intensity Breakpoint} = \frac{\text{Peak TFLOPS}}{\text{Peak TB/s}}$$

| GPU | AI Breakpoint (FLOPs/Byte) |
|-----|---------------------------|
| H100 SXM5 | 3,958 / 3,350 ≈ 1.18 (FP8 sparse) |
| H200 SXM | 3,958 / 4,800 ≈ 0.82 (FP8 sparse) |
| B200 | 18,000 / 8,000 ≈ 2.25 (FP4 sparse) |

> **Interpretation:** If your kernel's operational intensity is below the breakpoint, it's memory-bound. If above, it's compute-bound.

### Converting Between Precision Throughputs

For Tensor Cores, the general hierarchy is:

| Relationship | Rule |
|-------------|------|
| Sparse vs Dense | Sparse = 2× Dense (2:4 structured sparsity) |
| FP8 vs FP16/BF16 | FP8 = 2× FP16/BF16 |
| FP4 vs FP8 | FP4 = 2× FP8 |
| FP16 vs TF32 | FP16 ≈ 2× TF32 (varies by architecture) |
| BF16 vs FP16 | BF16 = FP16 (same throughput on Tensor Cores) |

### Estimating Memory-Bound Kernel Performance

$$\text{Achievable throughput} = \text{Memory Bandwidth} \times \text{Bytes per FLOP}$$

Example: A kernel that reads a weight matrix once per output (e.g., linear layer inference):

- H100 SXM5: 3.35 TB/s × (1 FP16 element = 2 bytes) = 1,675 TFLOPS achievable (memory-bound)
- B200: 8.0 TB/s × (1 FP8 element = 1 byte) = 8,000 TFLOPS achievable

### Power Efficiency

$$\text{Perf/Watt} = \frac{\text{Achieved TFLOPS}}{\text{TDP (W)}}$$

| GPU | FP8 Sparse TFLOPS/W | FP4 Dense TFLOPS/W |
|-----|---------------------|---------------------|
| H100 SXM5 | 5.65 | — |
| B200 | 9.0 | 9.0 |
| B300 (est.) | — | 10.7 |

---

## Notes

### Sparsity vs Dense

NVIDIA Tensor Cores support **2:4 structured sparsity**: for every 4 consecutive elements in a weight matrix, at least 2 must be zero. When enabled, the hardware processes the non-zero elements at **2× the dense throughput**. This is not free — the model must be pruned or trained with sparsity in mind. In practice:

- **Dense values** are the "real" FLOPS you get without sparsity optimization
- **Sparse values** are 2× dense and require 2:4 structured pruning to achieve
- For **inference workloads**, dense values are typically the more relevant metric unless the model is specifically sparsity-optimized
- For **training**, sparsity is harder to exploit and dense values dominate

> **Rule of thumb:** If a spec sheet only shows one number, check whether it's dense or sparse. NVIDIA marketing often leads with sparse numbers.

### HBM Generations

| Generation | Typical Bandwidth per Stack | Used In |
|-----------|---------------------------|---------|
| HBM2e | ~400 GB/s per stack | H100 PCIe |
| HBM3 | ~800 GB/s per stack | H100 SXM5, H100 NVL |
| HBM3e | ~1.2 TB/s per stack | H200, B200, B300, GB200 |

HBM3e provides ~50% more bandwidth per stack than HBM3, which is why H200 achieves 4.8 TB/s vs H100's 3.35 TB/s with the same compute die.

### NVLink Versions

| Version | Bandwidth (bidirectional) | Introduced |
|---------|--------------------------|------------|
| NVLink 3.0 | 600 GB/s | A100 |
| NVLink 4.0 | 900 GB/s | H100 SXM5 |
| NVLink 5.0 | 1,800 GB/s (1.8 TB/s) | B200, GB200 |

NVLink 5.0 also introduced NVLink-Chip-to-Chip (NVLink-C2C) for CPU↔GPU connectivity at 900 GB/s in the Grace Blackwell superchip.

### Form Factors

| Form Factor | Description | Typical Use |
|------------|-------------|-------------|
| **PCIe** | Standard PCI Express card, air-cooled | Edge, on-prem servers |
| **SXM5** | NVIDIA mezzanine module, liquid-cooled, NVLink switch | DGX, HGX, cloud |
| **SXM6** | Next-gen mezzanine for Blackwell | DGX B200, HGX B200 |
| **NVL** | Dual-GPU NVLink bridge pair | Large-model inference |
| **Superchip** | Grace CPU + Blackwell GPUs on one board | GB200 NVL systems |
| **Rack (NVL72)** | Full rack with NVLink switch fabric | Frontier training/inference |

### Key Unit Conversions

| Unit | Value |
|------|-------|
| 1 PFLOPS | 1,000 TFLOPS |
| 1 EFLOPS | 1,000 PFLOPS |
| 1 TB/s | 1,000 GB/s |
| 1 PB/s | 1,000 TB/s |

---

## Sources

- NVIDIA H100 Tensor Core GPU Datasheet (Rev. 2)
- NVIDIA H200 Tensor Core GPU Datasheet
- NVIDIA B200 Tensor Core GPU Datasheet
- NVIDIA GB200 NVL72 Datasheet
- NVIDIA GTC 2024 Keynote (Blackwell launch)
- NVIDIA GTC 2025 Keynote (Blackwell Ultra / GB300)
- NVIDIA DGX B200 Technical Brief
- Verified against AnandTech, ServeTheHome, and SemiAnalysis technical breakdowns

---

*This document is a living reference. Update when new NVIDIA datasheets or verified benchmarks are published.*
