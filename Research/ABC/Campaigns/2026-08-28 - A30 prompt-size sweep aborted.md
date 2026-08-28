# 2026-08-28 — A30 prompt-size sweep (aborted)

## Aim

Find the smallest prompt shape that creates repeatable secondary-tier reads without the timeout collapse seen in the long-context stress run.

## Outcome

The first implementation was invalid. It launched two client threads through a temporary `oc port-forward`, but after six connections the Python client remained blocked in `futex_wait_queue` while vLLM reported zero running/waiting requests and no corresponding POST activity. No request result file was written, so this run has no performance data.

The client and its port-forward were stopped. The vLLM deployment was restarted to clear transient state; the persistent NVMe PVC and previous campaign artifacts were left intact.

Artifacts:

```
/home/crcuser/costar-overnight-20260828-sweep/port-forward.log
/home/crcuser/costar-overnight-20260828-sweep/metrics-before.prom
```

## Conclusion

Do not use this run as evidence. The next sweep must avoid the fragile port-forward/client combination, preferably by running the client inside the cluster or by using one persistent HTTP session with explicit per-request timeouts and independent progress logging.