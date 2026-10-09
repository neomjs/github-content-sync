---
id: 620
title: 'First-run polish: unbroken possessive, the app''s window, verb-only link'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-09T03:56:49Z'
updatedAt: '2026-10-09T03:56:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/620'
author: neo-fable
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# First-run polish: unbroken possessive, the app's window, verb-only link

## Context

Clio's stranger read of the shipped frames (#535, comment 6073322534, 2026-10-09, after #613 merged) found two one-line follow-ups and one optional, and routed them to the row-1 steward rather than to #613 (approved, frozen). The frames: 5981064896; the operator's word on the promise line: 6072889640.

## The Problem

1. At the door's measure the promise line breaks between *other's* and *work* in both themes — a possessive split across the fold, a property of the measure, not of the words.
2. The token block's hint says "the vessel's own window": *vessel* is our word; the door's reader has met "the app" and nothing else.
3. The second door underlines the whole sentence "Joining a team that already runs one? Connect to it"; linking only the verb reads cleaner (optional, same file).

## The Architectural Reality

`apps/agentos/view/home/Container.mjs` — `PROMISE_LINE` and the first-run block (the primary button, the setup line, the Connect link); `apps/agentos/view/setup/CreateContainer.mjs` — the token block's `credential-help` text; `resources/scss/src/apps/agentos/home/Container.scss` (`.fm-home-link`); the goldens `home-first-run*.png` and `setup-card-create*.png` in `FleetCockpitVisual.spec.mjs` on the pinned host, stamped by `check-visual-baselines`.

## The Fix

- A no-break space joins `other's` and `work` inside `PROMISE_LINE` — in the string, so every measure keeps the pair (`text-wrap: pretty` would be the stylesheet alternative; the string is the smaller change).
- The hint reads "Entered in the app's own window and kept as an owner-only file; this card only ever shows its path."
- The second door becomes one line: the question as plain text, `Connect to it` the only link. The `connect-plane` reference and the `fm-home-connect` class stay, so the route to the card's Connect door is unchanged.
- Six goldens re-captured on the pinned host (Home first run in both skins; the Create front, Details, 720 px, light); the stamp re-taken.

## Acceptance Criteria

- [ ] The promise line renders "other's work" on one line at the door's measure in both skins (golden, and the unit assertion on the string).
- [ ] The token block's hint names the app, not the vessel.
- [ ] Home's second door shows the question as text and "Connect to it" as the only link; clicking it still opens the card's Connect door (e2e).
- [ ] The six goldens are re-captured and `check-visual-baselines` is green.

## Out of Scope

The already-running variant of the promise and the provisioned-server choice (row-1 gap lines on #351); the "Set up yours:" echo-fix (considered and declined in the read).

## Related

#535 (closed by #613) · #351 (row 1) · #613

Sweeps (2026-10-09 03:5xZ): the latest-open Institution queue (#618, #616, #614, #610, #603, #602 — none covers the first run's wording); A2A: Clio's read DM and my lane-intent 33dec554 (no competing leaf); Memory Core: this session's turn memories. Structure map: N/A (no new file). Decision Record impact: `none`.

Origin Session ID: 2ea2911e-ebbd-49be-9471-3e77369ca2b5
Retrieval Hint: "first-run promise line no-break space app's own window Connect to it link"

## Timeline

- 2026-10-09T03:56:49Z @neo-fable assigned to @neo-fable
- 2026-10-09T03:56:50Z @neo-fable added the `enhancement` label
- 2026-10-09T03:56:51Z @neo-fable added the `agent-os` label
- 2026-10-09T03:56:51Z @neo-fable added the `ai` label
- 2026-10-09T03:57:43Z @neo-fable added parent issue #351
- 2026-10-09T03:58:21Z @neo-fable cross-referenced by PR #621

