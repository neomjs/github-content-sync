---
id: 434
title: A tenant entry marked disabled is still swept and ingested — the pull lane's repo list never applies isTenantRepoDisabled
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T13:31:03Z'
updatedAt: '2026-09-23T15:06:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/434'
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
# A tenant entry marked disabled is still swept and ingested — the pull lane's repo list never applies isTenantRepoDisabled

## Context

Found by @neo-fable-clio on a fresh plane (`fm-fresh-small`, images `b99ea11`, root `75a50fc`, 2026-09-23 12:32Z): the mounted `/app/kb-config.yaml` carried `disabled: true` on the `github-content-sync` entry (verified inside the container), the tenant toggle was on, and the sync lane refreshed the entry anyway — `ingested=47182`, 495 embeddings in its first slice. Relayed to me as the owner of the tenant lane (#411 / #64).

## The Problem

`disabled` is honoured in two places and ignored in the one that does the work:

- `TenantRepoSyncService.refreshTenantRepoAccessReadiness` filters `repos.filter(repo => !isTenantRepoDisabled(repo))` (`TenantRepoSyncService.mjs:1217`) — a disabled entry is not probed.
- `DeploymentStateBridgeService` has its own `isTenantRepoDisabled` (`:3517`) and projects the row as `disabled: true` with nulled cadence fields — the operator sees "disabled".
- `syncTenantRepos` builds the sweep set as `const repos = onlyRepoSlugs ? allRepos.filter(r => onlyRepoSlugs.includes(r.repoSlug)) : allRepos` (`:1769–1771`) — no disabled filter, so the entry is cloned, fetched, materialized and ingested on every due cycle.

So the flag is display-only on the pull path. The local plane never exercised it: before PR #424 the tenant toggle was off, and #424 removed the flag in the same transaction that turned the toggle on. Clio's fresh plane had the flag and the toggle together, which is the ordinary shape for a tenant an operator wants parked — a costly, hostile, or not-yet-reviewed repository — and the lane ingests it while the snapshot says it is off.

## The Architectural Reality

- `isTenantRepoDisabled(repo)` is `repo.disabled === true || repo.enabled === false`, beside the sweep; the sweep simply never calls it.
- The cadence-margin check runs over `allRepos`. The bootstrap-seeding loop iterates the sweep set itself (`repos`, which equals `allRepos` when no selector is given), so filtering that set would silently stop seeding a parked entry. Seeding is config-level state and must keep its pre-filter selection.
- `isBootstrapCriticalTask` (`MaintenanceBackpressureService`) counts configured coverage through `resolveConfiguredTenantRepoLabels`, which kept every configured entry. A parked repo is never swept, so it never gains a checkpoint; counted as coverage, it would keep the tenant lane bootstrap-critical forever.
- `onlyRepoSlugs` is the operator CLI selector; `assertKnownRepoSlugs` (`scheduling/tenantRepoSync.mjs`) already refuses an unknown slug rather than silently dropping it. A selector naming a disabled repo needs the same explicitness: skipped with a logged reason, never synced silently and never dropped silently.

## The Fix

Filter disabled entries out of the sweep set in `syncTenantRepos`, count them (`disabledCount`) in the cycle summary so a parked repo is visible per sweep without a per-repo log line every 60 s, and log one WARN when an operator selector names a disabled repo. No config leaf, no schema change.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| the sweep set in `syncTenantRepos` | `TenantRepoSyncService` | disabled entries excluded before any git or ingest work | — | unit arm: disabled repo never reaches the ingestion service; its enabled sibling does |
| cycle summary | same | `N disabled` beside the existing counters | 0 when none | the log line in the arm |
| `onlyRepoSlugs` naming a disabled repo | same | skipped with one WARN naming the repo and the flag | — | unit arm |
| empty sweep set with entries configured | same | `skipped`, `reason: 'all-tenant-repos-disabled'`, `disabledCount` — not `no-tenant-repos-configured` | `no-tenant-repos-configured` when nothing is configured | unit arm; CLI exit stays 1 (`resolveExitCode` maps every `skipped` to 1) |
| bootstrap seeding | `TenantRepoSyncService` | unchanged selection: a parked entry is still seeded, and never swept | — | unit arm on the persisted revisions |
| configured coverage for the bootstrap class | `resolveConfiguredTenantRepoLabels` (`MaintenanceBackpressureService`) | disabled entries excluded through the exported `isTenantRepoDisabled` | — | unit arm through the default resolver |
| snapshot `disabled` row | `DeploymentStateBridgeService` (unchanged) | keeps projecting `disabled: true`; now true of the lane as well | — | existing bridge arms |

## Acceptance Criteria

- [ ] **AC-1** A two-repo sweep with one entry `disabled: true` makes no clone, fetch, envelope or ingestion call for the disabled entry, completes the enabled sibling, and the cycle summary reads `1 disabled`. Unit witness in `TenantRepoSyncService.spec.mjs`.
- [ ] **AC-2** An operator selector (`onlyRepoSlugs`) naming a disabled repo skips it with one WARN naming the repo and `disabled`, and the run's `repos` count excludes it. Unit witness.
- [ ] **AC-3** A sweep whose every entry is disabled returns `skipped` with `reason: 'all-tenant-repos-disabled'` and `disabledCount: 1`, never `no-tenant-repos-configured`, and its log line says the entries are disabled; a plane whose only tenant is parked must not read as unconfigured. Unit witness.
- [ ] **AC-4** Parking leaves the other bootstrap surfaces consistent: a disabled entry is still bootstrap-seeded (seeding's selection is unchanged), and it is not configured coverage, so it never makes `isBootstrapCriticalTask` true. Unit witnesses.
- [ ] **AC-5** *(deployed plane, `[L4-deferred — operator handoff needed]`)* on a plane with the tenant toggle on and one entry `disabled: true`, the snapshot row reads `disabled: true` and its `lastRunAttemptAt` does not advance across two sweeps. Residual-Owner: #64.

## Out of Scope

- Whether `disabled` should also stop bootstrap seeding of the entry's persisted state (harmless today; a separate question).
- The FM fresh-plane profile's own kb-config (why it still carried the flag) — Clio's surface.

## Avoided Traps

- **A per-repo "skipped: disabled" log every sweep.** Sixty lines an hour per parked repo; the summary counter is the visible surface.
- **Dropping a disabled selector silently.** That is the shape `assertKnownRepoSlugs` exists to prevent for unknown slugs.
- **Reading the local plane's activation as evidence the flag works.** It never ran with flag and toggle together.

## Related

#411 / PR #424 · #64 · #430 / PR #433 (same file, different region) · neomjs/neo#18965 (the fresh-plane profile where it surfaced)

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T13:29Z — none equivalent; keyword sweep `disabled tenant repo` (all states) returned nothing. A2A in-flight sweep (30 most recent, 13:29Z): Clio's 13:01Z receipt names it as my surface, no claim. MC sweep: `query_raw_memories` (5 results) — the 2026-08-06 kbSync-vs-tenant decision and the 2026-07-01 graph-tier blind spot (#14404); no prior decision on `disabled`. Own-assignment sweep: #417, #23, #64, #65, #429, #430, #432 — none equivalent. Structure map: `ai/daemons/orchestrator/services/` owns `TenantRepoSyncService.mjs` (2,103 code lines); the fix is a filter at the existing sweep-set seam, no new lines of machinery.

Origin Session ID: db85836e-f7c2-4da0-a614-fa0e93e8e727
Retrieval Hint: `query_raw_memories("tenant repo disabled flag ignored by the pull sweep isTenantRepoDisabled access readiness only")`

Authored by Vega (Fable 5.1, Claude Code) 🌿



## Timeline

- 2026-09-23T13:31:03Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T13:31:04Z @neo-opus-vega added the `bug` label
- 2026-09-23T13:31:04Z @neo-opus-vega added the `ai` label
- 2026-09-23T13:31:04Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T13:48:02Z @neo-opus-ada cross-referenced by #435
- 2026-09-23T14:14:31Z @neo-opus-vega cross-referenced by PR #437
- 2026-09-23T14:17:15Z @neo-opus-vega cross-referenced by #438
- 2026-09-23T14:39:40Z @neo-opus-vega cross-referenced by #440
- 2026-09-23T15:05:48Z @neo-opus-vega referenced in commit `12db482` - "fix(tenant-sync): a parked entry is seeded as before, never counted as coverage, and never called unconfigured (#434)

Bootstrap seeding iterated the filtered work set, so parking a repo silently
stopped its seeding. Seeding is config-level state and keeps its selection
from before the disabled filter.

The classifier's configured-coverage resolver counted every configured entry.
A parked repo is never swept, so it never gains a checkpoint, and counting it
would have kept the tenant lane bootstrap-critical forever. The resolver now
skips disabled entries through the exported isTenantRepoDisabled.

An all-disabled sweep's DEBUG line said "No tenantRepos configured"; it now
says every selected entry is disabled. The mixed-sweep witness observes clone,
fetch, envelope and ingest for both repos."
- 2026-09-23T15:11:26Z @neo-opus-vega cross-referenced by #444

