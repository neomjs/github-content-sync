---
id: 132
title: 'Skills #131''s merge published nothing: its 0.1.23 collided with #127''s'
state: CLOSED
labels:
  - bug
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T21:07:03Z'
updatedAt: '2026-09-30T21:44:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/132'
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
closedAt: '2026-09-30T21:44:30Z'
---
# Skills #131's merge published nothing: its 0.1.23 collided with #127's

## Context

#127 and #131 were both approved at 0.1.23 and merged 24 s apart (20:53:08Z and 20:53:32Z, 2026-09-30).
- #127's publish succeeded: 0.1.23 is `latest`, `v0.1.23` is tagged, and Emmy verified the `./agents-md` export (#100 issuecomment-5919711206).
- #131's publish failed (run 36775783489): `npm error 409 Conflict … Cannot publish over previously staged version "0.1.23"`.

**Sweep attestations (2026-09-30T21:06Z):**
- Live latest-open: the latest 20 open issues here, no equivalent.
- A2A: @neo-gpt-emmy routed the collision to me (21:06Z). No other claim.
- MC sweep ("two pull requests same version both merged second publish failed E409"): 5 results. The one decision on record is mine from 09-24: "the per-PR version bump is per publish, not per PR". Nothing argues against a follow-up bump.
- Own-assignment: 1 open (#76), not overlapping.

## The Problem

#131's skill edits are on `dev` but in no published version: memory-mining's rule without `AGENTS_STARTUP.md §3.3`, pr-review's trigger without it, create-skill's substrate list, and session-sunset's context-recovery boot. Consumers install the published package, so every consumer still materializes the old text. neomjs/neo#19337, which deletes `AGENTS_STARTUP.md` from the Engine, waits on this.

The order note on #131 (issuecomment-5919055685) asked for #127 first and a re-bump before #131 merged. Prose is not a gate. The version check runs only on PR events, so after #127 landed, #131's green check still reflected the old base.

## The Fix

Bump to 0.1.24. The merge publishes `dev` as it stands, which carries #131's edits.

Prevention is a repository setting and therefore the operator's call: "require branches to be up to date before merging" re-runs `check-version-bump` on the second PR once the first lands, which turns this collision red before merge. No AC depends on that choice.

## Acceptance Criteria

- [ ] AC-1: `package.json` and `package-lock.json` read 0.1.24, and `check-version-bump` passes against `dev` (0.1.23).
- [ ] AC-2 *(post-merge)*: the merge's publish run succeeds; npm `latest` is 0.1.24 with tag `v0.1.24`; and the published `memory-mining-protocol.md` has no `AGENTS_STARTUP`.

## Out of Scope

- The repository setting above.
- The Engine's own dependency bump, which neomjs/neo#19337 carries.

## Related

- #127, #131, #128 (the edits this publishes), #56 (the publish pipeline)
- neomjs/neo#19337

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636

## Timeline

- 2026-09-30T21:07:03Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T21:07:04Z @neo-opus-grace added the `bug` label
- 2026-09-30T21:07:04Z @neo-opus-grace added the `ai` label
- 2026-09-30T21:07:04Z @neo-opus-grace added the `build` label
- 2026-09-30T21:07:50Z @neo-opus-grace cross-referenced by PR #133
- 2026-09-30T21:44:30Z @tobiu referenced in commit `8b25a84` - "Merge pull request #133 from neomjs/grace/release-0.1.24

chore(release): bump to 0.1.24, so the skill edits #131 merged are published (#132)"
- 2026-09-30T21:44:30Z @tobiu closed this issue

