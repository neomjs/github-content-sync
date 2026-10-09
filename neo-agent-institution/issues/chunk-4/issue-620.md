---
id: 620
title: 'First-run polish: unbroken possessive, the app''s window, verb-only link'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-09T03:56:49Z'
updatedAt: '2026-10-09T14:07:34Z'
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
closedAt: '2026-10-09T14:07:34Z'
---
# First-run polish: unbroken possessive, the app's window, verb-only link

## Context

Clio's stranger read of the shipped frames (#535, comment 6073322534, 2026-10-09, after #613 merged) found two one-line follow-ups and one optional, and routed them to the row-1 steward rather than to #613 (approved, frozen). The frames: 5981064896; the operator's word on the promise line: 6072889640.

Widened 2026-10-09 12:xxZ: Grace's defect-note (A2A 32eb3a75, 04:10Z) found the visual arm "the witness row with two exits" red on clean `dev` after #613 — it is #613's surface, so its repair rides this PR (Clio's routing, DM 217f18b7).

## The Problem

1. At the door's measure the promise line breaks between *other's* and *work* in both themes — a possessive split across the fold, a property of the measure, not of the words.
2. The token block's hint says "the vessel's own window": *vessel* is our word; the door's reader has met "the app" and nothing else.
3. The second door underlines the whole sentence "Joining a team that already runs one? Connect to it"; linking only the verb reads cleaner (optional, same file).
4. The visual arm "the witness row with two exits" expects three `.fm-setup-preset` cards at boot and finds none: since #613 the presets sit under the `where` block's **Other choices** fold and the ledger's rows behind **Details**, so every full `test-visual` run on `dev` is red by one arm (the suite is not in CI), which blocks honest re-captures for everyone.

## The Architectural Reality

`apps/agentos/view/home/Container.mjs` — `PROMISE_LINE` and the first-run block (the primary button, the setup line, the Connect link); `apps/agentos/view/setup/CreateContainer.mjs` — the token block's `credential-help` text; `resources/scss/src/apps/agentos/home/Container.scss` (`.fm-home-link`); the goldens `home-first-run*.png` and `setup-card-create*.png` in `FleetCockpitVisual.spec.mjs` on the pinned host, stamped by `check-visual-baselines`. The witness-row arm's route through the front is the e2e walk's own (`FleetSetupCard.spec.mjs`): the token, then Other choices, then Details.

## The Fix

- A no-break space joins `other's` and `work` inside `PROMISE_LINE` — in the string, so every measure keeps the pair (`text-wrap: pretty` would be the stylesheet alternative; the string is the smaller change).
- The hint reads "Entered in the app's own window and kept as an owner-only file; this card only ever shows its path."
- The second door becomes one line: the question as plain text, `Connect to it` the only link. The `connect-plane` reference and the `fm-home-connect` class stay, so the route to the card's Connect door is unchanged.
- Six goldens re-captured on the pinned host (Home first run in both skins; the Create front, Details, 720 px, light); the stamp re-taken.
- The witness-row arm walks the front's route: the token's button, Other choices, the first preset's `choose`, Details, then the ledger's chips — the three row goldens stay as they are unless the run says otherwise.

## Acceptance Criteria

- [ ] The promise line renders "other's work" on one line at the door's measure in both skins (golden, and the unit assertion on the string).
- [ ] The token block's hint names the app, not the vessel.
- [ ] Home's second door shows the question as text and "Connect to it" as the only link; clicking it still opens the card's Connect door (e2e).
- [ ] The six goldens are re-captured and `check-visual-baselines` is green.
- [ ] The arm "the witness row with two exits" is green on a full `test-visual` run at the PR head on top of current `dev`.

## Out of Scope

The already-running variant of the promise and the provisioned-server choice (row-1 gap lines on #351); the "Set up yours:" echo-fix (considered and declined in the read).

## Related

#535 (closed by #613) · #351 (row 1) · #613 · #615 (the rebase conflict on the stamp)

Sweeps (2026-10-09 03:5xZ): the latest-open Institution queue (#618, #616, #614, #610, #603, #602 — none covers the first run's wording); A2A: Clio's read DM and my lane-intent 33dec554 (no competing leaf); Memory Core: this session's turn memories. Structure map: N/A (no new file). Decision Record impact: `none`.

Origin Session ID: 2ea2911e-ebbd-49be-9471-3e77369ca2b5
Retrieval Hint: "first-run promise line no-break space app's own window Connect to it link witness row arm Other choices Details"


## Timeline

- 2026-10-09T03:56:49Z @neo-fable assigned to @neo-fable
- 2026-10-09T03:56:50Z @neo-fable added the `enhancement` label
- 2026-10-09T03:56:51Z @neo-fable added the `agent-os` label
- 2026-10-09T03:56:51Z @neo-fable added the `ai` label
- 2026-10-09T03:57:43Z @neo-fable added parent issue #351
- 2026-10-09T03:58:21Z @neo-fable cross-referenced by PR #621
- 2026-10-09T12:27:42Z @neo-fable referenced in commit `32e1fc0` - "fix(agentos): the first run's words hold — an unbroken possessive, the app's own window, the verb as the link (#620)

A no-break space keeps "other's work" on one line at the door's measure; the token block's hint
names the app, not the vessel; Home's second door shows its question as text with "Connect to it"
as the only link. Six goldens re-captured on the pinned host."
- 2026-10-09T12:27:43Z @neo-fable referenced in commit `916e7b4` - "test(agentos): the witness row's arm walks the Create front's own route (#620)

Since #613 the setup card's presets sit under the where block's Other choices
and the ledger's rows behind Details, so the visual arm "the witness row with
two exits" counted zero presets at boot and every full test-visual run on dev
was red by one arm (the suite is not in CI). The arm now takes the e2e walk's
route — the token's button, Other choices, the first preset's choose, Details,
then the ledger's chips — and asserts that the wrapped reason ends inside its
row before the capture (scrollHeight equals clientHeight, the reason's bottom
at or above the row's).

The three row goldens are re-captured from a full run: the row is one pixel
shorter than the 2026-10-05 capture because #613 set the ledger's rows in the
detail font (11 px over 15.4 px lines), the metrics the approved Details golden
already carries; the row's box holds the second line with six pixels to spare."
- 2026-10-09T12:55:25Z @neo-fable referenced in commit `5d3464b` - "fix(agentos): the first run's words hold — an unbroken possessive, the app's own window, the verb as the link (#620)

A no-break space keeps "other's work" on one line at the door's measure; the token block's hint
names the app, not the vessel; Home's second door shows its question as text with "Connect to it"
as the only link. Six goldens re-captured on the pinned host."
- 2026-10-09T12:55:25Z @neo-fable referenced in commit `97ebf51` - "test(agentos): the witness row's arm walks the Create front's own route (#620)

Since #613 the setup card's presets sit under the where block's Other choices
and the ledger's rows behind Details, so the visual arm "the witness row with
two exits" counted zero presets at boot and every full test-visual run on dev
was red by one arm (the suite is not in CI). The arm now takes the e2e walk's
route — the token's button, Other choices, the first preset's choose, Details,
then the ledger's chips — and asserts that the wrapped reason ends inside its
row before the capture (scrollHeight equals clientHeight, the reason's bottom
at or above the row's).

The three row goldens are re-captured from a full run: the row is one pixel
shorter than the 2026-10-05 capture because #613 set the ledger's rows in the
detail font (11 px over 15.4 px lines), the metrics the approved Details golden
already carries; the row's box holds the second line with six pixels to spare."
- 2026-10-09T12:55:26Z @neo-fable referenced in commit `623e2d0` - "test(agentos): the witness row's captures come from its unobscured reading area (#620)

The Create door's foot fades under a sticky 28 px gradient, and the previous
captures took the row where that fade dimmed its second line. Before each of
the three captures the arm now scrolls the row to the centre of the door's own
scroll surface and asserts the boundary: the row's box clips nothing, the
wrapped reason ends inside it, and the row ends above the fade. The fade stays
as designed; the three goldens are re-captured from a full run with the second
line painted whole."
- 2026-10-09T12:56:21Z @neo-fable cross-referenced by #644
- 2026-10-09T14:07:34Z @tobiu referenced in commit `3bfe24f` - "fix(agentos): the first run's words hold — an unbroken possessive, the app's own window, the verb as the link (#620) (#621)

* fix(agentos): the first run's words hold — an unbroken possessive, the app's own window, the verb as the link (#620)

A no-break space keeps "other's work" on one line at the door's measure; the token block's hint
names the app, not the vessel; Home's second door shows its question as text with "Connect to it"
as the only link. Six goldens re-captured on the pinned host.

* test(agentos): the witness row's arm walks the Create front's own route (#620)

Since #613 the setup card's presets sit under the where block's Other choices
and the ledger's rows behind Details, so the visual arm "the witness row with
two exits" counted zero presets at boot and every full test-visual run on dev
was red by one arm (the suite is not in CI). The arm now takes the e2e walk's
route — the token's button, Other choices, the first preset's choose, Details,
then the ledger's chips — and asserts that the wrapped reason ends inside its
row before the capture (scrollHeight equals clientHeight, the reason's bottom
at or above the row's).

The three row goldens are re-captured from a full run: the row is one pixel
shorter than the 2026-10-05 capture because #613 set the ledger's rows in the
detail font (11 px over 15.4 px lines), the metrics the approved Details golden
already carries; the row's box holds the second line with six pixels to spare.

* test(agentos): the witness row's captures come from its unobscured reading area (#620)

The Create door's foot fades under a sticky 28 px gradient, and the previous
captures took the row where that fade dimmed its second line. Before each of
the three captures the arm now scrolls the row to the centre of the door's own
scroll surface and asserts the boundary: the row's box clips nothing, the
wrapped reason ends inside it, and the row ends above the fade. The fade stays
as designed; the three goldens are re-captured from a full run with the second
line painted whole."
- 2026-10-09T14:07:35Z @tobiu closed this issue

