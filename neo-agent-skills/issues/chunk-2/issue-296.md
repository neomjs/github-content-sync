---
id: 296
title: The ticket-create sweep reports the repository's backlog band
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-06T12:14:13Z'
updatedAt: '2026-11-06T16:19:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/296'
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
closedAt: '2026-11-06T16:19:05Z'
---
# The ticket-create sweep reports the repository's backlog band

## Context

@tobiu, 2026-11-06, verbatim: "ideally, <25 open tickets should be the green range, 2-50 orange, 50+ red, so that we never have too much prior art, which significantly reduces the work to even check for it." And: "of course new tickets can get created as needed, but peers need to get the awareness to keep backlogs reasonable. the band ranges can get challenged. e.g. < 10 is not feasible, since one new epic could already be more." "2-50" is read as 25–50.

Measured the same day:

| Repo | Open | Band |
|---|---|---|
| neomjs/neo | 99 | red (down from about 250) |
| neomjs/neo-agent-brain | 165 | red |
| this repo | 24 | green |

The cost case is neomjs/neo#17844. A 96-comment ticket kept an obligation open beside the Discussion (neomjs/neo D#18775) that superseded it. The predecessor was built (neomjs/neo#20410) and is now being reverted.

## The Problem

Nothing tells the person filing a ticket how loaded the repository's prior art already is. §1a lists the latest 20 open issues but never their total.

## The Fix

In `ticket-create-workflow.md` §1a, next to the attestation line, add one **Backlog band** step:
- Count the repository's open issues: under 25 is green, 25–50 orange, over 50 red.
- In orange or red, fold the work into an owning ticket or epic before filing; file only where none owns it.
- Record the count and band on the attestation line.

The package version is bumped.

## Acceptance Criteria

- [ ] **AC-1:** §1a carries the band step, with the count command, the three bands, fold-before-filing in orange or red, and the attestation.
- [ ] **AC-2:** `npm run lint`, `lint --base origin/dev` (growth justified) and `npm test` pass, and the version is bumped.

## Out of Scope

- A lint or dashboard that computes the band.
- Closing existing tickets: neomjs/neo#15000 and neomjs/neo-agent-brain#1438 run the premise reads.

## Related

#294 (the code-anchored prior-art sweep), neomjs/neo#17844, neomjs/neo#20450.

Live sweep, 2026-11-06 ~12:15Z: latest open issues, plus a search for "backlog band / open ticket count". #269 (closed) maps old tickets at intake and does not cover this.

Origin Session ID: a33fc3c4-7e98-4300-985d-25a10e216b50

🖖 Grace (Opus 5.5, Claude Code)

## Timeline

- 2026-11-06T12:14:13Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-06T12:14:15Z @neo-opus-grace added the `enhancement` label
- 2026-11-06T12:14:15Z @neo-opus-grace added the `ai` label
- 2026-11-06T12:14:15Z @neo-opus-grace added the `model-experience` label
- 2026-11-06T12:14:24Z @github-actions added the `needs-design-authority` label
- 2026-11-06T12:15:58Z @neo-opus-grace cross-referenced by PR #297
- 2026-11-06T15:31:23Z @neo-opus-grace referenced in commit `54c4164` - "fix(ticket-create): in orange or red, file the needed ticket when no owner exists; release 0.1.94 (#296)"
- 2026-11-06T16:01:47Z @neo-opus-grace referenced in commit `713c56c` - "chore(release): merge dev after #295, keeping 0.1.94 over its 0.1.93 (#296)"
- 2026-11-06T16:19:05Z @tobiu referenced in commit `69782b6` - "feat(ticket-create): the sweep reports the repository's backlog band (#296) (#297)

* feat(ticket-create): the sweep reports the repository's backlog band (#296)

The operator's bands: under 25 open issues green, 25-50 orange, over 50
red. Every open ticket is prior art each later sweep must read, so the
§1a sweep now counts them, and in orange or red the author folds the
work into an owning ticket or epic before filing.

[skill-growth-justified: an operator-directed band with no existing home in the sweep it governs]

* fix(ticket-create): in orange or red, file the needed ticket when no owner exists; release 0.1.94 (#296)"
- 2026-11-06T16:19:07Z @tobiu closed this issue

