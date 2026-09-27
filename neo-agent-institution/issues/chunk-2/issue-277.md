---
id: 277
title: 'Engine pin carrying neo #19306 and #19308, with their two NL receipts'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - dependencies
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-09-27T08:31:26Z'
updatedAt: '2026-09-27T10:42:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/277'
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
closedAt: '2026-09-27T10:42:40Z'
---
# Engine pin carrying neo #19306 and #19308, with their two NL receipts

## Context
Two engine PRs each leave one Institution-side acceptance criterion, and neither can run until an Institution engine pin carries the merged fix:
- neomjs/neo#19305 AC-3 (PR neomjs/neo#19306): `FleetObservatoryNL`'s `selectNode` clicks without resting the pointer first, and the click still selects.
- neomjs/neo#19307 AC-3 (PR neomjs/neo#19308): `FleetPermanenceMatrixRow4NL` passes.

Their named residual owners close first: PR #273 resolves #258, and PR #276 resolves #274. Euclid's review 5329489900 on neomjs/neo#19306 (Required Action 2) found this for the first; the second has the same shape. This leaf owns both receipts until they run.

## The Problem
A receipt whose owner closes before it can run is lost. Nothing on the open queue schedules the pin move that makes these two runnable.

## The Architectural Reality
- The Institution consumes the engine as a git dependency pinned by commit in `package.json` and `package-lock.json` (`a50ae57ce8` since #270).
- `test/playwright/e2e/agentos/FleetObservatoryNL.spec.mjs` (`selectNode`, around line 80) rests the pointer on the node before clicking, because the main thread sent a press ahead of the coalesced move (neomjs/neo#19305).
- `FleetPermanenceMatrixRow4NL` fails because the engine refused the first `savePerspective({activate: false})` into an empty library (neomjs/neo#19307).
- The visual stamp's engine axis (`buildScripts/checkVisualBaselines.mjs`: `package-lock.json` → `neo.mjs`) moves with any engine pin, so the stamp is re-taken in the same PR (the `#270` CI red on 2026-09-27).

## The Fix
1. Move the engine pin in `package.json` and `package-lock.json` to a neo `dev` commit that contains both merges.
2. Remove the pointer rest from `selectNode`, and the comment that explains it.
3. Re-take the visual stamp on the new pin.

## Acceptance Criteria
- [ ] AC-1: `package.json` and `package-lock.json` name a neo `dev` commit that contains the merges of neomjs/neo#19306 and neomjs/neo#19308.
- [ ] AC-2: `FleetObservatoryNL`'s `selectNode` clicks without resting the pointer, and the click selects (discharges neomjs/neo#19305 AC-3).
- [ ] AC-3: `FleetPermanenceMatrixRow4NL` passes on the new pin (discharges neomjs/neo#19307 AC-3).
- [ ] AC-4: `check-visual-baselines` matches a stamp taken on the new pin, and the unit and visual suites are green.

## Out of Scope
- The other local NL reds: the Liveness journey (#265's area) and the N-window race (#275).
- Any Brain pin move.

## Related
neomjs/neo#19305, neomjs/neo#19306, neomjs/neo#19307, neomjs/neo#19308, #258, #273, #274, #276, #270

Starts when both engine PRs have merged.

Live latest-open sweep: latest 20 open issues read at 2026-09-27T08:27Z and the newest 5 again at 08:31Z; no equivalent. A2A claim sweep (30 newest, all read states): no claim on an engine pin or these receipts. Own-assignment sweep: my open #275, #271, #269, #267, #264, #258 and #11; none owns the next engine pin. Memory Core rationale sweep: `query_raw_memories` and `query_summaries` timed out three times between 08:28Z and 08:31Z (the plane was recreated at 08:21Z); the premise rests on the live review and PR bodies cited above.

Retrieval Hint: "engine pin NL receipt selectNode pointer rest PermanenceMatrixRow4"
Origin Session ID: 0dc6daad-2744-44c9-91cb-38d82e9e82e6

Authored by Grace (Claude Opus 5.5, Claude Code).

## Timeline

- 2026-09-27T08:31:26Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-27T08:31:28Z @neo-opus-grace added the `enhancement` label
- 2026-09-27T08:31:28Z @neo-opus-grace added the `agent-os` label
- 2026-09-27T08:31:28Z @neo-opus-grace added the `ai` label
- 2026-09-27T08:31:29Z @neo-opus-grace added the `dependencies` label
- 2026-09-27T08:31:29Z @neo-opus-grace added the `testing` label
- 2026-09-27T08:32:29Z @neo-opus-grace cross-referenced by PR #19306
- 2026-09-27T08:32:31Z @neo-opus-grace cross-referenced by PR #19308
- 2026-09-27T08:32:32Z @neo-opus-grace cross-referenced by #19305
- 2026-09-27T08:32:33Z @neo-opus-grace cross-referenced by #19307
- 2026-09-27T08:46:46Z @neo-opus-grace cross-referenced by #278
- 2026-09-27T10:13:48Z @tobiu referenced in commit `93f69fa` - "chore(deps): engine pin → dev@942b43c8b8, which carries the move-before-press and first-save fixes (#277)"
- 2026-09-27T10:13:48Z @tobiu referenced in commit `283ee43` - "test(agentos): the Observatory NL clicks a node without resting the pointer first (#277)"
- 2026-09-27T10:13:50Z @neo-opus-grace cross-referenced by PR #282
- 2026-09-27T10:42:40Z @tobiu referenced in commit `cbedd42` - "Merge pull request #282 from neomjs/grace/277-engine-pin

chore(deps): engine pin → dev@942b43c8b8, and the Observatory NL clicks without a pointer rest (#277)"
- 2026-09-27T10:42:41Z @tobiu closed this issue

