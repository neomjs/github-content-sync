---
id: 712
title: Agent detail reveals with the selection and hides when it clears
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-gemini-julia
createdAt: '2026-10-16T14:34:55Z'
updatedAt: '2026-10-17T00:14:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/712'
author: neo-fable-mnemosyne
commentsCount: 1
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
closedAt: '2026-10-17T00:14:34Z'
---
# Agent detail reveals with the selection and hides when it clears

## Context

Found by the `memories` design sweep on the installed candidate of 2026-10-14 (Institution `afabdef6` / Brain `e219267e` / Engine `d52f0e3d`) — receipt on #505 ([6217579463](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6217579463), section 5 q1) — and ruled by the design seat on #505 ([6217717533](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6217717533), q1). A leaf of #505 for the cockpit steward; filed by the sweep's holder per the planner's routing in that read.

## The Problem

Selecting a roster card in the Overview perspective pins the Agent detail column: the dock item `detail` flips `autoHidden` true → false, the right edge zone (extent 0.25) opens as a column, and the center drops from 1332 to 988 px (window 1400 × 1109 CSS px; measured by `get_dom_rect` on the roster pane and the lower strip before and after the click). Nothing restores `autoHidden` when the selection clears: a `selectionModel.deselectAll` left the column pinned, the dock document stayed changed, and only the dock's own verb (`setItemAutoHidden detail true`) returned it. The handler's JSDoc says *reveal the auto-hidden detail pane*; the mechanism is a pin. Every reading in the lower strip (Memories, Tasks, Mailbox, Catch up) therefore pays 344 px of width for a column the operator did not pin, and a stranger cannot tell why the strip narrowed.

## The Architectural Reality

- `apps/agentos/view/fleet/cockpit/Controller.mjs` `onAgentSelect` (lines 245–261 at dev `4fd5945`): `applySelection(record)`, then `if (cockpit.dockModel?.items?.detail?.autoHidden)` → `cockpit.applyDockZoneOperation({operation: 'setItemAutoHidden', itemId: 'detail', autoHidden: false})` + `onDockZoneDocumentChange(result.document)`. `applySelection` (484–504) writes the provider pair `selectedAgentId` / `selectedAgentIdentity` through the view's ONE selection-write site. No selection-clearing path touches the dock document. The same un-hide pattern serves `onAddAgentRequest` (267–275, `defineAgent`).
- The dock's explicit act is its pin verb (the strip header's `unpin` / `setItemPinned`); `setItemAutoHidden` is the mechanism the reveal borrowed. The engine's dock operations live in neomjs/neo `src/dashboard/dock/model/Operations.mjs` (`setItemAutoHidden` 638, `setItemPinned`); a `RevealOverlay` exists for auto-hidden rail items (`neo-dashboard-dock-reveal-overlay-1` in the live tree — empty while the column is pinned).
- Design authority: "Ruling: a reveal, bound to the selection. Detail shows while an agent is selected and returns to auto-hidden when the selection clears (Escape, deselect, a roster refresh that drops the record); it stays only through the dock's own pin verb — the one explicit act that may outlive a selection. A reading in the lower strip does not hide detail by itself; the room a reading needs is #506 Fix 3's question (Maximize), not the pin's — keep the two apart." — the design seat, #505 6217717533 (q1); and the handler's own JSDoc, *reveal the auto-hidden detail pane*.

## The Fix

`detail.autoHidden` follows `selectedAgent != null` unless the item is pinned. The reveal on select stays as it is; on every selection-clearing path — Escape, deselect, a roster refresh that drops the record, an instance switch — the controller restores `autoHidden: true` unless the operator pinned the item through the dock's verb. Read the pin from the dock document's own pin state, never infer it from `autoHidden === false` (that is the bug's state). One site owns the dock write, beside the one selection-write site. A reading in the lower strip never hides detail by itself.

## Acceptance Criteria

- [ ] AC-1 Unit arm on the cockpit controller: `detail.autoHidden` follows `selectedAgent != null` unless the item is pinned — the dock document is the witness for select, Escape, deselect and a roster refresh that drops the selected record.
- [ ] AC-2 A perspective restore or a reload keeps a pinned detail and drops an unpinned one.
- [ ] AC-3 NL/e2e arm on a source build: the center drop (1332 → 988 px at 1400 px) occurs only while an agent is selected or detail is pinned; after Escape or deselect the center returns and the dock document equals the pre-selection document.
- [ ] AC-4 (post-merge, installed) On the next #12 cut: one selection → reveal → Escape, the dock document before and after, one receipt on #505.

## Out of Scope

The Memories reader's room (#506 Fix 3, the dock's Maximize); the Review perspective's missing detail column after a saved-session restore (the design/outbound seat's 2026-10-13 defect-note, routed to #12); the reveal's animation.

## Avoided Traps

Hiding detail when a reading opens in the lower strip (two questions conflated — the ruling keeps them apart); inferring "pinned" from `autoHidden === false`; a second selection-write site.

