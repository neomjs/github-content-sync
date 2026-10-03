---
id: 480
title: ROADMAP rows 2 and 4 name their epics; row 1 records the served-plane repair
state: CLOSED
labels:
  - documentation
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-10-03T08:24:42Z'
updatedAt: '2026-10-03T11:11:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/480'
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
closedAt: '2026-10-03T11:11:13Z'
---
# ROADMAP rows 2 and 4 name their epics; row 1 records the served-plane repair

The FM v1 ROADMAP is the lane board; two of its rows point at nothing a peer can pick. Row 2 ("truthful state and recovery guidance") lists only closed anchors and no epic — its epic is #477 as of today, with #478 (the census) and #479 (the installed walkthrough) as its first leaves. Row 4 ("one representative engineering workflow") still reads `open option` in the steward cell while #414 exists, is Grace's, and has three merged leaves (#415, #418, #426 via PRs #417, #461, #428). Row 1's state cell predates the served-plane repair: neomjs/neo-agent-brain#784 (PR #785, merged 2026-10-03) is the first Brain revision on which the wizard's `served-plane` step reads `ok` against a real plane, and neomjs/neo-agent-brain#782 (PR #796, in review) adds `validation` / `verify` / `done`.

## The Fix

`ROADMAP.md`, three cells:
- row 2 — Anchors gain #477 (the row's epic) with #478 and #479; the Steward cell names Clio as the row's steward with the epic; the State cell keeps `unknown` and names #479 as the check that will fill it.
- row 4 — Anchors gain #414 (the row's epic; its merged leaves #415 / #418 / #426); the Steward cell reads **Grace** — the outcome (#414's author, claimed 2026-10-02); the State cell keeps `unknown` and says the recording is the missing receipt.
- row 1 — the State cell gains one sentence: the served-plane observer repair (Brain #784 via PR #785, `dev@b2cd8be`) and the witness effect (Brain #782, PR #796 in review), with the live receipt that the complete first run read `done ok` against the canonical plane on 2026-10-03.

No other row changes; the planning finding behind this (four rows with zero open leaves) lives in the lane board A2A of 2026-10-03 08:18Z and in #477's problem scope, not in the ROADMAP.

## Acceptance Criteria

- AC-1: rows 2 and 4 each name their epic in Anchors and their steward in the Steward cell; no row reads `open option` while an owned epic exists for it.
- AC-2: row 1's State cell names #784 / PR #785 and #782 / PR #796 with the 2026-10-03 live receipt, bounded as the team's own receipt, not an outside operator's.

## Out of Scope

Row 3's and row 5's cells (their stewards update them with their receipts). Any state change to `passed`.

## Related

#335 (the ROADMAP's own ticket) · #477 · #414 · neomjs/neo-agent-brain#784 · neomjs/neo-agent-brain#782

Live latest-open sweep: the latest 20 open Institution issues read at 2026-10-03T08:22Z; no ROADMAP row update is open (the last was #456 → PR #457, row 5, 2026-10-02). A2A: none competing. Own-assignment: #351, #477.

Origin Session ID: fb9561d9-a0dd-4f35-912c-095864afbae4
Retrieval Hint: "ROADMAP row 2 epic 477 row 4 steward Grace 414 row 1 served-plane 785"

## Timeline

- 2026-10-03T08:24:42Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-03T08:24:44Z @neo-fable-clio added the `documentation` label
- 2026-10-03T08:24:44Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T08:24:44Z @neo-fable-clio added the `ai` label
- 2026-10-03T08:26:26Z @neo-fable-clio cross-referenced by #481
- 2026-10-03T08:28:43Z @neo-fable-clio cross-referenced by PR #482
- 2026-10-03T10:49:41Z @neo-fable-clio referenced in commit `ba47806` - "docs(roadmap): merge dev into the rows branch; row 5 keeps dev's install:mac clause beside the rewritten row 4 (#480)"
- 2026-10-03T10:59:34Z @neo-fable-clio cross-referenced by #499
- 2026-10-03T11:05:48Z @neo-fable-clio cross-referenced by #500
- 2026-10-03T11:11:13Z @tobiu referenced in commit `231cead` - "docs(roadmap): rows 2 and 4 name their epics and stewards; row 1 records the served-plane repair and the witness effect (#480) (#482)"
- 2026-10-03T11:11:14Z @tobiu closed this issue

