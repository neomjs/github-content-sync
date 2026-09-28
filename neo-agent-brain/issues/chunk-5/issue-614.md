---
id: 614
title: Container health counts reclaimable file cache as used memory
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-28T15:57:16Z'
updatedAt: '2026-09-28T16:56:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/614'
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
closedAt: '2026-09-28T16:56:53Z'
---
# Container health counts reclaimable file cache as used memory

## Context

On the local plane, the container-health diagnosis reads Chroma as out of memory while Chroma's own memory sits under its threshold. Measured 2026-09-28 at 14:52 and 14:55Z, in two identical samples (Docker API, cgroup v2):

| | GiB | of the 16 GiB limit |
|---|---|---|
| `memory_stats.usage` | 13.51 | 84.42 % |
| of which `inactive_file` (reclaimable file cache) | 2.24 | |
| of which `anon` | 11.19 | |
| `usage − inactive_file`, what `docker stats` shows | 11.27 | 70.45 % |

The store threshold is 80 %, so the diagnosis reports exhaustion. The orchestrator's own ledgers show what followed:
- The exhaustion diagnosis appears at every check the heal ledger still holds (since 2026-09-26 04:23 at the latest). It is 4,996 of the ledger's 5,000 events.
- `raise-ceiling` is refused at the knob's cap by design (ADR 0026 §2.8): 163 times since 09-26 06:24, three an hour. `chroma:raise-ceiling` is alarm-only.
- The snapshot's `memoryUsageBytes` carries the same raw usage, so the FM System view would report 13.5 of 16 GiB.

Surfaced by the System design (neomjs/neo-agent-institution#317; defect-note 2026-09-28 15:05Z), promoted on the operator's escalation.

## The Problem

The kernel reclaims inactive file pages before it OOM-kills a cgroup, so counting them as used memory turns a store's page cache into a diagnosis. Chroma reads its files through the page cache, which grows toward the limit in normal operation, so the store threshold fires on ordinary work. In August, Chroma's pressure was real: @neo-gpt measured 22 MB of file cache in 2 GiB, the rest anonymous. The metric was never checked against a cache-heavy reading.

## The Architectural Reality

- `ai/daemons/orchestrator/services/ContainerHealthDiagnosisService.mjs:1743` `calculateDockerMemoryPercent` returns `memory_stats.usage / memory_stats.limit`. `collectStatsFacts` uses it for every container-scoped memory percent (`:630`). Node services read their own heap instead (`resolveMemorySaturationScope`) and are unaffected.
- `DeploymentStateBridgeService.mjs:2798–2804` publishes `memoryPercent` (the same function) and `memoryUsageBytes` (raw `usage`).
- `memory_stats.stats.inactive_file` already arrives in every stats sample, so the fix needs no new read.

## The Fix

- The container memory percent counts `usage − inactive_file`, the way `docker stats` does. A payload without `inactive_file` keeps the raw figure.
- The snapshot publishes the in-use bytes beside the raw usage, so the System view can say "11.3 of 16 GiB in use, plus 2.2 GiB file cache", as the #317 design draws it.
- No threshold, service class or actuator changes.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Container memory percent (diagnosis and snapshot `memoryPercent`) | `calculateDockerMemoryPercent` | `(usage − inactive_file) / limit` | `usage / limit` when `inactive_file` is absent | function JSDoc | unit arm on Chroma's split |
| Snapshot in-use bytes (new field) | the bridge's stats summary | `usage − inactive_file` | null when unreadable | JSDoc | bridge arm |
| Snapshot `memoryUsageBytes` | same | unchanged, raw `usage` | — | — | — |

## Decision Record impact

aligned-with ADR 0025 (the memory detect signal reads what the kernel cannot reclaim) · ADR 0026 unchanged.

## Acceptance Criteria

- [ ] AC-1: Chroma's measured split (13.51 GiB usage, 2.24 GiB inactive file, 16 GiB limit) yields no store memory-exhaustion fact at 80 %, and the same usage without `inactive_file` does. The first half is red on dev.
- [ ] AC-2: a stats payload without `inactive_file` keeps the raw reading.
- [ ] AC-3: the deployment snapshot carries the in-use bytes beside the raw usage, and its `memoryPercent` reads the in-use share.
- [ ] AC-4 (post-merge, live) `[L4-deferred — agent-owned: after the plane runs this revision]`: on the local plane, Chroma's diagnosis leaves exhaustion while its in-use share stays under 80 %, and the refused `raise-ceiling` records stop.

## Out of Scope

- **The heal ledger's retention.** One repeating record can fill all 5,000 events. This fix stops it for this diagnosis only; the general case is separate.
- **Chroma's cap, and its growth past it:** #60.
- **Heap-scoped services**, which do not use this percent.

## Avoided Traps

- **Raising the store threshold.** It would also hide a real exhaustion; the numerator is wrong, not the bar.
- **Subtracting all file cache.** Active file pages are the working set, and `docker stats` subtracts only the inactive list. Today's samples had 0 active file pages, so the numbers above do not depend on this choice.

## Related

#60 · #593 (the CPU sibling in the same service) · #594 · neomjs/neo-agent-institution#317 · ADR 0025 · ADR 0026 §2.8

Live latest-open sweep: latest 20 open Brain issues at 2026-09-28 15:55Z, plus an org-wide search for page cache, `inactive_file`, file cache memory, `memory_stats.usage`, `calculateDockerMemoryPercent` and chroma exhaustion diagnosis. No equivalent: #60 and #56 cover the cap and the heap ceiling.
A2A in-flight sweep: the last 30 messages carry no claim on this.
MC sweep: @neo-gpt's 2026-08-01 Chroma reading (file cache 22 MB, pressure real). No prior decision on excluding cache.
Own-assignment sweep: #593 is the CPU sibling in the same service; no overlap.
Structure map: `ai/daemons/orchestrator/services`, both files, run 15:56Z.
Origin Session ID: ec438462-8e98-425e-92c3-723cdc3c9704
Retrieval Hint: "container memory percent page cache inactive_file chroma exhaustion raise-ceiling refused"


## Timeline

- 2026-09-28T15:57:17Z @neo-opus-vega added the `bug` label
- 2026-09-28T15:57:17Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-28T15:57:17Z @neo-opus-vega added the `ai` label
- 2026-09-28T15:57:17Z @neo-opus-vega added the `agent-os` label
- 2026-09-28T15:57:43Z @neo-opus-vega cross-referenced by #60
- 2026-09-28T16:03:34Z @neo-opus-vega cross-referenced by PR #615
- 2026-09-28T16:16:07Z @neo-opus-vega referenced in commit `c9f8ad0` - "refactoring(orchestrator): the touched files' comments carry the typed escape and drop their review provenance (#614)

The archaeology guard reads each changed file whole. This applies #594's comment-only migration of the diagnosis service, its spec and the bridge spec unchanged, so the two PRs merge clean in either order. The four legacy references in the bridge service move to the same form."
- 2026-09-28T16:41:17Z @neo-opus-vega referenced in commit `7c5bcab` - "chore(orchestrator): merge dev after #594, both PRs' spec helpers and arms kept (#614)"
- 2026-09-28T16:56:54Z @tobiu referenced in commit `4291291` - "Merge pull request #615 from neomjs/vega/614-memory-in-use

fix(orchestrator): container memory counts what the kernel cannot reclaim first (#614)"
- 2026-09-28T16:56:54Z @tobiu closed this issue

