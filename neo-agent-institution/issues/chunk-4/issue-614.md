---
id: 614
title: System's seat-move block outlives the move and hides the plane list
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-10-09T03:16:22Z'
updatedAt: '2026-10-09T04:15:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/614'
author: neo-fable-clio
commentsCount: 0
parentIssue: 505
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-09T04:15:58Z'
---
# System's seat-move block outlives the move and hides the plane list

## Context

The operator reported on 2026-10-09 (~03:10Z, screenshot in chat, the installed app at `127.0.0.1:3102`, Candidate F) that System › This installation shows the full consented seat-move list — twelve rows, `from` → `to`, each `move consented` — plus the pre-move copy-note *"The shell copies and checks each seat home before changing the root…"*, although the status line above it already reads `Root move committed · /Users/…/.neo-ai/agents`. The list runs past the window's bottom edge and nothing scrolls, so the System view's own content (the plane cards, the maintenance strip) is no longer reachable. His words: the team "dropped an unstyled migration folder list on top of the real content… not even scrollable".

Observation: the capture shows the committed state with the list and the copy-note visible and the plane list absent below the fold. Inference, verified at source below: the block is not gated on the move's outcome and cannot scroll.

Design authority: #505 (every important cockpit view is *reachable*), #12 (the shell's UX spec), the operator's report above. The surface's author (#582 / PR #585) is informed by A2A.

## The Problem

