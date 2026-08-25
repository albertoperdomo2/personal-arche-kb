---
title: "COSTAR-KV Experiment 2 — practical forced-admit policy benchmark"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "COSTAR-KV future-value placement / Experiment 2"
status: "conditionally-valid-negative-result"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
vllm_version: "v0.27.0-derived offline tooling"
code_base_revision: "e50f7d36960980c0c89651ffd0ce281a9fb8a466 plus uncommitted tools/costar experiment code"
tensor_parallelism: 8
concurrency: 32
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
secondary_tier: "filesystem-backed node-local NVMe"
workload: "AgentX/Weka C32"
random_seed: "deterministic; no randomized policy"
cache_cleaning_state: "offline replay of recorded native state"
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
corpus_sha256: "788adedcbc9850a64443681fbd8acf36ea62cdce950eeac6311f730f34cce687"
---

# COSTAR-KV Experiment 2 — practical forced-admit policy benchmark

## Aim

Experiment 1 established a large clairvoyant placement opportunity: with the same 131,072 CPU slots and the same recorded CPU-ready arrivals, a next-use policy made all 42 external targets CPU-complete and avoided all 12 native reads. Experiment 2 asks the next practical question:

> Can a low-overhead, history-only forced-admit policy recover a meaningful part of that opportunity without future knowledge?

The tested policies were LRU, FIFO, vLLM v0.27 ARC semantics, aged LFU, LRFU, 2Q-style, and LRU-2. The ATC'25 category-conditioned baseline was deliberately not fabricated: the accepted trace does not contain a stable workload/session category label required for a faithful implementation.

## Validity verdict — conditionally valid negative result

This is valid for comparing the listed policies under one deterministic, equal-capacity replay:

- all policies receive the exact same recorded CPU-ready arrival stream;
- every previously absent arrival is admitted;
- every policy has exactly 131,072 resident slots;
- request deadlines and complete external key targets come from the accepted C32 corpus;
- one warm-up and three timed replays are run per policy; reported Python runtime is the median.

It is not an end-to-end or globally endogenous simulator. Counterfactual demand fills, live protected-block constraints, transfer overlap, and the service cost of newly created misses are not modeled. Therefore, "service avoided" is a **gross** quantity. The request-level net miss delta is the safer policy comparison.

## Results

The recorded physical placement is a reference, not one of the replayed algorithms. The clairvoyant next-use result from Experiment 1 is an upper-bound diagnostic, not a practical policy.

| Placement | Complete external targets | External key hits | Recorded reads avoided | Gross service avoided | New external misses | Net miss delta vs recorded |
|---|---:|---:|---:|---:|---:|---:|
| Recorded physical reference | 30/42 (71.43%) | 108,839/157,283 (69.20%) | 0/12 | 0.000 s | 0 | 0 |
| LRU | 27/42 (64.29%) | 99,955/157,283 (63.55%) | 2/12 | 5.620 s | 5 | **+3** |
| FIFO | 27/42 (64.29%) | 97,325/157,283 (61.88%) | 3/12 | 8.035 s | 6 | **+3** |
| vLLM ARC | 24/42 (57.14%) | 104,696/157,283 (66.57%) | 2/12 | 5.620 s | 8 | **+6** |
| Aged LFU, 600 s | 25/42 (59.52%) | 106,158/157,283 (67.49%) | 2/12 | 5.620 s | 7 | **+5** |
| LRFU, 600 s | 26/42 (61.90%) | 106,214/157,283 (67.53%) | 2/12 | 5.620 s | 6 | **+4** |
| 2Q-style, 25% probation | 23/42 (54.76%) | 103,678/157,283 (65.92%) | 2/12 | 5.620 s | 9 | **+7** |
| LRU-2 reuse-distance approximation | 24/42 (57.14%) | 104,696/157,283 (66.57%) | 2/12 | 5.620 s | 8 | **+6** |
| Clairvoyant next-use reference | 42/42 (100%) | 157,283/157,283 (100%) | 12/12 | 36.440 s | 0 | -12 |

