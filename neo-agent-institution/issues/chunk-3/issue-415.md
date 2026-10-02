---
id: 415
title: The Activity PR row names the pull request's state and review verdict
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T08:30:42Z'
updatedAt: '2026-10-02T08:30:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/415'
author: neo-opus-grace
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
milestone: FM v1
---
# The Activity PR row names the pull request's state and review verdict

## Context

FM v1 row 4 (#414) watches a pull request through review and merge from the cockpit. Its script expects the Activity stream to show the review's verdict, then the merge. A [source audit](https://github.com/neomjs/neo-agent-institution/issues/335#issuecomment-5948137866) found that the Activity row renders a PR event as its ref and title only.

## The Problem

- `getActivityObjectText` (`apps/agentos/view/fleet/activity/RowContainer.mjs`) builds the row's object cell from `payload.number`, `repoSlug` and `title`.
- The Brain's `pr-activity` event already carries `state`, `reviewDecision` and `isDraft` (`createPrActivityEvents`, Brain `ai/services/fleet/fleetPrLaneActivityAdapter.mjs:150-166`). Nothing in `apps/agentos` reads them: `reviewDecision` occurs nowhere in it.
- So a PR that is approved, has changes requested, or merges only moves its row's time. The two gates row 4 exists to show, the cross-family review and the human merge, stay invisible.
- `KindRegistry` maps a `review` kind that no producer emits, so the verdict has no other surface either.

## The Architectural Reality

- Each PR has one `pr-activity` event under a stable `eventId`, re-emitted with the PR's latest `updatedAt`.
- `Neo.list.Buffered` recycles `RowContainer` by assigning `record`. `afterSetRecord` → `updateRow()` sets the object cell's `text`, and the row's `aria-label` joins the same text.
- The payload enums are GitHub's:
  - `state`: `OPEN` | `CLOSED` | `MERGED`
  - `reviewDecision`: `APPROVED` | `CHANGES_REQUESTED` | `REVIEW_REQUIRED` | `null`
  - `isDraft`: a boolean
- The cockpit's sample fixture (`test/playwright/fixture/fleetSample.mjs`) gives its PR events text payloads only, so no visual golden carries a PR state.

## The Fix

1. `getPullRequestStatus(event)`, exported beside `getActivityObjectText`. It is a pure resolver over one closed vocabulary:
   - `MERGED` → `merged`; `CLOSED` → `closed`;
   - an `OPEN` PR → `draft` when `isDraft`, else `approved`, `changes requested` or `review required` by its `reviewDecision`;
   - anything else → `null`, so nothing is guessed. That covers a non-PR event, an unknown or missing enum, and an open PR with no decision.
2. `getActivityObjectText` appends the label to a PR's object text: `brain#739 · title · approved`. The `aria-label` follows, because it joins the same text.
3. The row's five-cell anatomy stays: no new cell, token or chip.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `getActivityObjectText(event)`, consumed by `RowContainer#updateRow` and its unit spec | `RowContainer.mjs` | a `pr-activity` text ends `· <status>` when the status resolves | the text it renders today | JSDoc | unit matrix |
| `getPullRequestStatus(event)`, new export | `RowContainer.mjs` | the closed vocabulary above | `null` | JSDoc | unit matrix |

## Acceptance Criteria

- [ ] AC-1: a `pr-activity` row reads `merged` or `closed` for those states. An open PR reads `draft` when `isDraft`, else `approved`, `changes requested` or `review required` by its decision. An open PR with no decision, an unknown enum, or a missing payload adds nothing. Unit matrix over both functions.
- [ ] AC-2: every other kind's object text is unchanged; the existing grammar arms pass unmodified (unit).
- [ ] AC-3: when a recycled row's record moves from approved to merged, the object cell and the `aria-label` re-label in place (unit, the row component).
- [ ] AC-4: the full visual suite passes with no golden re-captured (visual).
- [ ] AC-5 `[L4-deferred — operator handoff needed]`: on the installed candidate, a real PR's approval and merge show on its row within the activity cadence. These are row 4's steps 4–5; residual owner #414.

## Out of Scope

- A Brain `review` event: the PR's one event already carries the verdict.
- State colours or a status pill. That's a design follow-up, if the text reads poorly at the sitting.
- The Agent Detail's Pull requests pane (#391, `D#19122`), and issue-activity states.

## Related

Parent #414 · #335 (the row-4 script and the audit) · #391

Decision Record impact: `none`. Structure map: N/A (one existing view module, no new file).

Live latest-open sweep: latest open Institution issues re-read at 2026-10-02T08:30:24Z, no equivalent (the newest is #414, this leaf's parent). MC sweep: "activity pull request row merged review verdict invisible pr chip review chip", 6 results, no prior decision found. Own-assignment sweep: 3 open (#414, #386, #11), none overlapping. A2A: last 30 read, no claim on the Activity row.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca
Retrieval Hint: "Activity PR row state review verdict getActivityObjectText getPullRequestStatus pr-activity reviewDecision"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-10-02T08:30:42Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-02T08:30:43Z @neo-opus-grace added the `enhancement` label
- 2026-10-02T08:30:44Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T08:30:44Z @neo-opus-grace added the `ai` label
- 2026-10-02T08:30:52Z @neo-opus-grace added parent issue #414
- 2026-10-02T08:30:56Z @neo-opus-grace added this to the **FM v1** milestone

