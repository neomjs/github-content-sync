---
id: 593
title: 'An Activity PR row says what happened to the PR, not just its number'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-07T13:46:24Z'
updatedAt: '2026-10-07T14:43:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/593'
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
closedAt: '2026-10-07T14:43:13Z'
---
# An Activity PR row says what happened to the PR, not just its number

## Context

On 2026-10-07 the operator showed his installed Activity tab: the PR rows read `brain#915` and `brain#917` with no event, and only a merge says `· merged`. He asked for the meaning instead: PR created, changes requested, changes addressed, approved. The rows are real events. At 02:30 PM CEST Sophie requested changes on Brain #915, and at 02:50 PM she approved it.

## The Problem

The PR rows now come from the Brain's open-work producer (`createPrTransitionEvents`, `ai/services/fleet/producerPrLaneEvents.mjs` in neo-agent-brain). It carries an event's meaning in `payload.transition: {kind, from, to}`. It sets `payload.state` only for `opened`, `merged` and `closed`, and a verdict event's `reviewDecision` is its `to`.

The row's status resolver `getPullRequestStatus` (`apps/agentos/view/fleet/activity/RowContainer.mjs`) still reads the corpus shape. It names `merged` or `closed`, then returns null unless `state === 'OPEN'`. So a verdict event (`state` null) names nothing, and an opened event (`OPEN`, no decision) names nothing either.

## The Architectural Reality

- Design authority: the resolver's own JSDoc says "An open PR names GitHub's `reviewDecision`". The producer's verdict events carry that decision, and the resolver never reaches it.
- The actor cell shows the event's `agentId`. The producer sets it to the PR's author, so a reviewer's approval reads under the author's name. Who moved a verdict, and a push answering a change request, are the Brain sibling neomjs/neo-agent-brain#919.

## The Fix

`getPullRequestStatus` reads `payload.transition` before the corpus path:
- `opened` → `opened`;
- `verdict` → the existing decision words for its `to` (`changes requested`, `approved`, `review required`);
- `head` → `changes pushed`;
- `merged` and `closed` as today.

A transition that names its reviewers (`transition.by`, #919) shows them after the word. An event without `transition` keeps today's path. An unknown kind or `to` names nothing; a word is never guessed.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `payload.transition.kind` (`opened`, `verdict`, `head`, `merged`, `closed`) | Brain `createPrTransitionEvents` over `PR_LANE_TRANSITION_KINDS` (`ai/services/fleet/producerPrLaneEvents.mjs`); `head` from neomjs/neo-agent-brain#920 | `opened`, `changes pushed`, the decision words, or the PR's state | any other kind names nothing, whatever `payload.state` says | `getPullRequestStatus` / `transitionStatus` JSDoc | unit (`container.spec.mjs`) |
| `payload.transition.to` on a verdict (`APPROVED`, `CHANGES_REQUESTED`, `REVIEW_REQUIRED`) | GitHub's `reviewDecision`, recorded by the Brain reducer (`openWorkReducer.changesOf`) | `approved` / `changes requested` / `review required` | any other decision names nothing | same | unit |
| `payload.transition.by` (`@<login>`, `login:<login>`, `team:<org>/<slug>`) | Brain #920's `contextOf`: the reviewers whose standing opinion became the decision | named after the word only when several moved the verdict | absent or empty names no one, never the author | same | unit; producer half in #920's unit tests |
| `payload.state` (`MERGED`, `CLOSED`, `OPEN`, null) | Brain `PR_STATE_BY_KIND` | read only for the `merged` and `closed` kinds | — | same | unit |
| The event's actor (`agentId`) | Brain #920: the single reviewer who moved a verdict, otherwise the PR's author | the actor cell shows it; this text never repeats it | until #920 merges, every event's actor is the author, and a single-reviewer verdict reads only its word | the row's actor cell | #920 unit; installed: #490 |
| Corpus-shaped events (no `transition`) | the corpus PR adapter (legacy) | the existing state, draft and decision path, unchanged | — | same | unit (existing controls) |

## Acceptance Criteria

- [ ] AC-1: producer-shaped opened, changes-requested, approved, merged and closed events each name their word on the row (unit).
- [ ] AC-2: a verdict that several reviewers moved names them after the word. A single reviewer is the row's actor (#919), so the text does not repeat them. One with no reviewer names none, never the author as reviewer (unit).
- [ ] AC-3: a `head` event names `changes pushed` (unit).
- [ ] AC-4: control: a corpus-shaped event (no `transition`) resolves as today (unit).

## Post-Merge Validation

- [ ] On the next installed candidate, one PR's review round reads `changes requested`, then `changes pushed`, `approved` and `merged`, each by the seat that acted (after #919 too).
  Residual-Owner: neomjs/neo-agent-institution#490

## Out of Scope

- PR titles on these rows (the transition events carry none).
- Review-request rows.

## Related

#414 (parent, row 4) · neomjs/neo-agent-brain#919 (the data) · #490 (row 4's installed walk).

Live latest-open sweep: checked the latest 20 open Institution and Brain issues at 13:43Z; no equivalent. A2A claim sweep (last 30 messages): no overlapping claim. Memory Core sweep: no prior decision. Own-assignment sweep: no same-surface ticket.

Owner: Vega, who also builds the Brain sibling. This change alone already names opened, changes requested, approved and merged on today's feed.

Origin Session ID: 86872728-bfc4-469c-b029-e656b742474f


## Timeline

- 2026-10-07T13:46:26Z @neo-opus-vega added the `bug` label
- 2026-10-07T13:46:26Z @neo-opus-vega added the `ai` label
- 2026-10-07T13:46:37Z @neo-opus-vega cross-referenced by #919
- 2026-10-07T13:46:51Z @neo-opus-vega added parent issue #414
- 2026-10-07T13:48:55Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-07T13:52:40Z @neo-opus-vega cross-referenced by PR #920
- 2026-10-07T13:56:59Z @neo-opus-vega cross-referenced by PR #594
- 2026-10-07T14:27:31Z @neo-opus-vega referenced in commit `2c72b40` - "fix(fleet): an unknown transition kind names nothing, even on a merged or closed PR (#593)

transitionStatus fell back to the PR-state words for every kind other than a verdict, a push or an opening, so a CI event on a merged PR read 'merged'. The state words now answer only the merged and closed kinds; any other kind names nothing. Review RA-1."
- 2026-10-07T14:43:13Z @tobiu referenced in commit `af8b67f` - "feat(fleet): an Activity PR row says what happened to the PR, not just its number (#593) (#594)

* feat(fleet): an Activity PR row says what happened to the PR, not just its number (#593)

getPullRequestStatus reads the open-work producer's payload.transition before the corpus path. A verdict names its new decision, a push answering a change request 'changes pushed', an opening 'opened', and a merge or close the PR's state. Reviewers are named only when several moved a verdict, since a single one is the row's actor. An unknown kind or decision names nothing. The visual-baseline stamp moves with RowContainer's blob; no capture spec renders a pr-activity row.

* fix(fleet): an unknown transition kind names nothing, even on a merged or closed PR (#593)

transitionStatus fell back to the PR-state words for every kind other than a verdict, a push or an opening, so a CI event on a merged PR read 'merged'. The state words now answer only the merged and closed kinds; any other kind names nothing. Review RA-1."
- 2026-10-07T14:43:14Z @tobiu closed this issue