The consent review (#582, PR #585, merged 2026-10-06) was built for one state: an installed shell that proposes moving its seats to the default seat root, before the operator consents. In that state the twelve rows and the copy-note are the decision. After the move is committed there is no decision left — `Review move` and `Move seats and relaunch` are hidden by design — yet the block keeps rendering the whole list and the pre-move guidance in the wrong tense, and because it never scrolls it takes the System view with it. On every machine the team runs today the move is committed (all twelve rows consented tonight), so the surface's only remaining appearance is this wrong one; a fresh outside install never has a legacy root and never needs it at all.

## The Architectural Reality

- `apps/agentos/view/system/Container.mjs:144` lays System out as `vbox · align stretch`; `:181` mounts `SeatRootContainer` with `flex: 'none'`; the plane list (`PlaneList`, reference `planes`) is the only `flex: 1` child and owns the scroll (`resources/scss/src/apps/agentos/system/List.scss:16` `overflow: auto`; `Container.scss:3` records the rule). A `flex: none` block that grows with twelve rows pushes the plane list and the maintenance strip below the viewport; no ancestor scrolls.
- `apps/agentos/view/system/SeatRootContainer.mjs` renders the plan box from presence alone: `planVisible = !!plan || Array.isArray(pending?.rows)` (render tail, ~line 330); the `outcome?.state === 'committed'` branch (~307–311) sets the status line and leaves the plan box untouched. The copy-note is a static child of `plan-summary` (items, ~lines 100–110); `SeatMoveList` is mounted `flex: 'none'` with no height bound.
- `resources/scss/src/apps/agentos/system/SeatRootContainer.scss` has no `overflow` or `max-height` rule (only `overflow-wrap`).
- `canReviewMove()` (~222–231) returns false once `snapshot.pending` is set or `root.origin === 'moved'`, so the committed state offers only `Recheck status`.
- `SeatMoveList` extends `Neo.list.Base` with `disableSelection: true`; its stock row styling is #589.

## The Fix

Render the block by the move's state, and never let it take the view.

1. **Cleanly committed:** the plan box (copy-note + list) is hidden; the status line stays the receipt it already is (`Root move committed · <root>`); if the snapshot carries the row count, the line may say `· 12 seats moved`, nothing invented. **Committed with retirement held** (Fleet start held until the old folders are archived): the rows stay — they are the reasons the hold names, per seat — scrolling, without the copy-note; #585's NL spec asserts exactly this state (`SeatRootConsentNL.spec.mjs`, the held-retirement arm). *Amended 2026-10-09 after Grace's delta on PR #619: the first filing hid the rows in the held state too.*
2. **Pending or reviewed (the one state the list is for):** the copy-note and the list render, and the list scrolls inside its own box (a bounded `max-height` with `overflow: auto` on the move list or `.fm-seat-root-plan`), so the plane list and the maintenance strip stay reachable at the operator's window height. The System view's scroll ownership (`List.scss`) is unchanged.
3. **The copy-note** renders only while a move can be reviewed or is pending — it describes what the shell *will* do.

Files: `apps/agentos/view/system/SeatRootContainer.mjs` (`sync()` / render tail), `resources/scss/src/apps/agentos/system/SeatRootContainer.scss`, the SeatRootContainer unit spec, the three System goldens from #585.

Decision Record impact: aligned-with ADR 0034 §2.3 item 11 (the seat-root move as a named broker — the broker, consent record and relaunch are untouched).

## Acceptance Criteria

- [ ] AC-1 — With `outcome.state === 'committed'` and no held retirement the plan box is hidden and the status line reads the receipt; with `outcome.retirement.state === 'held'` the rows stay visible and the copy-note is hidden; the SeatRootContainer unit spec pins both cases (red first on the current source).
- [ ] AC-2 — With consented pending rows (outcome `null`) or a reviewed plan, twelve rows render inside a box that scrolls within the System view; the plane list and the maintenance strip remain reachable at 1552 × 850 (the operator's window), proven by a rendered check, not by inspection.
- [ ] AC-3 — The copy-note is absent in the committed state and present in the reviewable/pending states.
- [ ] AC-4 — The System goldens are re-captured from a full visual run and stamped; the committed frame shows plane cards below the receipt line.
- [ ] AC-5 (post-merge only) — On the next #12 cut the operator's System view shows plane cards again; one receipt on #505's installed check.

## Out of Scope

The mover and consent broker (Brain #901, Institution #584, ADR 0034 item 11), the consent flow itself, any rollback verb (none exists in the UI; none is added), and #589's row styling (sibling leaf; fold it in only if the builder is already in the sheet).

## Avoided Traps

- **Deleting the block** (the first instinct on seeing it): a future root move needs the consent surface — the broker is a named ADR 0034 item. The defect is state-gating, not existence.
- **Scrolling the whole System vbox instead of the list:** breaks the recorded rule that the plane list owns the scroll and would double-scroll the plane cards.
- **Styling the rows as the fix:** #589 makes the rows look right; it does not stop the block from hiding the view.

## Related

Parent #505 · #582 / PR #585 (the surface) · #584 (the mover) · #589 (row styling) · #12 · Brain #901 · neomjs/neo#19428 / #19429 (ADR 0034 item 11).

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-10-09T03:14:51Z; nearest is #589 (styling only), no equivalent. A2A claim sweep: last 30 messages, no claim on System or the seat-root block (Emmy's #600 claim is a different surface). Memory Core sweep: PR #585's build record (2026-10-06) sets plan visibility from plan/pending presence with no committed-state gate and records no decision that the list should persist after the move. Own-assignment sweep: #505 (parent), #507, #351 — no overlap. Structure map: N/A, existing files, no placement decision.

unowned-rationale: filed by the design seat; the build is a builder's self-select (offered to Grace and Ada by A2A; the surface's author informed).

Origin Session ID: 3302ae6e-96e0-434c-a524-363820bc9f1b
Retrieval Hint: "System seat-move block outlives the committed move · SeatRootContainer planVisible · plane list unreachable"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 3302ae6e-96e0-434c-a524-363820bc9f1b

## Timeline

- 2026-10-09T03:16:23Z @neo-fable-clio added the `bug` label
- 2026-10-09T03:16:23Z @neo-fable-clio added the `agent-os` label
- 2026-10-09T03:16:24Z @neo-fable-clio added the `ai` label
- 2026-10-09T03:16:24Z @neo-fable-clio added the `design` label
- 2026-10-09T03:16:37Z @neo-fable-clio added parent issue #505
- 2026-10-09T03:36:24Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-09T03:48:07Z @neo-opus-grace cross-referenced by PR #619
- 2026-10-09T03:52:55Z @neo-fable-clio cross-referenced by #505
- 2026-10-09T03:56:50Z @neo-fable cross-referenced by #620
- 2026-10-09T04:15:58Z @tobiu referenced in commit `32627ab` - "fix(agentos): System's seat-move block reads by state, and its rows scroll in their own box (#614) (#619)

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
first card's head reads without a scroll). Baselines re-stamped."
- 2026-10-09T04:15:58Z @tobiu closed this issue
- 2026-10-09T06:16:36Z @neo-fable-clio cross-referenced by #632

