---
id: 826
title: The Fleet reports where a desktop seat's first session opened
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T19:12:54Z'
updatedAt: '2026-10-04T11:33:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/826'
author: neo-opus-ada
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 522 A desktop seat whose session opened in another folder says so'
closedAt: '2026-10-04T11:33:24Z'
---
# The Fleet reports where a desktop seat's first session opened

## Context

This is gap 3 of #571, accepted as one leaf ([disposition 5972558630](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5972558630)).

**Amended at intake (2026-10-03, author):** the observation's source moved from the host's shared transcripts to the seat's own Desktop profile. The shared transcripts cannot attribute a `wrong` folder to the seat; the profile can. See The Architectural Reality.

## The Problem

The pilot's single cause was the session's folder (#571, 5968933902). Claude Desktop cannot be launched into a folder (#669 AC-5). Everything Fleet projects (MCP rows, memory pin, hooks) keys on the managed clone, so a session opened anywhere else loads none of it. "Process ready" and "prompt delivered" both read green while the seat is unusable. The seat itself cannot refuse, because nothing it would refuse with has loaded.

## The Architectural Reality

- A provisioned Start launches the seat with `cwd` set to the managed clone (`startAgentProvisioned`, `targetRepoRoot`). The lifecycle record keeps that `cwd`, the seat's `instanceHome` and `startedAt`.
- The seat's Desktop profile, `<instanceHome>/claude-code-sessions/<account>/<org>/local_<id>.json`, records each Code-tab session. Each record carries the folder it opened in (`originCwd`, `cwd`), `createdAt`, `lastActivityAt`, `lastFocusedAt` and `isArchived`.
  - The profile is the seat's alone, so its records are attributable.
  - The host's shared `~/.claude/projects/<slug>/` transcripts are not: every session on the machine writes there, so a transcript outside the clone's slug cannot be tied to the seat.
- Live instance, measured read-only on 2026-10-03: the `neo-fable` Fleet seat's records report `originCwd: /Users/Shared/fable/neomjs/neo`, her pre-move clone, not the managed one.
- Desktop reopens a profile's earlier sessions on relaunch, so "since the launch" means active since `startedAt`, not created since then.
- A worktree session moves `cwd` while `originCwd` keeps the folder the session opened in.
- The record format is Claude Desktop's own and undocumented.
- The seat status crosses the wire as `fleetRuntimeStatus`. The cockpit roster row joins runtime facts in `fleetCockpitStatus` (as `repoOutcomes` does).

## The Fix

For a `claude-desktop` seat with a managed clone, the lifecycle status reads the seat's own session records. It takes the record most recently active since `startedAt`, skipping archived ones:
- opened in the managed clone (`originCwd`, else `cwd`) → `ok`;
- opened anywhere else → `wrong`, with the observed folder;
- no record active since the launch → `pending`;
- records present but unreadable → `unknown`, with the reason. Never `ok`.

The result rides the runtime row and the cockpit roster row as `sessionFolder: {state, expected, observed?, reason?}`. Other families carry none.

## Contract Ledger

*(Claimer-authored, added 2026-10-04 for RA-2 of Sophie's review on PR #828. The rows name the surfaces at `3c313493`.)*

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `readSeatSessionFolder({instanceHome, expected, since, fileSystem})` (`ai/services/fleet/seatSessionFolder.mjs`) | the seat's own Desktop profile, `<instanceHome>/claude-code-sessions/<account>/<org>/local_*.json` (Desktop's undocumented format) | `{state, expected, observed?, reason?}`, `state ∈ {pending, ok, wrong, unknown}`. The record most recently active since `since` decides; archived records are skipped; the folder is `originCwd`, else `cwd`. `observed` only with `wrong`, `reason` only with `unknown` | no session store → `pending`. Any other listing failure → `unknown` with the reason. A record that is unreadable, vanished after the listing, or refuses its metadata counts as unreadable → `unknown` with the reason when no readable record decides. Never `ok` from a failed read; never throws | module + function JSDoc | `seatSessionFolder.spec.mjs`, including `a record whose metadata read fails with ENOENT\|EACCES after the listing is unreadable, never a throw` |
| `FleetLifecycleService.status(id).sessionFolder` (`sessionFolderFor`) | the lifecycle record: `harnessType`, `state`, `instanceHome`, `cwd`, `startedAt` | the reader's result for a running `claude-desktop` seat with a checkout, a profile and a start time | always present: `null` for another family, a seat not running, a launch missing any of the three, or no record | `status` + `sessionFolderFor` JSDoc | `FleetLifecycleService.spec.mjs` › `only a running claude-desktop seat with a checkout, a profile and a start carries one` |
| `FleetManager.fleetRuntimeStatus`, the runtime row's `sessionFolder` | `status().sessionFolder` | copied onto the row when non-null | **omitted** from the row when `null` | method JSDoc | `FleetManager.spec.mjs` › `where a desktop seat's session opened rides its runtime row, and a status without one adds nothing` |
| `fleetCockpitStatus`, the roster row's `sessionFolder` | the runtime row | joined onto the roster row | **always present**: the object, or `null` when the runtime row omits it or there is no runtime row | module JSDoc | `fleetCockpitStatus.spec.mjs` › `carries where a desktop seat's session opened from the runtime row — null for every other row` |
| Consumer: Institution `FleetAgent.sessionFolder` (neomjs/neo-agent-institution#522) | the roster row | renders `wrong`, `pending` and `unknown` per #522's ACs; `ok` adds no line | `null` → nothing rendered | #522 | #522's ACs |

## Acceptance Criteria

- [ ] AC-1: a `claude-desktop` seat's status carries `sessionFolder.state ∈ {pending, ok, wrong, unknown}`, its `expected` folder, and the `observed` folder when `wrong` (unit, fixture records).
- [ ] AC-2: `ok` comes only from a record in the seat's own profile, active since the launch and opened in the managed clone. It never comes from the process cwd, readiness, or a reopened session older than the launch (unit).
- [ ] AC-3: an unreadable record set reads `unknown` with its reason (unit).

## Out of Scope

Opening the folder for the operator: Desktop cannot be launched into one. The seats root's visibility is the operator's decision, recorded on #571.

The card's line, `session opened in <folder> — expected <path>` with one next action, is neomjs/neo-agent-institution#522 (blocked by this ticket): a PR resolves one ticket.

## Related

Parent: #571. Evidence: #571 5968933902, 5968938045. #669 (no folder launch).

Sweeps: live latest-open Brain, 12 at 2026-10-03T19:12:13Z, plus a search for "session opened in": no equivalent.

Decision Record impact: `none`.

Origin Session ID: 84371353-afea-4f59-9b58-2b8777325f56
Retrieval Hint: "desktop seat session folder claude-code-sessions originCwd managed clone wrong folder state card"



## Timeline

- 2026-10-03T19:12:54Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-03T19:12:55Z @neo-opus-ada added the `enhancement` label
- 2026-10-03T19:12:55Z @neo-opus-ada added the `ai` label
- 2026-10-03T19:12:55Z @neo-opus-ada added the `agent-os` label
- 2026-10-03T19:33:31Z @neo-opus-ada cross-referenced by #571
- 2026-10-03T19:46:31Z @neo-opus-ada cross-referenced by #522
- 2026-10-03T19:53:48Z @neo-opus-ada cross-referenced by PR #828
- 2026-10-03T19:55:53Z @neo-opus-ada referenced in commit `cffff51` - "docs(fleet): the session-folder reader's module doc states the Desktop constraint without a ticket reference (#826)"
- 2026-10-04T11:06:34Z @neo-opus-ada referenced in commit `3c31349` - "fix(fleet): a session record that vanishes or refuses its metadata after the listing reads as unreadable, never a throw that rejects every seat's status (#826)

The statSync on each listed record ran outside the per-record catch, so a record Desktop removed after the listing (ENOENT), or whose metadata refused (EACCES), threw through FleetLifecycleService.status into FleetManager.fleetRuntimeStatus and rejected the whole runtime response. The stat now shares the record read's catch: such a record counts as unreadable and the answer is unknown with the reason. The reason drops 'since the launch', because a record whose metadata could not be read has no known time."
- 2026-10-04T11:33:24Z @tobiu referenced in commit `6e1185a` - "feat(fleet): a desktop seat's status says where its session opened (#826) (#828)

* feat(fleet): a desktop seat's status says where its session opened, read from the seat's own profile (#826)

readSeatSessionFolder reads the seat's Desktop profile, claude-code-sessions/
**/local_*.json, and compares the session most recently active since the
launch with the checkout the seat was launched in:
- opened in the checkout (originCwd, else cwd) is ok;
- opened anywhere else is wrong, naming the folder;
- no session since the launch is pending;
- records it cannot read are unknown with the reason, never ok.

A running claude-desktop seat's lifecycle status carries it as sessionFolder;
fleetRuntimeStatus and the cockpit roster row pass it through, as they do
the repository outcomes. The host's shared ~/.claude/projects transcripts are
not read: every session on the machine writes there, so they cannot say
whose a session is.

* docs(fleet): the session-folder reader's module doc states the Desktop constraint without a ticket reference (#826)

* fix(fleet): a session record that vanishes or refuses its metadata after the listing reads as unreadable, never a throw that rejects every seat's status (#826)

The statSync on each listed record ran outside the per-record catch, so a record Desktop removed after the listing (ENOENT), or whose metadata refused (EACCES), threw through FleetLifecycleService.status into FleetManager.fleetRuntimeStatus and rejected the whole runtime response. The stat now shares the record read's catch: such a record counts as unreadable and the answer is unknown with the reason. The reason drops 'since the launch', because a record whose metadata could not be read has no known time."
- 2026-10-04T11:33:24Z @tobiu closed this issue
- 2026-10-04T12:52:07Z @neo-opus-ada cross-referenced by #700
- 2026-10-04T14:07:32Z @neo-opus-grace cross-referenced by PR #546

