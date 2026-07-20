# Incidents

Operational memory: **one file per resolved issue**, so a recurrence months from now is one `search_kb` away.

## Convention
- **Filename:** `YYYY-MM-DD - <short symptom>.md` (e.g. `2026-07-17 - vLLM OOM on H100 node.md`).
- **Shape:** use the [Incident Template](../../Templates/Incident%20Template.md) — Symptom → Environment → Root Cause → Resolution → Prevention → Related.
- **Write for future-you:** put the observable *symptom* (error text, metric, behavior) up top in plain language. That is what search matches on.

## Log
| Date | Symptom | Root cause | File |
| --- | --- | --- | --- |
| 2026-07-20 | Ceph HEALTH_WARN, 7 OSDs crash-looping `Init:Error` "no disk found with OSD ID", ~29% objects degraded on diadochos | Node NVMe disks re-prepared with new OSD IDs after disruption; old OSD entries never cleaned up | [2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos](2026-07-20%20-%20Ceph%20HEALTH_WARN%20orphaned%20OSDs%20crash-looping%20on%20diadochos.md) |
| 2026-07-17 | H100 node NotReady on diadochos, Ceph HEALTH_WARN, kubelet stopped posting status | IBM Cloud instance `failed`, `cannot_start_capacity` in eu-de-2 — no H100 capacity available | [2026-07-17 - IBM Cloud H100 node failed cannot_start_capacity on diadochos](2026-07-17%20-%20IBM%20Cloud%20H100%20node%20failed%20cannot_start_capacity%20on%20diadochos.md) |