## Related

#505 (parent), #506 (the Memories reading surface), #12 (the installed cut), the sweep receipt 6217579463, the design read 6217717533, neomjs/neo `src/dashboard/dock/model/Operations.mjs`.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-16 14:33Z; no equivalent. A2A in-flight claim sweep at 14:33Z: none on this scope. MC sweep: "Agent detail auto-hidden reveal on roster selection setItemAutoHidden pin cockpit Controller onAgentSelect", 5 results — the July 2026 precedent that un-hides `defineAgent` on request, no decision on re-hiding. Own-assignment sweep: 6 open, none overlapping (#391 and #476 are the detail's panes, not its reveal).

Decision Record impact: none (a view contract, not an ADR).

unowned-rationale: filed from the #505 sweep's gap list per the design seat's routing (q1 → the cockpit steward); the steward claims it through a planner, per #505's rule.

Origin Session ID: 8be1b284-089e-4f28-94be-d1a45be666b8

Retrieval Hint: "detail column pinned on roster selection autoHidden center 1332 988 reveal bound to selection"

Mnemosyne (Fable 5.1, Claude Code) · session 8be1b284-089e-4f28-94be-d1a45be666b8

## Timeline

- 2026-10-16T14:34:56Z @neo-fable-mnemosyne added the `bug` label
- 2026-10-16T14:34:56Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-16T14:34:57Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-16T14:34:57Z @neo-fable-mnemosyne added the `design` label
- 2026-10-16T14:36:08Z @neo-fable-mnemosyne added parent issue #505
- 2026-10-16T19:21:38Z @neo-fable-clio cross-referenced by PR #717
- 2026-10-16T22:39:05Z @neo-gemini-julia assigned to @neo-gemini-julia
### @neo-gemini-julia - 2026-10-16T22:39:07Z

### `/ticket-intake` Validation — `neomjs/neo-agent-institution#712`

- **Ticket age:** `createdAt: 2026-10-16T18:56:58Z`, `updatedAt: 2026-10-16T18:56:58Z`
- **Bot stale-band:** `pre-stale` (`stale`: false, `no auto close`: false)
- **Parent Epic Review (`#505`):** Independent review on record at [`#505#issuecomment-6076015300`](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6076015300) (`@neo-fable`).
- **Design Authority:** [`#505#issuecomment-6217579463`](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6217579463) (Cockpit design sweep receipt `q1`) and [`#505#issuecomment-6217717533`](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6217717533) (`@neo-fable-clio` design seat ruling).
- **Core-Idiom Pre-Flight (§1.9.6):** Grounded against `src/core/Base.mjs`, `src/Neo.mjs`, `src/state/Provider.mjs`, `src/selection/Model.mjs`, `src/selection/ListModel.mjs`, `src/dashboard/dock/interaction/PerspectiveSelection.mjs`, and `src/dashboard/dock/model/Operations.mjs`.
- **Prescription checked:**
  - `apps/agentos/view/fleet/roster/SelectionModel.mjs` & `apps/agentos/view/fleet/roster/Controller.mjs` — owns `Escape` (`onKeyDownEscape`) and `deselect` / `deselectAll` propagation so empty selection fires `agentSelect: {agentId: null}` to the cockpit controller.
  - `apps/agentos/util/CockpitPerspectives.mjs`, `apps/agentos/view/fleet/cockpit/Controller.mjs` & `apps/agentos/view/fleet/cockpit/Container.mjs` — owns the single dock-write site beside `applySelection` (`detail.autoHidden` follows `selectedAgent != null` unless `detail.pinned === true` in the dock document) and preserves pinned `detail` / drops unpinned `detail` across perspective restore and reload (`AC-2`).
- **Verdict:** `valid-as-written` (Positive ROI). Claiming and branching `fix/712-detail-reveal-selection` off `origin/dev`.

- 2026-10-16T23:01:03Z @neo-gemini-julia cross-referenced by PR #720
- 2026-10-16T23:06:24Z @neo-gemini-julia referenced in commit `5e44800` - "chore(cockpit): stamp visual baselines after controller fix (#712)"
- 2026-10-16T23:54:08Z @neo-gemini-julia referenced in commit `0f956ae` - "fix(cockpit): reconcile detail pin in activatePerspective and clear removed roster selection in List (#712)"
- 2026-10-17T00:14:34Z @tobiu referenced in commit `d1f6158` - "fix(cockpit): reveal agent detail on selection and hide when cleared (#712) (#720)

* fix(cockpit): reveal agent detail on selection and hide when cleared (#712)

* chore(cockpit): stamp visual baselines after controller fix (#712)

* fix(cockpit): reconcile detail pin in activatePerspective and clear removed roster selection in List (#712)"
- 2026-10-17T00:14:35Z @tobiu closed this issue
- 2026-10-23T15:13:23Z @neo-fable-clio cross-referenced by #791
- 2026-10-24T21:27:44Z @neo-opus-ada cross-referenced by PR #805

