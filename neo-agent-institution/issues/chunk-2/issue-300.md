---
id: 300
title: 'Engine pin carrying neo #19312, so rail tooltips stop sticking'
state: CLOSED
labels:
  - ai
  - dependencies
assignees:
  - neo-opus-grace
createdAt: '2026-09-27T14:57:34Z'
updatedAt: '2026-09-27T15:30:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/300'
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
closedAt: '2026-09-27T15:30:09Z'
---
# Engine pin carrying neo #19312, so rail tooltips stop sticking

## Context
neomjs/neo#19312 merged the fix for tooltips that stick: a target left inside the show delay showed its tooltip a moment later, and it stayed. In the FM this is the keeper rail. A quick pass over its icons leaves a label such as "Observatory" hanging at the rail's edge, and the Institution's visual arms dwell and leave to keep it out of their shots. The Institution pins the engine at `942b43c8b8`, which predates the fix, so the FM does not have it yet.

## The Problem
Until the pin moves, the installed FM keeps the glitch, however long the engine has had the fix.

## The Architectural Reality
- The engine is a git dependency pinned by commit in `package.json` and `package-lock.json`, as #277 did last.
- `942b43c8b8..c300181b4d` on neo `dev` is #19309 (tests), #19310 (tests) and #19312 (the fix), so the one runtime change is the tooltip's.
- The visual stamp's engine axis (`package-lock.json` → `neo.mjs`) moves with the pin, so the stamp is re-taken in the same PR.
- The Brain pin stays. Moving it to `dev` 742de62 (Brain #582's System view) first needs the harness to follow Brain #573's `fleet.instanceRoot` → `fleet.agentsRoot` rename (`harness/brain.mjs:348`, `:537`), which is the Brain bump's own work.

## The Fix
Move the engine pin to `c300181b4d9254e5eb51bd80b3f4502a1787a668` and re-take the stamp.

## Acceptance Criteria
- [ ] AC-1: `package.json` and `package-lock.json` name a neo `dev` commit that contains the merge of neomjs/neo#19312.
- [ ] AC-2: The unit and visual suites are green on the new pin, and `check-visual-baselines` matches a stamp taken on it.

## Out of Scope
- The Brain pin and the harness rename it needs.
- Removing the visual arms' dwell-and-leave workarounds, which can go once this pin is on dev.

## Related
- #277 (the last engine pin), neomjs/neo#19311, #10.

Live latest-open sweep: checked the latest 20 open issues at 14:57Z; no pin ticket is open. A2A: my own `[lane-claim · Institution pins]` at 14:49Z; no other claim. Own assignments: none on this surface.

Origin Session ID: 0dc6daad-2744-44c9-91cb-38d82e9e82e6


## Timeline

- 2026-09-27T14:57:35Z @neo-opus-grace added the `ai` label
- 2026-09-27T14:57:35Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-27T14:57:35Z @neo-opus-grace added the `dependencies` label
- 2026-09-27T14:59:00Z @neo-opus-grace cross-referenced by PR #301
- 2026-09-27T15:23:13Z @neo-opus-ada cross-referenced by #302
- 2026-09-27T15:30:09Z @tobiu referenced in commit `1325460` - "Merge pull request #301 from neomjs/grace/300-engine-pin

chore(deps): engine pin → dev@c300181b4d, which carries the tooltip that no longer sticks (#300)"
- 2026-09-27T15:30:09Z @tobiu closed this issue

