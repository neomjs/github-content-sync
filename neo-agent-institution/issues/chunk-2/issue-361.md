---
id: 361
title: 'ROADMAP: the first row-state update — row 1 gains its epic and two receipts, row 3 its resolution, row 5 its update-path anchors'
state: CLOSED
labels:
  - documentation
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-30T15:05:53Z'
updatedAt: '2026-09-30T15:46:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/361'
author: neo-fable-clio
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
closedAt: '2026-09-30T15:46:46Z'
milestone: FM v1
---
# ROADMAP: the first row-state update — row 1 gains its epic and two receipts, row 3 its resolution, row 5 its update-path anchors

## Context

`ROADMAP.md` (#335 → PR #336, `dev@d692b09`) fixes three accounting rules: a merged PR never retires an installed check; every row carries one dated state with a receipt; a row's scope changes only with a dated reason. Its *How it runs* paragraph makes each session's outcomes update the rows. Since it landed (13:34Z) the following objects changed, read live on 2026-09-30 15:0xZ:

- **Row 1 (first run and connection):** the graduating epic exists — [#351](https://github.com/neomjs/neo-agent-institution/issues/351), provisional until D#18965's quorum; [#214](https://github.com/neomjs/neo-agent-institution/issues/214) closed 15:02Z via [PR #350](https://github.com/neomjs/neo-agent-institution/pull/350) (merged 14:56Z), whose evidence is L3 on a checkout **and a packaged build** (`organism-build-info.json` staged 14:05Z, Brain `5153a4b`, Engine `067f9fb`, Electron 43.5.0): the stored-plane boot against a fixture plane reports `{mode: 'plane-attach', up: true}`; [#341](https://github.com/neomjs/neo-agent-institution/issues/341) closed 14:56Z via [PR #342](https://github.com/neomjs/neo-agent-institution/pull/342) — Home offers *Connect a plane* on first run (door two; the wizard's door is #351's); the first FM-launched seat completed a turn on the team's plane (@neo-gpt-emmy's receipt, A2A 14:43Z: full app Institution `54d8ac2` · Brain `6a714ae` · Engine `067f9fb` · Electron 43.5; isolated process, operator login, first-turn assent witnessed — "distinct boundaries, not a claim of completed full onboarding").
- **Row 3 (the Observatory):** [#312](https://github.com/neomjs/neo-agent-institution/issues/312)'s epic-resolution review by its steward (14:17Z, `RECOMMEND_CREATE_MISSING_SUBS`): all six leaves closed at L2; the installed walkthrough is the epic's own L4 close; two untracked gaps (Q5 evidence for non-source kinds; the cold `get_graph_scene` read on the installed FM).
- **Row 5 (ordinary supported recovery):** the update path gained two merged anchors — [#345](https://github.com/neomjs/neo-agent-institution/issues/345) → PR #346 (Fleet registrations, credentials and keys live under `<userData>/brain/fleet`, outside the replaceable bundle; installed witness owed under #12) and [#347](https://github.com/neomjs/neo-agent-institution/issues/347) → PR #348 (every plane member under the user data root; closed 14:31Z).

## Problem

The file's rows are dated 2026-09-30 morning and do not carry these receipts; a reader of row 1 cannot find the epic, and row 5's update arm still names only #7 and #259. Under the file's own rules, none of these merges changes a row's **state** — no installed check as one journey has run — so the update is ledger and anchors, with every state cell staying as it is and saying why.

## Proposed solution

One dated edit of `ROADMAP.md`:

1. Row 1 anchors: add #351 (the row's epic, provisional); #214 and #341 marked closed with dates. Row 1 state stays `blocked` (the wizard is #351's deliverable); its receipt line gains the three receipts above — the packaged stored-plane boot against a fixture plane (a check of the *connect* door, on our own fixture, not an outside operator's run), Home's Connect door, and the first FM-launched seat with the full-app pins.
2. Row 3 state stays `blocked`; its text cites the epic-resolution review and names the two gaps as the row's open items beside the three L4 residuals.
3. Row 5 anchors: add #345/#346 and #347/#348 to the update arm; state stays `unknown` with the reason (installed witness owed under #12).
4. No scope change on any row; the deferred set unchanged.

## Acceptance Criteria

- [ ] AC-1 Row 1's anchor cell names #351 with the provisional qualifier and marks #214 / #341 closed with their dates; its state cell stays `blocked` and its receipt line carries the three receipts with links (PR #350's evidence, PR #342, the A2A receipt quoted with its pins).
- [ ] AC-2 Row 3's state cell cites the epic-resolution review (comment id) and lists the two gaps; no state change.
- [ ] AC-3 Row 5's anchor cell adds the two update-path leaves with their PRs; state `unknown` with the owed witness named.
- [ ] AC-4 `git diff --stat`: `ROADMAP.md` only; no state cell changes value; every added link resolves (`gh api` per target).

## Sweeps

(i) artifact: open Institution issues mentioning "ROADMAP row" → #349, #351 (neither is this update). (ii) live latest-open queue (#359 #358 #355 #354 #351 #349 #312 #287, read 15:04Z) — unrelated. (iii) Memory Core rationale sweep (`query_raw_memories` on the roadmap's accounting rules) — nothing beyond my own trail: the update was deferred at 13:37Z with the named falsifier "#350's merge or D#18965's quorum"; #350 merged 14:56Z. (iv) own open assignments here: #351 — not this. (v) n/a (leaf).

## Related

#335 · PR #336 (`d692b09`) · #351 · D#18965 · #214 / PR #350 · #341 / PR #342 · #312 · #345 / PR #346 · #347 / PR #348 · #12 · [milestone FM v1](https://github.com/neomjs/neo-agent-institution/milestone/1)

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 4a2cca3d-9951-4e9a-b577-2a3374a22045

## Timeline

- 2026-09-30T15:05:53Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-30T15:05:56Z @neo-fable-clio added the `documentation` label
- 2026-09-30T15:05:56Z @neo-fable-clio added the `agent-os` label
- 2026-09-30T15:05:56Z @neo-fable-clio added the `ai` label
- 2026-09-30T15:06:15Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-09-30T15:08:28Z @neo-fable-clio cross-referenced by PR #363
- 2026-09-30T15:44:39Z @tobiu referenced in commit `904c277` - "Merge pull request #363 from neomjs/clio/361-roadmap-first-ledger

docs(roadmap): the first row-state update — row 1's epic and receipts, row 3's resolution, row 5's update-path anchors (#361)"
- 2026-09-30T15:46:47Z @tobiu closed this issue

