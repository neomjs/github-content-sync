---
id: 644
title: The setup door's foot fade shows only while the door overflows
state: OPEN
labels:
  - bug
  - ai
  - design
assignees:
  - neo-fable
createdAt: '2026-10-09T12:56:20Z'
updatedAt: '2026-10-09T12:56:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/644'
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
# The setup door's foot fade shows only while the door overflows

## Context

A leaf of #351 (row 1, the setup card), from two reads of PR #621's row captures at 916e7b4: Sophie's RA-1 (review 5470219687) found the witness row's second reason line dimmed in the new goldens, and Clio's design read (A2A 7c4daf6c, defect-note f41080eb) named the cause from the stylesheet: the Create door's foot fade is unconditional.

## The Problem

`.fm-setup-create::after` (`resources/scss/src/apps/agentos/setup/CreateContainer.scss:9–20`) is a sticky 28 px gradient at the door's foot that says "the rows continue". It paints whether or not the door overflows, so when the ledger ends inside the viewport the last row's second line sits under the fade and reads as faint — a signal with nothing to signal. PR #621 keeps its captures honest by scrolling the row into the reading area and asserting the boundary; the design defect stays.

## The Architectural Reality

- The door is the scroll surface: `.fm-setup-door { overflow-y: auto }` (`Panel.scss`), the card bounded at `max-height: 62vh`; the fade is the door's own `::after` with `margin-top: -28px`, so it overlays the last 28 px of content at the scrollport's foot.
- Nothing reads the overflow state today — no class on the door, no measurement in `CreateContainer.mjs`.
- The visual arm "the witness row with two exits" asserts, since #621, that the row ends above the fade before each capture (`clearance ≥ 0`); it will keep passing once the fade is conditional.

## The Fix

The fade shows only while the door can still scroll down. Two shapes, the implementer picks after one measurement in the installed app:

1. **Stylesheet-only:** the door becomes a scroll-state container (`container-type: scroll-state`) and the fade moves to a child element at the door's foot (the pseudo-element of the container itself cannot be queried), gated by `@container scroll-state(scrollable: bottom)`. No JavaScript, follows the scroll position by itself.
2. **Measured:** `CreateContainer` toggles `is-overflowing` on the door after each render and on resize (`scrollHeight > clientHeight`), and the `::after` gates on that class. One observer, one class.

Either way the fade's look is unchanged; only its presence follows the overflow.

## Acceptance Criteria

- [ ] AC-1 With the ledger under Details ending inside the door, no fade paints and the last row's second line reads at full ink (visual arm, both skins).
- [ ] AC-2 With the door overflowing, the fade paints at the foot and leaves once the door is scrolled to its end (visual arm, or a computed assertion on the fade's opacity/height).
- [ ] AC-3 The witness-row arm's reading-area guard (`clearance ≥ 0`) stays green; `check-visual-baselines` green.

## Out of Scope

The row's own metrics (#620 settled them: the row clips nothing, the reason ends inside it); the Home door; the served path's own scrolling.

## Related

#351 (row 1) · #620 / PR #621 (the captures' guard) · #613 (the Create front) · #535

Sweeps (2026-10-09 12:5xZ): the latest 20 open Institution issues — none on the door's fade; `gh issue list --search "fade gradient setup door"` — none; A2A: Clio's defect-note f41080eb is this leaf's origin, no claim; Memory Core: this session's turn memories only. Structure map: N/A (existing files). Decision Record impact: `none`.

Origin Session ID: 2ea2911e-ebbd-49be-9471-3e77369ca2b5
Retrieval Hint: "setup door foot fade conditional overflow scroll-state sticky gradient last row dimmed"


## Timeline

- 2026-10-09T12:56:21Z @neo-fable assigned to @neo-fable
- 2026-10-09T12:56:21Z @neo-fable added the `bug` label
- 2026-10-09T12:56:21Z @neo-fable added the `ai` label
- 2026-10-09T12:56:21Z @neo-fable added the `design` label
- 2026-10-09T12:56:58Z @neo-fable cross-referenced by PR #621
- 2026-10-09T12:57:03Z @neo-fable added parent issue #351

