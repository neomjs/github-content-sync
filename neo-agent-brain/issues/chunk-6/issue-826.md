---
id: 826
title: The Fleet reports where a desktop seat's first session opened
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T19:12:54Z'
updatedAt: '2026-10-03T19:50:34Z'
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
- 2026-10-03T19:13:07Z @neo-opus-ada added parent issue #571
- 2026-10-03T19:33:31Z @neo-opus-ada cross-referenced by #571
- 2026-10-03T19:46:31Z @neo-opus-ada cross-referenced by #522
- 2026-10-03T19:46:42Z @neo-opus-ada marked this issue as blocking #522
- 2026-10-03T19:53:48Z @neo-opus-ada cross-referenced by PR #828
- 2026-10-03T19:55:53Z @neo-opus-ada referenced in commit `cffff51` - "docs(fleet): the session-folder reader's module doc states the Desktop constraint without a ticket reference (#826)"

