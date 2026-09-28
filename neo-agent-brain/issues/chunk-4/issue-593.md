---
id: 593
title: Container health diagnosis cannot see a service starved at its CPU quota
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T17:09:12Z'
updatedAt: '2026-09-28T16:39:15Z'
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
closedAt: '2026-09-28T16:39:15Z'
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
3. **Surfaced, not acted on.** The fact reaches the service's diagnosis in the deployment snapshot and the heal-event ledger through the existing record terminal. The FM System view shows a diagnosis's recovery class and action today, not this fact's ratio. No new action class and no actuator change.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `cpu-throttling` fact | `ContainerHealthDiagnosisService#collectStatsFacts` | Emitted when the throttled share of periods over the sustained window reaches `cpuThrottledRatio`. Carries `quotaCpus`, `throttledRatio`, `throttledPeriods`, `periods`, `throttledTimeMs`, `threshold` and the observed window. Non-authoritative | No fact when the counters are unmeasured or reset, below the ratio, over a window shorter than the CPU clock, without a declared quota, or without an inspect/stats identity every sample shares | method JSDoc | diagnosis arms, and the bridge arm with inspect A / stats A against A / B |
| CPU saturation percent | same | Read against `HostConfig.NanoCpus` under that proved identity | The per-core reading, `quotaCpus: null` | same | diagnosis arm |
| `cpuThrottledRatio` | the service's config defaults, beside `cpuSaturationPercent` | 0.25 | — | config comment | mc-server arm |
| Classifier outcome | `classifyDiagnosis`, last branch | `cpu-throttling-recorded` → the existing record terminal | An actionable diagnosis keeps its own route | — | diagnosis arm |

## Decision Record impact

aligned-with ADR-0025 §2.4 (implements the windowed CPU detect signal against the container's quota, adds a detect fact, no action-class change) · ADR-0026 unchanged.

## Acceptance Criteria

- [ ] **AC-1:** a service whose throttled-period ratio over the sustained window passes the declared threshold yields a CPU-throttling fact carrying the ratio and window; below it, no fact. Unit arms over recorded stats samples include mc-server's measured shape (10,670 of 19,124).
- [ ] **AC-2:** CPU% is reported against the container's CPU quota when one is set, and unchanged when none is.
- [ ] **AC-3:** the fact reaches the service's diagnosis in the deployment snapshot and the heal-event ledger through the existing record terminal. No action class or actuator operation changes.
- [ ] **AC-4 (post-merge, live)** `[L4-deferred — agent-owned: after the plane runs this revision]`**:** on the local plane, mc-server's row carries the fact while it is throttled. Receipt on #64, where it tests the AC-9 / AC-10 hypothesis.

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
- 2026-09-28T13:31:04Z @neo-opus-vega referenced in commit `0d052d3` - "refactoring(orchestrator): the ADR citations carry the typed escape, and the spec drops its review provenance (#593)

The archaeology guard in neo-agent-skills 0.1.19 scans every changed file
whole and no longer accepts the legacy `ticket-ref-ok` escape, so the seven
in ContainerHealthDiagnosisService and its spec, all present on dev before
this branch, failed this PR. Each now reads
`ADR-NNNN [not-ticket-ref: decision-record authority] §x`, the in-tree shape
#600 settled: the section stays in the prose and the escape sits on one
line. The spec's "Window provenance" header loses its reviewer and RA
reference.

Comment-only: the guard reports 0 of the 8, and the spec passes 123/123."
- 2026-09-28T15:44:31Z @neo-opus-vega referenced in commit `8eca865` - "fix(orchestrator): the CPU quota and the throttling window need one proved container (#593)

Inspect and stats are separate reads, and a recreate between them lands
them on different containers. The bridge already proves which container
each read hit, and hands diagnosis `runtimeContainerId` only when the
two agree. `collectStatsFacts` did not read it.

It now applies the provider-residual gate: the quota divides CPU, and the
throttling counters form one window, only when every sample carries the
proved identity. Without that proof, CPU keeps its per-core reading and
no throttling fact is claimed. The fact also needs a declared quota, so
"starved at its quota" is true by construction.

The quota arms now carry a proved identity. A diagnosis arm covers four
cases: no quota, no proof, and a window spanning two containers, beside
a positive control. A bridge arm runs inspect A with stats A against
inspect A with stats B through `collectSnapshot`."
- 2026-09-28T15:47:20Z @neo-opus-vega referenced in commit `dd36fa3` - "refactoring(orchestrator): the bridge spec's comments drop their review provenance (#593)

Touching the bridge spec brings the whole file under the archaeology guard.
Three comments named a review or ticket as their reason, and they now
state the reason itself. The ADR-0019 citation takes the typed escape.
Comment-only."
- 2026-09-28T15:57:17Z @neo-opus-vega cross-referenced by #614
- 2026-09-28T16:03:34Z @neo-opus-vega cross-referenced by PR #615
- 2026-09-28T16:39:15Z @tobiu referenced in commit `3c64cf5` - "Merge pull request #594 from neomjs/vega/593-cpu-throttling-fact

fix(orchestrator): container health diagnosis sees a service starved at its CPU quota (#593)"
- 2026-09-28T16:39:15Z @tobiu closed this issue

