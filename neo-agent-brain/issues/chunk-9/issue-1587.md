---
id: 1587
title: The Fleet checks for and fetches harness builds on its own
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-11T00:13:23Z'
updatedAt: '2026-11-11T02:58:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1587'
author: neo-opus-ada
commentsCount: 0
parentIssue: 1531
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-11T02:58:36Z'
---
# The Fleet checks for and fetches harness builds on its own

Sub of #1531, its point 4 ("Fleet acquires"): the missing trigger. Without it, no harness on the operator's Mac ever gets a new build.

## Context

- **Seats are the only users of these apps.** The operator, 2026-11-11: "i do NOT use codex, claude or antigravity outside FM … in case this means 'there will never be updates', it is a failure." #1531 recorded the same constraint on 2026-11-09.
- **Nothing runs acquisition.** On `origin/dev` of Brain and Institution, 2026-11-11, `codexDesktopUpdate.mjs` (#1537) and `antigravityUpdate.mjs` (#1539) have no production importer; only their specs import them. The Out of Scope of both subs names only the cockpit, so no leaf owns *when* acquisition runs.
- **Every seat updater is off.** Codex by `CODEX_SPARKLE_ENABLED=false`, Antigravity by `autoCheckForUpdates`, and Claude by the `disableAutoUpdates` policy, installed 2026-11-10 23:48Z.
- **Net: every harness is frozen at its installed build.**

## The Problem

#1531's terminal predicate has seats run builds the Fleet fetched and admitted itself, with each seat moving to its family's newest admitted build at its next Start. The fetch half exists only as functions nobody calls. So nothing new ever reaches the generation store, apart from the installed bundle, and nobody updates that one.

## The Architectural Reality

- `acquireCodexDesktopUpdate` and `acquireAntigravityUpdate` each read their vendor feed, fetch into staging, verify the download (EdDSA, SHA-512 and length), and hand the bundle to `materializeHarnessGeneration`, which serializes runs per family.
- Adoption is already wired: an opted-in Start launches the newest admitted generation (`FleetLifecycleService.resolveHarnessGeneration`) and never downloads.
- Periodic-job precedent: `wireFleetOpenWorkSource.mjs`, a `wire*` module that `devFleetServer.mjs` starts and stops, with its own `setInterval` pulse.
- Opt-ins: `fleet.harnessGenerations.{codexDesktop,antigravity,claudeDesktop}`, all `false` by default.

## The Fix

`ai/services/fleet/wireFleetHarnessAcquisition.mjs`, started by `devFleetServer.mjs` (the Fleet Manager's server entry) beside `wireFleetOpenWorkSource`:
- It runs each opted-in family's acquisition once soon after boot, without delaying any Start, and then every `fleet.harnessGenerations.checkIntervalHours`, a new leaf.
- One run per family at a time. A failed fetch or a rejected admission is recorded and changes no selection.
- A family without a source is skipped by name; Claude has none until its source sub lands.
- The job's `status()` and the Fleet log carry, per family, `{lastCheckAt, outcome, candidateVersion, selectedVersion}`. Bridge and cockpit rendering stay with #1531 point 7.

Adoption stays at each seat's next Start, so a Fleet restart moves every seat to its family's newest admitted build.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleet.harnessGenerations.checkIntervalHours` (`NEO_FLEET_HARNESS_CHECK_INTERVAL_HOURS`) | `ai/configBase.mjs` leaf | hours between acquisition runs, default 6 | a value past what a Node timer holds (about 596 h) runs at that maximum; a non-finite one wires no job | leaf JSDoc | `wireFleetHarnessAcquisition.spec.mjs` |
| `FleetLifecycleService#harnessAcquisitionContext(harnessType)` | `ai/services/fleet/FleetLifecycleService.mjs` | `{optedIn, root, installedBundle}`, from the reads a Start makes | none | method JSDoc | `FleetLifecycleService.spec.mjs` |
| `wireFleetHarnessAcquisition({context, intervalMs})` → `{pulse, status, stop}` | `ai/services/fleet/wireFleetHarnessAcquisition.mjs`, started by `devFleetServer.mjs` | first run two minutes after boot, then every interval; `status()` names each family `pending`, `not-opted-in`, `no-source`, `checked` or `failed`; `stop()` ends both timers without awaiting a run, on shutdown and on a failed boot | `null` without a context, a positive finite interval or a finite boot delay | module and function JSDoc | `wireFleetHarnessAcquisition.spec.mjs`, `openWorkBoot.spec.mjs` |

## Acceptance Criteria

- [ ] With a family opted in, the Fleet runs its acquisition after boot without delaying any Start, and again each interval (unit tests with clock and acquisition seams).
- [ ] A run that finds a newer build fetches and admits it with no operator action. A failed or rejected run is recorded for its family and changes no selection.
- [ ] A family that is not opted in, or has no source, is never fetched, and the status says which reason applies.
- [ ] Two runs never overlap for one family.
- [ ] **Post-merge** `[L4-deferred — operator handoff needed]` (Residual-Owner: #1531), on the operator's Mac with Antigravity or Codex opted in: the next vendor release reaches a restarted seat with no hand install. Receipt: the ledger's `source` is the feed URL, and the seat's pinned generation is that build.

## Out of Scope

- Turning the opt-ins on: Antigravity's is #1590, and Codex first needs its app-server CLI readers to follow the seat's pin (#1531 point 6).
- Claude's source: its own sub after #1584. The candidates are measured in #1531 comment 6781813120, and the operator's OK on Homebrew's cask record as the pointer is pending.
- The cockpit rendering (#1531 point 7) and retiring unpinned generations.

## Avoided Traps

- **Fetching inside Start or Start Fleet** (rejected in #1531): a download of several hundred MB would hold every Start, and a failed one would block it.
- **Writing into `/Applications`**: seats run generation copies, and the installed bundle stays only the bootstrap candidate.
- **A hand install as the fallback**: the operator never opens these apps outside the Fleet, so a manual path means no updates.

## Decision Record impact

`aligned-with neomjs/neo-agent-institution#adr:0034-electron-shell-architecture`, through #1531's amendment of §2.5. ADR-0019: one new declarative leaf, `fleet.harnessGenerations.checkIntervalHours`, read at its use site.

## Related

#1531 (parent) · #1537 · #1539 · #1543 · #1584 · neomjs/neo-agent-institution#732 (the Fleet Manager's own updates)

Live latest-open + closed sweep: checked the latest 20 open plus an open and closed search (`harness acquisition schedule`, `checks for updates`, `fetches harness`, `acquisition trigger`) in Brain, and Institution's open issues (`harness update OR vendor update check`), at 2026-11-11T00:12Z. No equivalent: Institution #732 covers the Fleet Manager's own updates, #682 is the Claude outcome #1584 delivers, and the Out of Scope of #1537 and #1539 was read. Backlog band: 161 open (red); this leaf completes #1531 point 4. A2A claim sweep: none. Own-assignment sweep: #1531, #1584, #571, #133 and #1508 do not cover it. Memory Core sweep: no prior art for a trigger. Structure map: `ai/services/fleet`, sibling `wireFleetOpenWorkSource.mjs`.

Origin Session ID: 3ac6b9a1-a988-4aed-86f0-c8640799f150

Retrieval Hint: `query_raw_memories("harness acquisition trigger no caller frozen updates fleet seats only")`

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-11T00:13:23Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-11T00:13:25Z @neo-opus-ada added the `enhancement` label
- 2026-11-11T00:13:25Z @neo-opus-ada added the `ai` label
- 2026-11-11T00:13:25Z @neo-opus-ada added the `architecture` label
- 2026-11-11T00:13:25Z @neo-opus-ada added the `agent-os` label
- 2026-11-11T00:13:35Z @neo-opus-ada cross-referenced by #1531
- 2026-11-11T00:41:46Z @neo-opus-ada cross-referenced by PR #1588
- 2026-11-11T01:13:41Z @neo-opus-ada cross-referenced by #1590
- 2026-11-11T02:08:05Z @neo-opus-ada referenced in commit `9adb2e0` - "fix(fleet): bound acquisition timers, feed from selected build (#1587)

An interval or boot delay beyond Node's 32-bit timer range runs at the longest delay Node keeps instead of 1 ms, and a non-finite one wires nothing. A failed boot stops the acquisition job with the other observers. Antigravity reads its feed from the selected generation, as Codex does, so a stale or missing installed bundle no longer names it."
- 2026-11-11T02:28:51Z @neo-gemini-maryam cross-referenced by PR #1591
- 2026-11-11T02:30:52Z @neo-opus-ada referenced in commit `0646e98` - "fix(fleet): state the timer cap as a literal, not an exponent (#1587)

The retry-bound lint reads any exponent as a retry-growth candidate; the cap is a constant, not growth."
- 2026-11-11T02:58:35Z @tobiu referenced in commit `90c00a1` - "feat(fleet): the Fleet fetches harness builds on its own (#1587) (#1588)

* feat(fleet): the Fleet fetches harness builds on its own (#1587)

wireFleetHarnessAcquisition runs each opted-in family's acquisition (Codex: Sparkle appcast; Antigravity: electron-updater manifest) two minutes after devFleetServer boots, then every fleet.harnessGenerations.checkIntervalHours (a new leaf, default 6). Runs never overlap per family, a failure is recorded per family, and a seat still adopts a new build only at its next Start. FleetLifecycleService.harnessAcquisitionContext hands the job the same opt-in, store root and installed bundle a Start reads.

Shutdown stops the timers without awaiting a download in flight. Antigravity now sweeps a killed run's staging at the start of its next run, as Codex already did.

* fix(fleet): bound acquisition timers, feed from selected build (#1587)

An interval or boot delay beyond Node's 32-bit timer range runs at the longest delay Node keeps instead of 1 ms, and a non-finite one wires nothing. A failed boot stops the acquisition job with the other observers. Antigravity reads its feed from the selected generation, as Codex does, so a stale or missing installed bundle no longer names it.

* fix(fleet): state the timer cap as a literal, not an exponent (#1587)

The retry-bound lint reads any exponent as a retry-growth candidate; the cap is a constant, not growth."
- 2026-11-11T02:58:37Z @tobiu closed this issue