A positive net miss delta is harmful: the policy creates more newly incomplete external requests than recorded native-read requests it makes complete.

Figure 1 shows the policy result at the correct unit of value: a request needs its complete external target, not merely many of its blocks.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Complete external requests under equal CPU capacity","width":720,"height":300,"data":{"values":[{"policy":"Recorded","complete_percent":71.43,"class":"Reference"},{"policy":"LRU","complete_percent":64.29,"class":"Practical"},{"policy":"FIFO","complete_percent":64.29,"class":"Practical"},{"policy":"vLLM ARC","complete_percent":57.14,"class":"Practical"},{"policy":"Aged LFU","complete_percent":59.52,"class":"Practical"},{"policy":"LRFU","complete_percent":61.90,"class":"Practical"},{"policy":"2Q-style","complete_percent":54.76,"class":"Practical"},{"policy":"LRU-2","complete_percent":57.14,"class":"Practical"},{"policy":"Next-use oracle","complete_percent":100.0,"class":"Clairvoyant"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy","type":"nominal","title":"Placement policy","sort":null,"axis":{"labelAngle":-25}},"y":{"field":"complete_percent","type":"quantitative","title":"External targets CPU-complete (%)","scale":{"zero":true,"domain":[0,100]}},"color":{"field":"class","type":"nominal","title":"Policy class","scale":{"domain":["Reference","Practical","Clairvoyant"],"range":["#6b7280","#4c78a8","#e45756"]}},"tooltip":[{"field":"policy","type":"nominal","title":"Policy"},{"field":"complete_percent","type":"quantitative","title":"Complete targets (%)","format":".2f"}]}}
~~~

None of the practical policies reaches the recorded 71.43% request completeness. The 100% next-use result shows that placement opportunity still exists; the practical policies simply do not identify it.

Figure 2 makes the regression explicit. All practical policies have a positive net miss delta.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — Net external-request misses created versus recorded placement","width":650,"height":290,"data":{"values":[{"policy":"LRU","net_misses":3},{"policy":"FIFO","net_misses":3},{"policy":"vLLM ARC","net_misses":6},{"policy":"Aged LFU","net_misses":5},{"policy":"LRFU","net_misses":4},{"policy":"2Q-style","net_misses":7},{"policy":"LRU-2","net_misses":6}]},"layer":[{"mark":{"type":"bar","color":"#e45756"},"encoding":{"x":{"field":"policy","type":"nominal","title":"Practical policy","sort":null,"axis":{"labelAngle":-22}},"y":{"field":"net_misses","type":"quantitative","title":"Additional incomplete external requests","scale":{"zero":true}},"tooltip":[{"field":"policy","type":"nominal","title":"Policy"},{"field":"net_misses","type":"quantitative","title":"Net additional misses"}]}},{"mark":{"type":"rule","color":"#222","strokeDash":[5,4]},"encoding":{"y":{"datum":0}}}]}
~~~

## What the metrics mean

- **External key hit ratio** asks how many target chunks were resident. It is a useful diagnostic but not the movement outcome.
- **Complete external targets** asks whether every required external chunk for a request was resident. This is the request-level readiness outcome.
- **Recorded reads avoided** counts baseline native-read requests whose complete target the alternate policy retained.
- **New external misses** counts requests that required no native read in the recorded execution but become incomplete under the alternate policy.
- **Net miss delta** is new misses minus recorded reads avoided. Positive is a regression.
- **Gross service avoided** sums measured service only for eliminated recorded reads. It cannot subtract the unknown service cost of newly created counterfactual reads, so it must not be called net savings.

## Interpretation

### 1. Key-hit ratio would have produced the wrong decision

LRFU has the best practical key-hit ratio, 67.53%, close to the recorded 69.20%. Yet it completes only 26/42 external requests, eliminates 2 recorded misses, creates 6 new misses, and therefore regresses by 4 requests.

KV movement is request-completion oriented: retaining almost all chunks of many requests can be less valuable than retaining every chunk of fewer imminent requests.

