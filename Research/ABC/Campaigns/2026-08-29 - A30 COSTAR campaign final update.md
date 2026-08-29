# 2026-08-29 — A30 COSTAR campaign final update

The isolated ARC treatment has completed and the deployment has been restored to LRU.

The exact pressure/reuse sequence produced:

- LRU: seed 45.56 s; eight unique contexts 58.66–58.68 s; reuse after pressure 8.02 s.
- ARC: seed 45.94 s; eight unique contexts 58.81–58.89 s; reuse after pressure 8.03 s.

All requests succeeded. ARC did not improve readiness or end-to-end latency within measurement noise. This is a useful negative result: generic ARC replacement is not sufficient for the remaining gap.

The positive signal remains explicit continuation locality: immediate exact repeats were 2.31 s, while post-pressure reuse was ~8.0 s and cold unique contexts ~58.7 s. The next serious treatment should therefore carry continuation/session value into a soft CPU retention priority or bounded TTL. It should not add another generic eviction heuristic.

The original LRU configuration has been restored. Treatment artifacts are under:

```
/home/crcuser/costar-overnight-20260829-arc/
```