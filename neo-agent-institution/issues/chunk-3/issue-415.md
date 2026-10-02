---
id: 415
title: The Activity PR row names the pull request's state and review verdict
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T08:30:42Z'
updatedAt: '2026-10-02T09:17:46Z'
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
closedAt: '2026-10-02T09:16:42Z'
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
- **Found while building (intake):** even a rendered status would never change on screen. `FleetActivityEvents#ingestSnapshot` updates a known event in place (`record.set`) with its events suspended, then fires one `load`. The list re-binds the same record instance, so the row's `afterSetRecord` never fires, and any row whose event was upserted keeps its first rendering until it is recycled. The unit arm proves it: after the merge snapshot, the row still read `approved`.

## The Architectural Reality

- Each PR has one `pr-activity` event under a stable `eventId`, re-emitted with the PR's latest `updatedAt`.
- `Neo.list.Buffered` recycles `RowContainer` by assigning `record`. `afterSetRecord` → `updateRow()` sets the object cell's `text`, and the row's `aria-label` joins the same text.
- Before every bind, `Buffered#getPooledComponent` stamps `component.lastRecordVersion = record.version`. `RecordFactory` bumps that version only when a field really changes; `grid.column.Component` reads the same stamp. This is the engine's in-place-change signal for pooled components.
- The Brain's payload also carries `humanGateState` (`approved` / `changedRequested`, read from the latest reviews by `getPrHumanGateState`). GitHub leaves `reviewDecision` null on a repository without required reviews, which can describe an outside operator's own repository.
- The payload enums are GitHub's:
  - `state`: `OPEN` | `CLOSED` | `MERGED`
  - `reviewDecision`: `APPROVED` | `CHANGES_REQUESTED` | `REVIEW_REQUIRED` | `null`
  - `isDraft`: a boolean
- The cockpit's sample fixture (`test/playwright/fixture/fleetSample.mjs`) gives its PR events text payloads only, so no visual golden carries a PR state.

## The Fix

1. `getPullRequestStatus(event)`, exported beside `getActivityObjectText`. It is a pure resolver over one closed vocabulary:
   - `MERGED` → `merged`; `CLOSED` → `closed`;
   - an `OPEN` PR → `draft` when `isDraft`, else `approved`, `changes requested` or `review required` by its `reviewDecision`, and where GitHub computes none, by `humanGateState`;
   - anything else → `null`, so nothing is guessed. That covers a non-PR event, an unknown or missing enum, and an open PR with no verdict.
2. `getActivityObjectText` appends the label to a PR's object text: `brain#739 · title · approved`. The `aria-label` follows, because it joins the same text.
3. `RowContainer` declares `lastRecordVersion_` reactive. `afterSetLastRecordVersion` redraws only when the record already shown has moved past the version it was drawn at (`renderedVersion`). A bind of a different record still arrives through `afterSetRecord`, so a recycle never draws twice.
4. The row's five-cell anatomy stays: no new cell, token or chip. The store is unchanged.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `getActivityObjectText(event)`, consumed by `RowContainer#updateRow` and its unit spec | `RowContainer.mjs` | a `pr-activity` text ends `· <status>` when the status resolves | the text it renders today | JSDoc | unit matrix |
| `getPullRequestStatus(event)`, new export | `RowContainer.mjs` | the closed vocabulary above | `null` | JSDoc | unit matrix |
| `RowContainer#lastRecordVersion_`, written by `Neo.list.Buffered#getPooledComponent` | the engine's bind stamp | a same-record bind at a new version redraws the row | no redraw (the version is unchanged) | JSDoc | the in-place re-label arm |

## Acceptance Criteria

- [x] AC-1: a `pr-activity` row reads `merged` or `closed` for those states. An open PR reads `draft` when `isDraft`, else `approved`, `changes requested` or `review required` by its decision, falling back to `humanGateState` when GitHub gives none. An open PR with no verdict, an unknown enum, or a missing payload adds nothing. Unit matrix over both functions.
- [x] AC-2: every other kind's object text is unchanged; the existing grammar arms pass unmodified (unit).
- [x] AC-3: when a recycled row's record moves from approved to merged, the object cell and the `aria-label` re-label in place (unit, the row component).
- [x] AC-4: the full visual suite passes with no golden re-captured (visual).
- [ ] AC-5 `[L4-deferred — operator handoff needed]`: on the installed candidate, a real PR's approval and merge show on its row within the activity cadence. These are row 4's steps 4–5; residual owner #414.

AC-1 to AC-4 were delivered by PR #417 at `901a008`, merged as `af1e5ee` on 2026-10-02 after cross-family approval by @neo-gpt-sophie (review 5389926329). Each AC's evidence row is in the PR body. AC-5 is row 4's installed sitting, on #414.

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
- 2026-10-02T08:42:22Z @neo-opus-grace cross-referenced by PR #417
- 2026-10-02T08:52:45Z @neo-fable cross-referenced by #418
- 2026-10-02T09:16:42Z @tobiu referenced in commit `af1e5ee` - "feat(agentos): the Activity PR row names the pull request's state and review verdict (#415) (#417)

getPullRequestStatus reads the pr-activity payload the Brain already sends (state, isDraft, reviewDecision, and humanGateState where GitHub computes no decision) into one closed vocabulary, and the row's object cell appends it. RowContainer reacts to the lastRecordVersion stamp Neo.list.Buffered sets before every bind, so an event upserted in place redraws its row instead of keeping its first rendering until it is recycled."
- 2026-10-02T09:16:43Z @tobiu closed this issue
- 2026-10-02T11:38:26Z @neo-opus-grace cross-referenced by #414

