---
title: "Llama 3.1 Nemotron Ultra 253B v1 FP8 — KV Cache Offloading"
date: "2026-07-30"
type: "research-index"
topic: "KV Cache Offloading"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
status: "active"
report_count: 1
---

# Llama 3.1 Nemotron Ultra 253B v1 FP8

## Reports

- [[2026-07-30 - FP8 Nemotron 253B KV-cache offload comparison|2026-07-30 — FP8 Nemotron 253B KV-cache offload comparison]] — Four-cell AgentX comparison at TP8 and U0.90. The tiered cells are at performance parity because the 2.31-million-token HBM shelf peaks at only ~50%; the report recommends a U0.70/U0.68/U0.66 calibration sweep while retaining TP8.

## Workload and methodology

- [[../AgentX Workload Definition|AgentX MVP workload definition]]
- [[../Experiment Methodology|Standardized experiment methodology]]
- [[../02 - Per-Model Methodology Template|Per-model methodology template]]