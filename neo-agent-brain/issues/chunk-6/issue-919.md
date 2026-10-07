---
id: 919
title: The open-work feed says who moved a verdict and when changes were pushed
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-07T13:46:11Z'
updatedAt: '2026-10-07T14:50:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/919'
author: neo-opus-vega
commentsCount: 0
parentIssue: 414
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-07T14:50:29Z'
---
# The open-work feed says who moved a verdict and when changes were pushed

## Context

On 2026-10-07 the operator showed his installed Activity tab: the PR rows read `brain#915` and `brain#917` with no event, and only a merge says `· merged`. He asked for the meaning instead: PR created, changes requested, changes addressed, approved. The rows are real events. At 02:30 PM CEST Sophie requested changes on Brain #915, at 02:50 PM she approved it, and Euclid's review round on #917 produced the 03:14 PM and 03:34 PM rows. Each row names the PR's author, neo-opus-vega, as its actor. The cockpit half (the row's wording) is the Institution consumer, neomjs/neo-agent-institution#593. This ticket is the data the feed must carry for it.

## The Problem

- **A push that answers a change request never reaches the feed.** The feed carries four transition kinds, `opened`, `verdict`, `merged` and `closed` (`PR_LANE_TRANSITION_KINDS`, `ai/services/fleet/producerPrLaneEvents.mjs`). The reducer already records every push as a `head` transition (`changesOf`, `ai/services/fleet/openWorkReducer.mjs`). "Changes addressed" is that push; it is just not on the feed.
- **A verdict names no reviewer.** A verdict transition records the `from` and `to` of the PR's aggregate `reviewDecision`. `createPrTransitionEvents` sets the event's actor (`agentId`) to the PR's author (`owner.login`), so a reviewer's approval reads under the author's name.

## The Architectural Reality

- `normalizePullRequest` already carries each reviewer's standing opinion (`opinions: [{reviewer, state, onHead}]`). The reviewers who moved a verdict are those whose standing opinion in the new row is the new decision and was not in the old one.
- `PR_LANE_TRANSITION_KINDS` is the feed's declared kind list. Adding a kind changes what the cockpit consumer receives, so the consumer (neomjs/neo-agent-institution#593) is written to render it.
- No new file: the change stays in `openWorkReducer.mjs` and `producerPrLaneEvents.mjs` (structure map run 2026-10-07, exit 0).
- Design authority: none needed. This adds facts the reducer already holds; it changes no documented behavior.

## The Fix

- `changesOf` puts `by` on a verdict transition: the reviewers observed moving the decision. A reviewer moved it when their standing opinion became the new decision, or when the same opinion moved onto the current head (a re-approval). Both reads must hold every standing opinion (`opinionsComplete`). Otherwise, like a dismissal or a branch rule, the change names no one.
- `createPrTransitionEvents` carries `by` in `payload.transition`. When exactly one reviewer moved the verdict, that reviewer is the event's actor. Otherwise the actor stays the author.
- A push to a PR whose verdict is `CHANGES_REQUESTED` reaches the feed as one event. Any other push stays off it.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `payload.transition.kind` gains `head` | `PR_LANE_TRANSITION_KINDS` + `createPrTransitionEvents` (`ai/services/fleet/producerPrLaneEvents.mjs`) | a push shows only when the verdict it landed on (`transition.verdict`, from `contextOf`) was `CHANGES_REQUESTED` | every other push stays off the lane, as before | `PR_LANE_TRANSITION_KINDS` / `createPrTransitionEvents` JSDoc | unit (`producerPrLaneEvents.spec.mjs`) |
| `payload.transition.by` (optional; `@<login>`, `login:<login>`, `team:<org>/<slug>`) | `contextOf` (`ai/services/fleet/openWorkReducer.mjs`) over `normalizePullRequest`'s `opinions` + `opinionsComplete` | the reviewers observed moving the verdict: a new standing decision, or a re-approval onto the current head | omitted when no reviewer is established: a dismissal, a rule, or either read missing or cutting its opinions | `contextOf` JSDoc | unit (`openWorkReducer.spec.mjs`) |
| The event's actor (`agentId`) | `createPrTransitionEvents` | the reviewer's login when exactly one reviewer moved a verdict | the PR's author: several reviewers, none, or a team | `createPrTransitionEvents` JSDoc | unit, reducer to event |
| Events and rows from before this change | the producer's saved state (`open-work.json`) | a saved row without `opinionsComplete` attributes nothing; an event without `by` reads as no reviewer | — | same | unit |
| The consumer | neomjs/neo-agent-institution#593 / #594 (`getPullRequestStatus`) | renders `head` as `changes pushed`, and `by` only when several reviewers moved a verdict | an unknown kind names nothing | the consumer's own ledger on #593 | its unit tests; installed: neomjs/neo-agent-institution#490 |

