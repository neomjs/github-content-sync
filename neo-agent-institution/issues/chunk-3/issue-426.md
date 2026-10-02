---
id: 426
title: 'The compose form leaves the operator inbox 96 px, under one row'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-10-02T09:10:13Z'
updatedAt: '2026-10-02T11:34:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/426'
author: neo-opus-vega
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
closedAt: '2026-10-02T11:34:39Z'
---
# The compose form leaves the operator inbox 96 px, under one row

## Context

The cockpit's operator mailbox renders no rows. Measured in the e2e cockpit (fixture fleet, viewer `@e2e-operator`, 50 rows landed, pane state `rows`) on `dev` d52a90e and on #420's branch alike, at two viewports:

| Viewport | south strip | inbox pane | pane head | grid | grid body | compose form |
|---|---|---|---|---|---|---|
| 1280×720 | 211 px | 96 px (`min-height: 96px`, flex `1 1 0%`) | 23 px | 67 px | 48 px | 91 px (flex `0 1 auto`) |
| 1440×1800 | 635 px | 96 px | 23 px | 67 px | 48 px | 515 px |

A mailbox row is 84 px (`mailbox/Grid.scss`, the grid's `rowHeight`). The engine derives `availableRows = ceil(48 / 84) − 1 = 0` and `createViewData` renders nothing: the pane shows its head and the compose form over an empty rowgroup. The strip's height does not matter, because the inbox never leaves its 96 px floor: the compose form takes the rest at its natural height (about 560 px: recipients list 132 px fixed, message field 160 px, priority, wake switch, send), and the floor is below one row.

Found while writing #416's browser witness; filed first as a defect-note (2026-10-02 08:48Z). It blocks FM v1 row 4's step 2 (the viewer's own subject in rows, #414) and every pane walk of row 3's sitting sees it (#335).

## The Problem

The operator's own inbox is unreadable in the cockpit at any window height. The 96 px floor (`OperatorContainer.scss:14`, "the inbox floor") predates the designed 84 px row lattice (#40) and was written for a form that was meant to be a reveal: the mailbox design page (`apps/agentos/design/institution-mailbox-pane.html`, "Compose placement") puts the compose affordance in the pane header as a right-pinned chip, reachable without scroll, and leaves one slot open, inline reveal or an own south tab. The shipped pane opens the form permanently instead, and the inbox pays for it.

## The Architectural Reality

- `apps/agentos/view/fleet/mailbox/OperatorContainer.mjs`: a vbox of the inbox pane (`flex: 1`), the identity-warning marker (`flex: 'none'`, hidden until a conflated posture) and `OperatorComposeForm` (`flex: '0 1 auto'`, "shrinkable, never 'none' … the skin adds the internal scroll + the inbox floor").
- `resources/scss/src/apps/agentos/fleet/mailbox/OperatorContainer.scss:14`: `min-height: 96px` on the inbox pane.
- `apps/agentos/view/fleet/mailbox/ComposeForm.mjs`: recipients `List` `height: 132`, message `TextAreaField` `height: 160`, radios, switch, actions; it fires `compose` and shows the outcome rows the owner writes back.
- `apps/agentos/view/fleet/mailbox/Container.mjs`: the pane head is title + freshness chip (`fm-pane-head`); the design page's compose chip belongs beside the freshness chip.
- The design decision slot: inline reveal vs own south tab. This leaf ships the reveal; the chip "works in both worlds" per the page, so a later tab decision re-routes the chip, not the rows.

## The Fix

1. **The chip.** The pane head of the operator mailbox carries `✎ compose` as a right-pinned affordance chip (the page's class), after the freshness chip. A click toggles the form.
2. **The reveal.** `OperatorContainer` gains `composeOpen_` (default `false`); the compose form is hidden while it is false. A send keeps the form open with its outcome rows; the chip, or Escape inside the form, closes it. The identity-warning marker keeps its place above the form and its own visibility rule.
3. **The floor from the lattice.** Closed, the form is out of the layout and the inbox takes the strip. Open, the inbox keeps one designed row of context: `min-height` = head 23 + 84 + the pane's rhythm (≈ 119 px), stated in the SCSS as that sum, and the form scrolls internally in what is left (today's skin). At 1280×720 that is about 90 px of form; at 1440×1800 the form's whole natural height fits beside it.
4. **Goldens.** `pane-mailbox.png` re-captured (chip in the head, no form); stamp refreshed.

Nothing changes in the compose intents, the inbox read relay, or #420's scroll edge.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `✎ compose` chip in the operator mailbox's head | `OperatorContainer.mjs` (owner) rendering into the pane head; the design page's "Compose placement" | Toggles `composeOpen`; right-pinned, reachable without scroll; present only on the operator host (the per-agent subject host has no compose). | Absent compose form (no owner) → no chip. | the design page | `operatorContainer.spec.mjs`: chip present, toggles the form |
| `composeOpen_` on `OperatorContainer` | `OperatorContainer.mjs` | `false` by default; the form's `hidden` follows it; a `compose` intent's settled outcome does not close it; Escape in the form closes it. | — | docblock | `operatorContainer.spec.mjs` |
| the inbox floor | `OperatorContainer.scss` | With the form open, `min-height` = head 23 + gap 6 + the grid's chrome 20 + one lattice row 84 + slack 8 (141 px), derived from the constants named in the comment; closed, no floor is needed because the form is out of the layout. | — | the SCSS comment | the NL journey below, reading the engine's `availableRows`: `≥ 1` closed (measured: grid 147 px → 1) and `≥ 1` open (grid 101 px → 1) at 1280×720; dev read a 48 px body → 0 |

## Acceptance Criteria

- [ ] At 1280×720 with a 50-row window and the form closed, the operator mailbox renders at least one full lattice row by the engine's own reading (`availableRows ≥ 1`; measured 1: grid 147 px) and the first row is visible (red-first: dev reads a 48 px body, `availableRows 0`, no row). The strip's height bounds the count: two rows need a 169 px grid. *(Restated 2026-10-02 from "at least two rows" after the measurement; the first wording was not the strip's.)*
- [ ] With the form open in the same strip, the grid keeps one full row (`availableRows ≥ 1`; measured 1: grid 101 px from the 141 px floor) and the form has room to scroll in (measured 46 px).
- [ ] The compose form is hidden until the head's chip opens it; the chip closes it; a send leaves it open with its outcome rows; Escape inside the form closes it (unit).
- [ ] The compose intents, the recipient options and the inbox read relay are unchanged (existing arms stay green).
- [ ] `pane-mailbox.png` re-captured; `check-visual-baselines` matches.
- [ ] *(L4, the row-4 sitting, #414)* step 2 reads the viewer's own subject in rows on the installed candidate. Not claimed by this leaf.

## Out of Scope

- Compose as its own south tab (the design page's open slot; the chip routes there if the team decides it).
- The compose form's internals and its fixed field heights.
- The south strip's dock proportions (the document's).

## Avoided Traps

- **Raising the floor alone.** At 720 the strip is 211 px; a two-row floor leaves the always-open form 8 px. The reveal is what frees the strip; the floor only guards the open state.
- **Shrinking the form's fields to fit.** 132 and 160 px are the form's designed sizes; the fix is when the form is shown, not how small it can get.
- **A taller strip by default.** The proportions are the operator's document; the pane must read in the strip it is given.

## Related

#414 (parent, row 4 step 2) · #335 (rows 3 and 4 sittings) · #416 / PR #420 (the pane's boot drain; found there) · #40 (the row lattice) · #10 (the design-led surface) · `apps/agentos/design/institution-mailbox-pane.html`

Live latest-open sweep: the 20 newest open Institution issues at 09:08Z (#425 … #13) and the A2A inbox's last 12 messages at 09:09Z: no equivalent ticket or claim (Grace asked at 08:58Z whether it rides #416 or is a leaf here; answered: here). `query_raw_memories` on the symptom at 09:08Z: the drain's and the form's origins, no prior decision on the floor. Own open assignments: #416 and neo #19356.

Origin Session ID: 60d9be31-4233-40fd-9b07-6ee6a9ebf6cf
Retrieval Hint: "operator mailbox compose form reveal chip inbox floor 96px 84px row zero rows drawer"




## Timeline

- 2026-10-02T09:10:13Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-02T09:10:15Z @neo-opus-vega added the `bug` label
- 2026-10-02T09:10:16Z @neo-opus-vega added the `agent-os` label
- 2026-10-02T09:10:16Z @neo-opus-vega added the `ai` label
- 2026-10-02T09:10:16Z @neo-opus-vega added the `design` label
- 2026-10-02T09:10:39Z @neo-opus-vega added parent issue #414
- 2026-10-02T09:28:11Z @neo-opus-vega cross-referenced by PR #428
- 2026-10-02T09:30:47Z @neo-opus-vega cross-referenced by #429
- 2026-10-02T09:33:57Z @neo-opus-vega referenced in commit `16d539c` - "test(agentos): the visual baseline stamp follows the compose reveal's committed inputs (#426)"
- 2026-10-02T09:57:43Z @neo-opus-vega referenced in commit `c4064d4` - "test(agentos): the rows journey reads the engine's own row count; one full row is the guarantee the 1280×720 strip gives (#426)"
- 2026-10-02T10:07:37Z @neo-gpt-sophie cross-referenced by PR #420
- 2026-10-02T10:19:53Z @neo-opus-vega referenced in commit `fdfcfe9` - "fix(agentos): compose is a reveal behind the inbox head's chip, so the operator's rows get the strip (#426)"
- 2026-10-02T10:19:53Z @neo-opus-vega referenced in commit `fda7bf4` - "test(agentos): the visual baseline stamp follows the rebase onto #420 (#426)"
- 2026-10-02T10:19:54Z @neo-opus-vega referenced in commit `4cbdc6c` - "test(agentos): the rows journey reads the engine's own row count; one full row is the guarantee the 1280×720 strip gives (#426)"
- 2026-10-02T11:34:39Z @tobiu closed this issue
- 2026-10-02T11:34:40Z @tobiu referenced in commit `22cf097` - "fix(agentos): compose is a reveal behind the inbox head's chip, so the operator's rows get the strip (#426) (#428)

* fix(agentos): compose is a reveal behind the inbox head's chip, so the operator's rows get the strip (#426)

* test(agentos): the visual baseline stamp follows the rebase onto #420 (#426)

* test(agentos): the rows journey reads the engine's own row count; one full row is the guarantee the 1280×720 strip gives (#426)"
- 2026-10-02T11:38:26Z @neo-opus-grace cross-referenced by #414
- 2026-10-02T11:41:40Z @neo-opus-vega cross-referenced by #12
- 2026-10-02T12:20:29Z @neo-opus-grace cross-referenced by #436
- 2026-10-02T12:47:45Z @neo-gpt-sophie cross-referenced by PR #434
- 2026-10-02T13:56:00Z @neo-gpt-sophie cross-referenced by PR #437

