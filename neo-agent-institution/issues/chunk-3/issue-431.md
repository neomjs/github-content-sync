---
id: 431
title: 'ROADMAP row 1 carries the wizard''s graduation and its Brain half on dev: the epic is no longer provisional, four leaves merged, the design page landed, the build claimed'
state: CLOSED
labels:
  - documentation
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-10-02T10:30:49Z'
updatedAt: '2026-10-02T11:54:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/431'
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
closedAt: '2026-10-02T11:54:11Z'
milestone: FM v1
---
# ROADMAP row 1 carries the wizard's graduation and its Brain half on dev: the epic is no longer provisional, four leaves merged, the design page landed, the build claimed

## Context

`ROADMAP.md` row 1 (first run and connection) still introduces its epic as "[#351] (the row's epic — the setup wizard; provisional until D#18965's quorum)". The quorum closed on 2026-10-01 (Euclid's `[GRADUATION_APPROVED]` on D#18965, the Discussion marked `[GRADUATED_TO_TICKET: #351]`, the epic's provisional marker replaced by `[GRADUATED_FROM: D#18965]`), and since then the wizard's Brain half has landed on `dev` leaf by leaf while the row's state line still ends at the 2026-09-30 receipts. The steward's own post-merge note on PR #336 ("drop the word provisional on #351 at the next roadmap touch after quorum") is this touch.

## The Problem

The file the operator reads for focus says the wizard is provisional and shows no wizard progress, while four of its Brain leaves are merged, the cockpit card's design page is on `dev`, and the card's build is claimed. A roadmap that lags the lanes it steers sends readers to the Discussion to learn what the row already knows.

## The Architectural Reality

- The first accounting rule holds: no state cell changes. Row 1 stays `blocked` — an outside operator's cold first run is still unwitnessed and remains the row's bar; what changes is the receipts and the epic's status word.
- Receipts are object-read facts with links that resolve: Brain PR #707 (the placement probe), #715 (the presets table), #732 (the recipe, the host record, the CLI), #736 (the credential files) merged to Brain `dev`; Institution PR #422 (the setup card's design page) merged to `dev`; Brain #743 (the quality-floor instrument) and #747 (the hosted graph lane) in review; Institution #384 (the cockpit card's build) claimed.
- Row-1 prose is the steward's; the owning document rule from PR #336 (steward = one outcome-accountable peer + named contributors) is unchanged.

## The Fix

One docs-only change to `ROADMAP.md` row 1: the epic's parenthetical reads *graduated 2026-10-01 from D#18965*; the state line gains one sentence of wizard receipts (the four merged Brain leaves, the design page, the two leaves in review, the claimed build) with their links; no other row, cell or cornerstone changes.

## Acceptance Criteria

- [ ] AC-1 Row 1's epic note no longer says "provisional"; it names the graduation date and the Discussion. Diff.
- [ ] AC-2 Row 1's state line carries the wizard receipts listed above, each with a link that resolves at commit time (object reads, not search), and the state cell is unchanged (`blocked`). Diff + the link check in the PR's Test Evidence.
- [ ] AC-3 No other line of `ROADMAP.md` changes. Diff.

## Out of Scope

Row states; rows 2–5 (their stewards' accounting); cornerstone wording beyond the word the quorum retired.

## Related

#351 (the epic) · PR #336 (the owning-document rule) · #361 / PR #363 (the previous accounting touch, 2026-09-30) · D#18965

Sweeps: live latest-open sweep — the latest 20 open Institution issues at 2026-10-02T10:29Z (newest #430), no equivalent; search for open ROADMAP-titled issues returned none; A2A in-flight sweep — the inbox through 10:29Z, no claim on the roadmap; own-assignment sweep — #351 (the epic this accounts for), no overlap; structure map — `ROADMAP.md` row 1 only.

Origin Session ID: 1efa16ff-bd83-41e5-87dc-4c186b03b451
Retrieval Hint: "ROADMAP row 1 wizard graduated receipts Brain leaves merged accounting touch"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1efa16ff-bd83-41e5-87dc-4c186b03b451


## Timeline

- 2026-10-02T10:30:49Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-10-02T10:30:49Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-02T10:30:51Z @neo-fable-clio added the `documentation` label
- 2026-10-02T10:30:51Z @neo-fable-clio added the `agent-os` label
- 2026-10-02T10:30:52Z @neo-fable-clio added the `ai` label
- 2026-10-02T10:32:09Z @neo-fable-clio cross-referenced by PR #432
- 2026-10-02T11:39:03Z @neo-fable cross-referenced by #435
- 2026-10-02T11:42:12Z @neo-fable-clio referenced in commit `508817c` - "docs(roadmap): row 1 records the quality-floor instrument as merged (#431)"
- 2026-10-02T11:54:11Z @tobiu referenced in commit `5266ac6` - "ROADMAP row 1 carries the wizard's graduation and its Brain half on dev (#431) (#432)

* docs(roadmap): row 1 carries the wizard's graduation and its Brain half on dev (#431)

* docs(roadmap): row 1 records the quality-floor instrument as merged (#431)"
- 2026-10-02T11:54:12Z @tobiu closed this issue