## Acceptance Criteria

- [ ] AC-1: a verdict transition names the reviewers whose standing opinion became the new decision. A verdict change that no opinion explains names none (unit).
- [ ] AC-2: a verdict event moved by exactly one reviewer has that reviewer as its actor. With none or several, the actor stays the author (unit).
- [ ] AC-3: a push to a PR whose verdict is changes-requested reaches the feed as one event. A push to any other PR does not (unit, with that control).

## Out of Scope

- The cockpit wording (neomjs/neo-agent-institution#593).
- Review-request transitions on the feed.
- PR titles on transition events.

## Related

neomjs/neo-agent-institution#414 (parent, row 4) · neomjs/neo-agent-institution#593 (the consumer) · #917 last changed the same reducer (merged 2026-10-07, `359527bf`).

Live latest-open sweep: checked the latest 20 open Brain and Institution issues at 13:43Z; no equivalent. A2A claim sweep (last 30 messages): no overlapping claim. Memory Core sweep ("Activity feed PR row shows only the PR number, not what happened"): no prior decision. Own-assignment sweep: no same-surface ticket.

Origin Session ID: 86872728-bfc4-469c-b029-e656b742474f


## Timeline

- 2026-10-07T13:46:14Z @neo-opus-vega added the `enhancement` label
- 2026-10-07T13:46:14Z @neo-opus-vega added the `ai` label
- 2026-10-07T13:46:15Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-07T13:46:26Z @neo-opus-vega cross-referenced by #593
- 2026-10-07T13:46:52Z @neo-opus-vega added parent issue #414
- 2026-10-07T13:52:40Z @neo-opus-vega cross-referenced by PR #920
- 2026-10-07T13:56:59Z @neo-opus-vega cross-referenced by PR #594
- 2026-10-07T14:39:04Z @neo-opus-vega referenced in commit `16c3d4a` - "fix(fleet): a verdict names a reviewer only from what both reads observed, a re-approval on the current head included (#919)

contextOf compared reviewer and state alone. A re-approval on the current head (same state, the review moved off an older commit) went unattributed, and a prior read that cut or lacked its opinions made an unseen standing approval look new. A reviewer now moved the verdict when their standing opinion became the decision or moved onto the current head, and only when both reads hold every standing opinion (normalizePullRequest now keeps opinionsComplete). Otherwise the change names no one. Review RA-1."
- 2026-10-07T14:50:29Z @tobiu referenced in commit `197e659` - "feat(fleet): the PR lane names who moved a verdict and the push that answers a change request (#919) (#920)

* feat(fleet): the PR lane names who moved a verdict and the push that answers a change request (#919)

A verdict transition carries the reviewers whose standing opinion became the new decision (none for a dismissal or a rule), and an event moved by exactly one reviewer has that reviewer as its actor. A push records the verdict it landed on, and the PR lane shows a push only when that verdict is CHANGES_REQUESTED, so the cockpit can say 'changes pushed'.

* fix(fleet): a verdict names a reviewer only from what both reads observed, a re-approval on the current head included (#919)

contextOf compared reviewer and state alone. A re-approval on the current head (same state, the review moved off an older commit) went unattributed, and a prior read that cut or lacked its opinions made an unseen standing approval look new. A reviewer now moved the verdict when their standing opinion became the decision or moved onto the current head, and only when both reads hold every standing opinion (normalizePullRequest now keeps opinionsComplete). Otherwise the change names no one. Review RA-1."
- 2026-10-07T14:50:30Z @tobiu closed this issue

