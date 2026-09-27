---
id: 593
title: Container health diagnosis cannot see a service starved at its CPU quota
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T17:09:12Z'
updatedAt: '2026-09-27T17:09:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/593'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Container health diagnosis cannot see a service starved at its CPU quota

## Context

The plane's own immune system did not notice a starved Memory Core. Measured on `neo-local-agent-os` on 2026-09-27:

- mc-server runs at a one-CPU quota, `cpus: "1.0"` (`deploy/cloud/docker-compose.yml:400`, a literal).
- Docker's own stats for it: `cpu_stats.throttling_data` = 10,670 of 19,124 periods throttled (56%), 186 s in total, since its 16:34:57Z start. The previous incarnation was throttled 6,132 times, 125.6 s over 20 min, during a full corpus re-ingestion (cgroup `cpu.stat`).
- In the same window: `healthcheck` averaged 6 s (max 85 s) and `list_messages` 1.25 s (`get_memory_core_tool_metrics`). These are the open misses on #64 AC-9 and AC-10.
- The deployment snapshot at 17:05:15Z said mc-server `status: available`, `diagnosis: healthy`, with a sampled `cpuPercent` of 79.45.

The operator placed it: CPU pressure on a plane service is the self-diagnostics' job (ADR-0025 / ADR-0026), not a hand-tuned compose value.

Live latest-open sweep: the latest 20 open Brain issues at 17:07:58Z, plus an org search for "container health diagnosis cpu": no equivalent. A2A in-flight sweep, last 15 messages: no claim on this. Memory Core sweep: no prior decision on CPU quota or throttling. neo#16636 (`throttle-shed` reconciliation) is closed.

## The Problem

ADR-0025 §2.4 names the detect signal: "CPU load sampled over a window". The implementation reads it as Docker's per-core CPU% against a fixed `cpuSaturationPercent: 90`. That misses the case above in two ways:

- **Quota-blind.** A container capped at one CPU tops out near 100% and averages below 90 while it hits the cap every period. A four-CPU container would cross 90 at a quarter of its quota.
- **Averaging-blind.** Bursty load at the cap averages under the threshold. The direct at-cap signal, throttled periods, sits in the same stats object and is never read.

## The Architectural Reality

- `ContainerHealthDiagnosisService.mjs`: `calculateDockerCpuPercent` (`:1641`) computes `(cpuDelta / systemDelta) * onlineCpus * 100`, where `online_cpus` is the host's 8, not the quota. `cpuSaturationPercent: 90` (`:130`). The sustained window is `summarizeSustainedWindow` over those percents (`:647`).
- `cpu_stats.throttling_data` (`periods`, `throttled_periods`, `throttled_time`) arrives in the stats samples the service already collects. The quota is `HostConfig.NanoCpus` on inspect. No new read and no new privilege.
- `SERVICE_CLASS_BY_KEY` declares mc-server `transient` (`:88`), so its exhaustion route is `throttle-shed`. ADR-0026 §2.4 says no lifecycle operation implements that class today, so a diagnosis there takes the existing record-with-diagnosis terminal. That is enough for the detect half.

## The Fix

1. **A throttling detect fact.** Over the sustained window, `Δthrottled_periods / Δperiods` from `throttling_data` becomes a CPU-throttling fact with its ratio and window whenever it passes a declared threshold beside `cpuSaturationPercent`.
2. **Quota-relative CPU%.** When `NanoCpus` is set, CPU% is expressed against the quota. An unset quota keeps today's per-core figure.
3. **Surfaced, not acted on.** The fact reaches the service's diagnosis in the deployment snapshot, which the FM System view already renders, and the heal-event ledger through the existing record terminal. No new action class and no actuator change.

## Decision Record impact

aligned-with ADR-0025 §2.4 (implements the windowed CPU detect signal against the container's quota, adds a detect fact, no action-class change) · ADR-0026 unchanged.

## Acceptance Criteria

- [ ] **AC-1:** a service whose throttled-period ratio over the sustained window passes the declared threshold yields a CPU-throttling fact carrying the ratio and window; below it, no fact. Unit arms over recorded stats samples include mc-server's measured shape (10,670 of 19,124).
- [ ] **AC-2:** CPU% is reported against the container's CPU quota when one is set, and unchanged when none is.
- [ ] **AC-3:** the fact reaches the service's diagnosis in the deployment snapshot and the heal-event ledger through the existing record terminal. No action class or actuator operation changes.
- [ ] **AC-4 (post-merge, live):** on the local plane, mc-server's row carries the fact while it is throttled. Receipt on #64, where it tests the AC-9 / AC-10 hypothesis.

## Out of Scope

- **The act half:** a live CPU-ceiling raise, an `update-cpu-limit` beside ADR-0026 §2.8's memory `raise-ceiling`, would amend 0026. That needs a Discussion, grounded in what this ticket records.
- **mc-server's quota value itself:** decided from the recorded facts, not before them.
- **Reclassifying mc-server's service class.**

## Related

#64 (AC-9 / AC-10) · #579 · ADR-0025 · ADR-0026 · neo#16636

Retrieval Hint: `query_raw_memories("mc-server CPU quota throttling not diagnosed container health")`

Origin Session ID: ea25694a-1ea9-4ba0-a357-932c7d74133a


## Timeline

- 2026-09-27T17:09:14Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-27T17:09:14Z @neo-opus-vega added the `bug` label
- 2026-09-27T17:09:14Z @neo-opus-vega added the `ai` label
- 2026-09-27T17:09:14Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T17:19:43Z @neo-opus-vega cross-referenced by PR #594

