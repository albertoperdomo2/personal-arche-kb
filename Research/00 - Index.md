# Research — Index

Durable record of research experiments, including hypotheses, experiment runs, failures, fixes, breakthroughs, conclusions, and next decisions.

## Active experiments

- [[ABC/00 - Index|ABC]] — Activity-Based KV Cache Tier Placement: evolving the vLLM KV offload engine from reactive, demand-driven block fetching toward predictive, speculative prefetching across a four-tier hierarchy (GPU HBM / CPU DRAM / NVMe / CephFS). Four-phase path: baseline characterization → naive prefetch N blocks → heuristic prefetch → speculative (XGBoost temperature prediction + cost-aware placement + session-aware prefetch).
- [[KV Cache Offloading/00 - Index|KV Cache Offloading]] — determine when HBM-only, CPU offload, and CPU+NVMe offload produce materially different performance for agentic prefix-reuse workloads.
- [[llm-d/llm-d-sc/00 - Index|llm-d-sc]] — Code review, pipeline analysis, and improvement opportunities for the llm-d semantic classifier runtime (Rust/Candle, gRPC, anchor-based classification). Primary finding: pipeline mechanics are sound; the real lever is anchor quality and generalization.

## Capture conventions

Each experiment lives under `Research/<experiment>/` and keeps:

- a `00 - Index.md` with the research question, current status, latest conclusion, key documents, and open threads;
- dated analysis notes for experiment plans, run batches, breakthroughs, and conclusions;
- direct MLflow links and stable run IDs;
- deployment, workload, topology, software-version, seed, and cache-state details needed to reproduce a run;
- failed or rejected runs with the rejection reason and what changed afterward.

Separate measured observations from interpretations and conclusions. Update the experiment index whenever the working conclusion or next experiment changes.