### 2. Generic exact-key history is insufficient at first admission

ARC, LFU/LRFU, 2Q, and LRU-2 can protect keys after reuse becomes observable. The decisive placement decision often occurs before a new key has useful exact-key history. These policies therefore preserve previously popular blocks without reliably identifying which newly mirrored prefix will form an imminent complete working set.

This is an inference from the observed policy behavior, not yet a direct feature-availability measurement. That measurement should be the next experiment.

### 3. Gross read avoidance hides substitution

FIFO appears best if viewed only through gross service: it eliminates 3 reads / 8.035 seconds. But it creates 6 new incomplete requests, for a net regression of 3. Without counterfactual new-read timings, no positive service claim is justified.

### 4. The oracle gap remains large

The negative practical result does not invalidate future-value placement. Recorded placement completes 30/42 targets, all practical history-only policies complete 23–27, and clairvoyant next-use completes 42. The unresolved problem is information and objective design: recognize the future contribution of an arriving request/prefix before its first external reuse, and optimize complete request readiness rather than isolated key hits.

## Action and runtime cost

| Policy | Evictions | Metadata entries at end | Median replay time | Median ns/event |
|---|---:|---:|---:|---:|
| LRU | 274,145 | 131,072 | 3.582 s | 6,241 |
| FIFO | 273,634 | 131,072 | 2.197 s | 3,827 |
| vLLM ARC | 271,275 | 262,144 | 6.337 s | 11,040 |
| Aged LFU | 271,275 | 131,072 | 15.493 s | 26,991 |
| LRFU | 271,275 | 131,072 | 10.111 s | 17,615 |
| 2Q-style | 271,275 | 262,144 | 4.512 s | 7,861 |
| LRU-2 | 271,275 | 131,072 | 5.651 s | 9,845 |

These are single-process Python replay costs, not production scheduler overhead. They provide relative complexity evidence only. Aged LFU and LRFU are both slower and no better at the request outcome, so neither is a sensible live candidate from this result.

## Decision

**Do not implement any of these seven forced-admit exact-key history policies in live vLLM on the strength of C32.**

Keep LRU and FIFO as simple offline baselines. Keep the next-use policy as a value-of-information upper-bound diagnostic. Do not claim the opportunity is disproven: the 12-read oracle gap remains, but these mechanisms fail to recover it.

## Next experiment

Run an offline **first-admission information audit** before adding another policy:

1. label each arriving block by whether retaining it contributes to a future complete external request, its time to first demand, request-level reload cost, and prefix coverage;
2. determine what was knowable at that arrival: request identity, prefix offset, prompt/decode origin, session/category if present, neighboring-prefix state, prior request/session reuse, and queue/deadline context;
3. measure how much valuable first reuse is distinguishable using each signal alone and in simple combinations;
4. report request-completion precision/recall and cost-weighted ranking lift, not only block classification accuracy;
5. identify trace fields that are absent before proposing an online implementation.

If the current trace lacks the necessary request/session/category fields, extend the offline instrumentation first. That is a concrete finding, not permission to fabricate ATC'25 or session-aware results.

Only after a signal demonstrates out-of-sample lift over LRU/FIFO and positive request-level net benefit should it become a live admission/eviction candidate.

## Reproduction and verification

Implementation:

- `tools/costar/practical_policy_benchmark.py`
- `tools/costar/run_practical_policy_benchmark.py`
- `tests/tools/test_costar_practical_policy_benchmark.py`

Command:

~~~text
../vllm/.venv/bin/python -m tools.costar.run_practical_policy_benchmark   /private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite   --compact
~~~

Verification:

~~~text
ruff check tools/costar/run_practical_policy_benchmark.py   tests/tools/test_costar_practical_policy_benchmark.py

../vllm/.venv/bin/python -m pytest   tests/tools/test_costar_practical_policy_benchmark.py -q

10 passed
~~~

`git diff --check` passed. The local worktree contains pre-existing uncommitted research and vLLM changes; nothing was committed.

Source corpus: [MLflow C32 run f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow).