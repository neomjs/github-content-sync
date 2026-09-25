---
id: 217
title: 'The plane-setup card wears the engine''s default theme, not the FM tokens'
state: CLOSED
labels:
  - bug
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T19:32:13Z'
updatedAt: '2026-09-25T20:12:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/217'
author: neo-fable-clio
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
closedAt: '2026-09-25T20:12:01Z'
---
# The plane-setup card wears the engine's default theme, not the FM tokens

## Context

#212 (merged 19:21Z) ships the packaged shell's "Connect this shell to a plane" card — the first surface a new user sees on an unconfigured double-click. The operator's first double-click on the .app rebuilt from that dev (19:2xZ) showed the card in the engine's stock theme: two 48 px blue button slabs (`Not now`, `Connect`) at the engine's semibold label size, the panel-header label at the engine's size, the field label and input in Arial at the engine's sizes, and the stock clear trigger. The operator's words: "missing styles". The card's stylesheet styles only the frame (border, ground, lede, row, status line); nothing at its scope re-values the controls.

## The Problem

Every other control in the shell speaks the FM token system (#13): the top toolbar re-values the four `--button-*` scale vars (#208), the cockpit bar puts the button label in the body role (#204), the add-agent form carries the quiet/primary hierarchy, fields ride the FM font. The card's controls fall to the engine defaults because nothing at its scope re-values them — the same class of drift #199/#204/#208 removed from the bar and the band, now on the surface that decides whether a new user trusts the product at all.

## The Architectural Reality

- `resources/scss/src/apps/agentos/PlaneSetupPanel.scss` — frame only.
- `apps/agentos/view/PlaneSetupPanel.mjs` — a `Neo.container.Panel` with a header toolbar (label + `dismiss-button`), a row (TextField + `connect-button`), a status line; the buttons carry no `cls`, so no skin can address their hierarchy.
- The engine reads `--button-height`, `--button-min-width`, `--button-padding`, `--button-border-radius` at the element and `--button-text-font-size` / `-weight` at `.neo-button-text` (`button/Base.scss`); the text field reads `--textfield-input-font` (`400 13px/17px Arial` in `theme-neo-dark`) and `--textfield-label-color`.
- The idioms already in the product: `.agent-top-toolbar` for the scale (`Viewport.scss`), `.fm-cockpit-bar .neo-button-text { font: var(--fm-text-body); font-weight: inherit }` (`fleet/cockpit/Container.scss`), `.neo-button.agent-button` (quiet) and `.fm-add-submit` (the signal primary, `fleet/instances/AddAgentForm.scss`), `.fm-add-heading` (body role, 600 for the strong).

## The Fix

1. `PlaneSetupPanel.scss`, scoped to `.agent-plane-setup`: re-value the four `--button-*` vars (32 / 32 / `0 var(--fm-space-3)` / 6); `.neo-button-text { font: var(--fm-text-body); font-weight: inherit }`; the header label in the body role at 600; `--textfield-input-font: var(--fm-text-body)` and the field label in the body role in `--fm-ink-dim`; `.neo-textfield { margin: 0 }`; the status line in `--fm-text-detail`.
2. The view: `cls: ['agent-plane-setup-dismiss']` on `Not now` (the quiet idiom) and `cls: ['agent-plane-setup-connect']` on `Connect` (the signal idiom, weight 600) — a class per role, so the hierarchy is declared where the button is.
3. A visual golden of the card in both skins: the card created into the viewport above the shell through the App worker, the way `ViewportController#mountPlaneSetup` inserts it — the harness never boots packaged, so this golden is the card's only render witness.

## Acceptance Criteria

- [ ] AC-1 The card's two buttons read the FM tokens: 32 px tall, radius 6, label in the body role; `Not now` quiet (panel lift + ink), `Connect` in the signal — computed styles asserted in the visual arm.
- [ ] AC-2 The field's input and label render in the FM font and the body role, the label in `--fm-ink-dim`; the header label in the body role at 600.
- [ ] AC-3 Goldens `plane-setup-card.png` and `plane-setup-card-light.png` capture the card as inserted above the shell.
- [ ] AC-4 No other surface changes: the stylesheet stays scoped to `.agent-plane-setup`; the rest of the visual suite passes unchanged.

## Out of Scope

- The way back from a configured-but-failing record (Forget / re-offer): a #12 leaf, per the #212 review.
- The engine theme's `Arial` text-field default and its stock clear trigger.

## Related

#212 / #211 (the card) · #13 (parent) · #12 (the shell UX specification) · #199 (the `--button-*` scale mechanism) · #204 (the label role) · #208 (the band switch)

Live latest-open sweep: the latest 20 open issues checked at 2026-09-25 19:29Z — no equivalent (#13 is the parent epic, #12 the specification). A2A claim sweep (last 60 min, all read-states): no claim on the card's styling. Memory Core rationale sweep: `query_raw_memories` on the symptom's nouns returned initialization noise only (semantic recall degraded tonight), attested. Own open assignments: #10 only.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "plane setup card FM tokens engine default theme first-run buttons"

## Timeline

- 2026-09-25T19:32:14Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T19:32:15Z @neo-fable-clio added the `bug` label
- 2026-09-25T19:32:15Z @neo-fable-clio added the `ai` label
- 2026-09-25T19:32:15Z @neo-fable-clio added the `design` label
- 2026-09-25T19:32:25Z @neo-fable-clio added parent issue #13
- 2026-09-25T19:37:16Z @neo-fable-clio cross-referenced by PR #218
- 2026-09-25T19:50:05Z @neo-fable-clio referenced in commit `2c4dc82` - "fix(shell): the plane-setup card's row sizes to its content (#217)

The row took the panel body's leftover height and its hidden overflow clipped the 32 px
controls at the top; it now sizes to its content and centers them. The visual arm asserts
that both controls lie inside the row and share a top, so a squeezed row cannot pass on
element heights again."
- 2026-09-25T20:12:01Z @tobiu referenced in commit `82d7c1c` - "Merge pull request #218 from neomjs/fix/217-plane-card-tokens

fix(shell): the plane-setup card wears the FM tokens (#217)"
- 2026-09-25T20:12:01Z @tobiu closed this issue

