---
id: 294
title: Prior-art sweeps miss the tickets the touched code names
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-06T11:59:48Z'
updatedAt: '2026-11-06T16:00:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/294'
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
closedAt: '2026-11-06T16:00:43Z'
---
# Prior-art sweeps miss the tickets the touched code names

## Context

@tobiu, 2026-11-06, on neomjs/neo#20410: "graduating the discussion should probably have closed the ticket which euclid picked up. also meaning: creating the discussion should have found the ticket as prior art."

Measured on neomjs/neo:
- neomjs/neo#17844 ("Run the dock NL whitebox battery against the external Brain runtime") was created 2026-08-29.
- neomjs/neo D#18775 ("ai.Client fuses a service registry with a socket…") was created 2026-09-16, resolved 2026-10-12 and closed 2026-11-05. Its body and its 8 comments mention neomjs/neo#17844 0 times, and the ticket's 96 comments mention the Discussion 0 times.
- The Discussion's direction (no bridge, no Brain checkout) supersedes the ticket's.
- On 2026-10-12, one peer reviewed the Discussion at 13:12 and accepted the ticket's full-cycle obligation at 17:36.
- That obligation was built as neomjs/neo#20410 the night after the Discussion closed, and is now being reverted (neomjs/neo#20450).

The one mechanical link was in the code. `buildScripts/util/e2eCiSelection.mjs` carried `tracker: '#17844'` on the exclusion for the exact specs the Discussion was about.

## The Problem

`ideation-sandbox/audits/pre-authoring-adjacency-sweep.md` runs four sweeps, all of them keyed on words: live, semantic, exact and memory. Graduation's same-turn reconciliation reuses those results. A proposal framed in different nouns from its predecessor (client/socket versus dock battery/external Brain) passes every sweep and then supersedes the predecessor without anyone noticing.

## The Fix

Add one bullet to the sweep:

> **Code sweep:** `git grep` the files the proposal would change for ticket references (`tracker:` fields, `#N` in comments and selection lists). Every open ticket they name is prior art, whatever words its title uses.

Graduation reconciliation already reuses adjacency results, so it inherits the bullet with no second edit. The package version is bumped.

## Acceptance Criteria

- [ ] **AC-1:** the adjacency sweep carries the code-sweep bullet.
- [ ] **AC-2:** `npm run lint`, `lint --base origin/dev` and `npm test` pass, and the version is bumped.

## Out of Scope

- Rewriting the reconciliation rule. It exists (#192); the gap is discovery.
- Retroactive reconciliation of other Discussions.

## Related

#192, #258, neomjs/neo#17844, neomjs/neo#20410, neomjs/neo#20450, neomjs/neo#20451.

Live sweep, 2026-11-06 ~12:00Z: latest open issues, plus a search for reconciliation, prior art and code tracker. #192 (closed) owns the reconciliation rule and #258 dead-trail closure; neither covers code-anchored discovery.

Origin Session ID: a33fc3c4-7e98-4300-985d-25a10e216b50

🖖 Grace (Opus 5.5, Claude Code)

## Timeline

- 2026-11-06T11:59:48Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-06T11:59:49Z @neo-opus-grace added the `enhancement` label
- 2026-11-06T11:59:50Z @neo-opus-grace added the `ai` label
- 2026-11-06T11:59:50Z @neo-opus-grace added the `model-experience` label
- 2026-11-06T11:59:58Z @github-actions added the `needs-design-authority` label
- 2026-11-06T12:01:03Z @neo-opus-grace cross-referenced by PR #295
- 2026-11-06T12:14:14Z @neo-opus-grace cross-referenced by #296
- 2026-11-06T12:15:58Z @neo-opus-grace cross-referenced by PR #297
- 2026-11-06T15:31:21Z @neo-opus-grace referenced in commit `aadeaec` - "chore(release): 0.1.93, in the merge order of the open skills PRs (#294)"
- 2026-11-06T15:48:58Z @neo-opus-grace referenced in commit `0ad44ae` - "chore(release): merge dev after #293, keeping 0.1.93 over its 0.1.92 (#294)"
- 2026-11-06T16:00:43Z @tobiu referenced in commit `6bda158` - "feat(ideation-sandbox): the prior-art sweep reads the tickets the touched code names (#294) (#295)

* feat(ideation-sandbox): the prior-art sweep reads the tickets the touched code names (#294)

* chore(release): 0.1.93, in the merge order of the open skills PRs (#294)"
- 2026-11-06T16:00:44Z @tobiu closed this issue

