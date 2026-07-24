# Learnings

TILs, patterns, and gotchas that are worth keeping but aren't tied to a single incident — tooling tricks, mental models, config that bit you, benchmarking notes.

One file per learning keeps them searchable. Name them descriptively (e.g. `vLLM paged-attention memory math.md`).

## Learnings
- [[Arche Vega-Lite renderer guardrails]] — why valid charts can fail Arche's pre-render safety/feature guard, plus safe condition and composition workarounds.
- [[CephFS performance tuning for KV cache offloading]] — six root causes of CephFS regression in KV cache offloading (replication, OSD memory, MDS sizing, mount options, FS tier threads, Squid bug) and the tuning applied to diadochos.