# Ramesh 1:1 Meeting Index

## Overview
Weekly one-on-one meetings with Ramesh Doddaiah focused on:
- Internal Red Hat team sync (CephFS/KV cache performance)
- Innovation work on KV cache offloading
- Research direction and project planning

---

## Meeting Log

| Date | File | Key Topics |
|------|------|------------|
| [2026-07-24](./2026-07-24.md) | 2026-07-24.md | CephFS performance parity, Activity-based KV cache offloading concept |

---

## Recurring Themes

### Internal Sync Topics
- CephFS performance benchmarking
- NVMe vs CephFS latency/bandwidth comparison
- Multi-replica testing
- Disaggregated PD (prefill/decode) architecture

### Innovation Research
- **Activity-Based KV Cache Offloading** (primary focus)
  - Time series forecasting for hot block prediction
  - 80/20 rule application
  - Compression + prediction dual optimization
- Model selection (XGBoost → deep learning)
- Feature engineering from KV cache events

---

## Open Questions for Future Meetings

1. What are all available KV cache events in llm-d codebase?
2. How to integrate time series model with existing llm-d infrastructure?
3. What metrics define "hot" vs "cold" blocks?
4. How to handle model drift in real-time predictions?
5. Patent filing strategy for predictive caching?

---

## Related Work

- [vLLM KV Cache Offloading](../../Research/KVCacheOffload/) - Main research folder
- [llm-d Performance](../../../Repositories/llm-d/) - Implementation details

---

*Last updated: 2026-07-24*
