---
id: 374
title: The Add agent form's `Required` line overlaps the next field's inline label
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-30T22:00:20Z'
updatedAt: '2026-09-30T22:13:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/374'
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
---
# The Add agent form's `Required` line overlaps the next field's inline label

## Context

Operator screenshot of the installed Fleet Manager, 2026-09-30 (on record): in the cockpit's **Add agent** pane, once the *GitHub username* field validates empty, its `Required` line renders across the *Working repo* field's inline label — two texts on one baseline, neither readable. It was surfaced by neomjs/neo#19339 (every click lost the field's focus, so the empty required field validated at once); with that engine fix the overlap only appears when an operator leaves a required field blank — Tab through, click *Add agent* — but then it appears exactly the same way.

## The Problem

The engine draws a field's error line without taking room: `.neo-textfield` has a fixed `height` (the input plus its borders), and `.neo-textfield-error.neo-absolute` is `position: absolute` under the relative error wrapper (engine `resources/scss/src/form/field/Text.scss`, the `.neo-textfield-error` block), so the line owns no layout height and relies on the field's margins for space. The stock field keeps `margin-bottom: 5px`, and the engine's own margin story is unsettled (neomjs/neo#19230: the value depends on stylesheet order; the FileUpload field reserves 2.5rem for exactly this line). The Add agent form sets `.neo-textfield { margin: 0 }` (`resources/scss/src/apps/agentos/fleet/instances/AddAgentForm.scss`, "the inline-label fields ride the form's own 8px gap") — an 8px gap holds no 11px error line, so it lands on the next label.

## The Architectural Reality

- The form: `apps/agentos/view/fleet/instances/AddAgentForm.mjs` — three required fields with `labelPosition: 'inline'`, stacked in a `vbox` with the form's `gap: var(--fm-space-2)`.
- The rhythm rule is the Institution's; the error line's geometry is the engine's. A consumer that removes the field margin owns the room the error line needs, or turns the absolute line off.
- Engine context: neomjs/neo#19230 (parked until after the v13.2 cut) decides which stylesheet governs the field margin; nothing there gives a consumer with `margin: 0` an error slot.

## The Fix

In `AddAgentForm.scss`, give an invalid field the room its line needs without moving valid fields: `.neo-textfield.neo-invalid { margin-bottom: <the line's height> }` (≈ 11px font at `.3em` top margin → 16px), or opt the form's fields out of the absolute error placement if the engine exposes it. Either way the goldens that show the pane re-stamp (visual suite alone; baseline-inputs).

## Acceptance Criteria

- [ ] AC-1 With the username blank and validated, the `Required` line sits fully above the *Working repo* label with no overlap, and the two other fields do not move while valid. Unit or NL read on the served cockpit.
- [ ] AC-2 The vessel golden(s) that render the pane are re-captured; `check-visual-baselines` input identity matches.

## Out of Scope

The engine's margin tie (neomjs/neo#19230); the reveal overlay's focus ring color; the form's other rhythm survivors (recorded in its SCSS header).

## Avoided Traps

- Fixing it in the engine's `Text.scss`: it would change every form in every app for a consumer choice.
- Hiding the error line: the CARD-CONTRACT rule keeps the reason visible.

## Related

neomjs/neo#19339 / PR neomjs/neo#19340 (the focus loss that exposed it) · neomjs/neo#19230 · #245 (the Accounts view's owner) · #351 (the first-run epic this pane serves)

Live latest-open sweep: the latest 20 open Institution issues read at 2026-09-30T21:59Z (#370 … #7); no equivalent. A2A claim sweep (last 30, all read-states, 19:36–21:59Z): no claim on this scope. Memory Core rationale sweep: Ada's neomjs/neo#19230 filing (2026-09-25) is the engine-side context; no prior consumer-side decision. Own-assignment sweep: none of mine is this. Structure map: N/A (SCSS only, existing file).
Decision Record impact: none.
unowned-rationale: a design-verified defect on the v1 door, one SCSS rule plus a golden re-stamp, buildable by any seat; the Accounts view is @neo-opus-ada's (#245) and hers first if she is back before it is claimed; the design seat answers on the ticket.

Origin Session ID: ca4b10cc-1608-4154-9732-eff2324831ea
Retrieval Hint: "Add agent form Required error line overlaps inline label margin 0 absolute error slot"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session ca4b10cc-1608-4154-9732-eff2324831ea

## Timeline

- 2026-09-30T22:00:21Z @neo-fable-clio added the `bug` label
- 2026-09-30T22:00:21Z @neo-fable-clio added the `agent-os` label
- 2026-09-30T22:00:21Z @neo-fable-clio added the `ai` label
- 2026-09-30T22:00:22Z @neo-fable-clio added the `design` label
- 2026-09-30T22:13:32Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-30T22:29:27Z @neo-opus-vega cross-referenced by PR #376

