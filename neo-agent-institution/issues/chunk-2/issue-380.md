---
id: 380
title: 'Brain pin 5: the installed FM carries seat survival and seat instructions'
state: OPEN
labels:
  - agent-os
  - ai
  - dependencies
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T11:39:54Z'
updatedAt: '2026-10-01T11:39:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/380'
author: neo-opus-grace
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
# Brain pin 5: the installed FM carries seat survival and seat instructions

## Context

The operator's priority of 2026-10-01 (Clio's broadcast, 11:05Z) is that the Fleet Manager works for Ada before the setup wizard. Its path runs: neomjs/neo-agent-brain#669 → #378's seat-root change plus a Brain-pin bump → repackage and install → Ada's seat moves once. The installed app reaches Brain work only through the Brain pin (`package.json`: `9204d8b865`).

## The Problem

Brain `dev` now carries what the next installed FM needs, and the pin carries none of it:

- **Seat survival** (neomjs/neo-agent-brain#662): an FM quit no longer ends app-bundle seats, and a restarted Fleet server re-adopts them.
- **Seat instructions** (neomjs/neo-agent-brain#654): a Fleet seat starts with its repository's maintainer instructions.
- **The write guard without a roster entry** (neomjs/neo-agent-brain#665): unblocks Sophie's public writes.
- **The roster's model family by harness** (neomjs/neo-agent-brain#666).
- **neomjs/neo-agent-brain#669**, once merged.

The Brain now declares `neo-agent-skills` `^0.1.23`, the first release exporting `./agents-md`. The Institution declares `^0.1.14`, and its lock resolves `0.1.19`.

## The Architectural Reality

The pin moves in three places together, as Brain pin 4 did (#269 / PR #270): `package.json`, `package-lock.json`, and the CI Brain checkout in `.github/workflows/ci.yml`. The Fleet contract allowlist in the content policy follows any contract module the new Brain adds.

## The Fix

- Move the Brain pin to Brain `dev` once neomjs/neo-agent-brain#669 has merged, so one repackage carries everything.
- Raise `neo-agent-skills` to `^0.1.24`, so a single hoisted copy satisfies both declarations.
- Check the contract allowlist.

## Acceptance Criteria

- [ ] AC-1: `package.json`, `package-lock.json` and `ci.yml` name the same Brain commit, and it contains neomjs/neo-agent-brain#662, #654, #665 and, once merged, #669. `node_modules/neo-agent-skills` is `0.1.24` or later and exports `./agents-md`.
- [ ] AC-2: The Institution's unit suite is green on the new pin. A new Fleet contract module, if any, is admitted by the content policy.
- [ ] AC-3 `[L4-deferred — operator handoff needed]` (post-merge, installed): the repackaged app boots on the new pin. Owner after merge: #7.

## Out of Scope

- The engine pin (unchanged).
- The repackage and install themselves (Emmy, #7).
- The seat moves (neomjs/neo-agent-brain#571).

## Related

#269 / PR #270 (Brain pin 4) · #378 · #7 · neomjs/neo-agent-brain#571 · neomjs/neo-agent-brain#669

Live latest-open sweep: the latest open Institution issues at 2026-10-01T11:39Z, plus searches for a Brain pin: no open pin ticket. A2A: Clio's critical path names the Brain-pin bump beside my #378. Own assignments: #378 (seat root), a separate leaf.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364

## Timeline

- 2026-10-01T11:39:54Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T11:39:56Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T11:39:56Z @neo-opus-grace added the `ai` label
- 2026-10-01T11:39:56Z @neo-opus-grace added the `dependencies` label
- 2026-10-01T11:42:46Z @neo-opus-grace cross-referenced by PR #381
- 2026-10-01T13:04:05Z @neo-fable cross-referenced by PR #383

