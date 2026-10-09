---
id: 589
title: Seat-move review rows inherit default list styling
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-10-06T17:58:35Z'
updatedAt: '2026-10-09T12:11:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/589'
author: neo-gpt
commentsCount: 0
parentIssue: 13
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-09T12:11:09Z'
---
# Seat-move review rows inherit default list styling

## Context

The operator reported a visual defect in the live **System → This installation → reviewed seat-home move** screen on 2026-10-06: broad gray/olive row fills clash with the surrounding FM surface. The screenshot also shows an oversized heading/name/copy hierarchy. The exact application and compiled-theme revisions of that frame are unknown; its pixels are observation, not proof of a particular CSS cascade.

## The Problem

The new review list presents informational move rows as stock list items. This makes a sensitive review surface look unfinished and can suggest selectable/actionable rows even though selection is disabled. This is one bounded visual repair, independent of executing the seat migration.

## The Architectural Reality

Source checked at Institution `3b68995f1a5e9b0011feae3d2301dddd7abfa329`:

- `apps/agentos/view/system/SeatMoveList.mjs` extends `Neo.list.Base`, retains `neo-list`/`neo-list-item`, adds `fm-seat-move-row`, and sets `disableSelection: true`.
- [SeatRootContainer.scss](https://github.com/neomjs/neo-agent-institution/blob/3b68995f1a5e9b0011feae3d2301dddd7abfa329/resources/scss/src/apps/agentos/system/SeatRootContainer.scss#L54) defines row geometry but no local default/hover/active list background binding. Engine `resources/scss/src/list/Base.scss` paints those states through `--list-item-background-color*` and gives ordinary rows a pointer cursor.
- The same sheet uses `--fm-text-subhead` (line 15), `--fm-radius` (67), and `--fm-text-label` (83), which are absent from the current FM structural token block in `resources/scss/src/apps/agentos/Viewport.scss`. That block defines display/body/detail/micro/chrome roles and the contracted chip geometry. The copy-note and code paragraphs also need an explicit appropriate text role.
- `resources/scss/src/apps/agentos/fleet/instances/MemoryCandidateList.scss` demonstrates local list-token rebinding. `system/List.scss` demonstrates FM panel/ink/type styling. These are precedents to consume, not new global styling rules.

Design authority: [TOKENS.md §04](https://github.com/neomjs/neo-agent-institution/blob/3b68995f1a5e9b0011feae3d2301dddd7abfa329/apps/agentos/TOKENS.md#L45) says “A surface picks a role, never a pixel”; its surface census assigns `--fm-panel` to card/panel fills. #13 owns consuming this existing visual system. The operator explicitly requested a ticket for this observed screen.

## The Fix

Conform the seat-root review panel in its existing `SeatRootContainer.scss`: bind the stock list surface/state tokens locally so informational rows use the FM surface treatment at rest and while hovered/pressed; remove the stock selectable-row affordance; replace invalid font/radius references with the existing documented roles/geometry, including the copy note and refusal/reason text. Verify the compiled cascade before choosing the smallest override. Use the existing light/dark theme tokens, not hardcoded colors, a global engine reset, or new aliases that merely conceal the missing bindings.

Contract Ledger: N/A — restores the documented visual token contract; no API, payload, state vocabulary or interaction is added.

Decision Record impact: none. Structure map: N/A — presentation-only repair in an existing Institution System SCSS owner; no runtime or module placement changes.

## Acceptance Criteria

- [ ] In dark and light themes, the reviewed move rows use the intended FM surface with no inherited gray/olive stock-list slab at rest, hover or press. Computed-style evidence identifies the effective rule/token and shows these read-only rows do not suggest selection.
- [ ] Heading, seat name, copy guidance, source/destination paths, code/reason and status consume valid documented FM text roles; prose is readable, long paths/reasons wrap, and row geometry/radius resolves correctly.
- [ ] A before/after rendered receipt with multiple mixed-disposition rows and a refused row is recorded at a wide and narrow viewport using the real view and compiled theme. Use synthetic paths; no host move or consent is needed for this visual check.
- [ ] The change preserves all row text/dispositions and consent availability, and leaves other lists and the global engine theme unchanged.

Evidence required: L3 rendered view plus computed styles of the actual compiled themes. Source/CI alone does not prove the visual repair.

## Post-Merge Validation

Re-read this screen on the next selected packaged candidate carrying the fix; record candidate/theme revisions and an operator-visible screenshot. Installed validation remains with #12 and the broader rendering outcome #505.

## Out of Scope

Seat copy/verification/rebinding, root selection, memory adoption, consent/relaunch IPC, conflict recovery, and a redesign of the whole System view. The reported destination conflict is not diagnosed or changed by this styling ticket.

## Related

Parent: #13. Related: #505, #582, #585, #12.

Historical precedent: MC memory `afd23d46-5099-42e3-aa00-0013d917ef65`, session `52911fe4-68e5-4262-a176-d91c3b1cfb87`, records an earlier roster default-list-chrome incident. It motivates checking compiled assets and every interaction state; it is not proof that the current screen has stale assets.

unowned-rationale: operator-requested visual follow-up, queued for a separate presentation pickup while the active seat-migration work continues; this filing does not assign that work to the migration owner.

Origin Session ID: 01a110db-3db8-7c30-933e-883d691417d2
Retrieval Hint: "System seat-home reviewed move rows gray backgrounds stock list styling missing FM tokens"

Creation sweeps at 2026-10-06 17:58 UTC: live latest 20 open Institution issues read with author/labels/URL; no equivalent leaf. Historical searches found the owning token-conformance epic #13 only. Latest 30 all-state A2A messages: no overlapping styling claim. Three problem-framed MC queries found the earlier roster chrome precedent above, no current seat-panel decision; final freshness query returned unrelated records, not evidence of absence. KB ticket search could not resolve this new panel. Own-assignment sweep: #477 body read; its truthful-state outcome does not own this styling repair. No open Institution PR at filing.

## Timeline

- 2026-10-06T17:58:36Z @neo-gpt added the `bug` label
- 2026-10-06T17:58:36Z @neo-gpt added the `agent-os` label
- 2026-10-06T17:58:37Z @neo-gpt added the `ai` label
- 2026-10-06T17:58:37Z @neo-gpt added the `design` label
- 2026-10-06T17:59:15Z @neo-gpt added parent issue #13
- 2026-10-09T03:16:23Z @neo-fable-clio cross-referenced by #614
- 2026-10-09T03:48:07Z @neo-opus-grace cross-referenced by PR #619
- 2026-10-09T03:49:23Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-09T04:12:45Z @neo-opus-grace cross-referenced by PR #622
- 2026-10-09T04:18:04Z @neo-opus-grace referenced in commit `3d58392` - "chore(agentos): merge dev after #619 into the #589 branch, baselines re-stamped (#589)"
- 2026-10-09T05:12:52Z @neo-fable-clio cross-referenced by #505
- 2026-10-09T06:17:19Z @neo-opus-grace cross-referenced by #633
- 2026-10-09T06:36:59Z @neo-opus-grace cross-referenced by #635
- 2026-10-09T06:53:27Z @neo-opus-grace cross-referenced by #638
- 2026-10-09T12:11:09Z @tobiu referenced in commit `ad98d3b` - "fix(agentos): the seat-move rows sit on the FM panel and every text node picks a documented role (#589) (#622)

* fix(agentos): System's seat-move block reads by state, and its rows scroll in their own box (#614)

A committed move leaves no decision, so the plan box (line, copy-note, rows)
gives way to the receipt on the status line even though the consent record
still carries the rows. While the rows are a decision they scroll inside a
40vh box, so the plane list keeps the view's scroll and stays in reach.

* fix(agentos): a held retirement keeps the seat-move rows in their box, and the frames at the operator's window (#614)

Fleet start stays held while old folders are not all archived, so that state
keeps the plan line and rows (scrolling, 24vh) without the pre-move copy-note;
only a clean commit reduces the block to its receipt. New System goldens at
1552x850: committed (receipt, planes in view) and pending (rows scroll, the
first card's head reads without a scroll). Baselines re-stamped.

* fix(agentos): the seat-move rows sit on the FM panel and every text node picks a documented role (#589)

The sheet named --fm-text-subhead, --fm-text-label and --fm-radius, none of
which the FM token block defines, so the title, the copy-note and the seat
names fell back to the 16px default. The engine list's state tokens are bound
to --fm-panel and the rows lose the pointer: they are information, not a
selection.

* test(agentos): the reviewed-move fixture's rows carry the destination every plan row needs (#589)

* test(agentos): the reviewed move on the FM skin, both skins, wide and narrow (#589)

Mixed dispositions and a refused row, the row's surface read against a
--fm-panel probe at rest, hover and press in both skins, every capture at rest.
The System frames carrying the seat-root title are refreshed for its display
role; baselines re-stamped."
- 2026-10-09T12:11:10Z @tobiu closed this issue

