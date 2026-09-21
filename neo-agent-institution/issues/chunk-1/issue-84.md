---
id: 84
title: 'The Perspectives rail pane opens on a developer placeholder: give it a product empty state that names the built-in perspectives, or keep it off the rail until its view lands'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-02T15:47:40Z'
updatedAt: '2026-09-02T18:14:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/84'
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
closedAt: '2026-09-02T18:14:36Z'
---
# The Perspectives rail pane opens on a developer placeholder: give it a product empty state that names the built-in perspectives, or keep it off the rail until its view lands

Sub-issue of #10 (the cockpit UI/UX epic). Seen on 2026-09-02 during the pre-rehearsal design pass of the cockpit on engine pin `4e0e9dd8c3` (PR #82 + PR #83 merged), both themes.

## Context

The cockpit's right rail carries four auto-hidden panes: Agent detail, Perspectives, Add agent, Wake routes. Clicking **Perspectives** reveals a pane whose entire content is the line *"Perspectives — this pane's view lands with its own leaf"* (`apps/agentos/view/fleet/cockpit/Container.mjs`, the zone resolver's `default:` branch — `cls: ['fm-pane-placeholder']`, `html: \`${item?.title ?? componentRef} — this pane's view lands with its own leaf\``). The three sibling reveals open on product empty states (*"Select an agent to inspect"*; *"Can they be woken? … Read routes"*; the S5 form).

The placeholder was deliberate ("an honest labelled placeholder, never a blank pane masquerading as a finished surface") and it did its job while the pane had no story. It has one now: the cockpit already ships three perspectives — the Overview / Focus / Review presets in the top bar — and the Neural Link exposes `list_perspectives` / `capture_perspective` / `restore_perspective`.

## The Problem

A rail tab that opens on developer copy is the one surface of the cockpit that reads as unfinished, one click away from the roster, in a rehearsal that shows the rails. "Lands with its own leaf" is our vocabulary, not the operator's.

## The Architectural Reality

- `cockpit/Container.mjs` — the zone resolver: every named zone returns its module; `default:` returns the placeholder component. `perspectives` is in the dock document (auto-hidden, right rail) but has no `case`.
- The presets: `Overview` / `Focus` / `Review` are dock perspectives the cockpit's controller applies (the top-bar segmented control); the Neural Link's perspective tools capture / list / restore the same shape.
- The other three rail panes show the empty-state grammar this pane should speak: a headline, one line of explanation, and the explicit first act as a button (Wake routes: *"Read routes"*).

## The Fix (the smallest cut first)

1. **A product empty state** for the `perspectives` zone: headline *"Saved layouts"*, the three built-in perspectives listed by name with a one-line description each and an *Apply* affordance that calls the same preset path the top bar uses, and a *Capture current layout* button as the explicit first act (disabled with a reason until capture is wired, or wired through the existing perspective service if that is a one-liner). No developer copy anywhere in the pane.
2. Alternatively — the operator's call for the rehearsal — keep the zone out of the rail until the view lands (the dock document's `autoHidden` item list), so the rail shows three tabs that all open on something.

The placeholder branch stays for genuinely unwired zones, but no zone that is IN the shipped dock document may resolve to it.

## Acceptance Criteria

- [ ] AC-1 Clicking the Perspectives rail tab reveals a pane with no developer copy: a headline, the three built-in perspectives by name, and an explicit affordance — read by eye in both themes.
- [ ] AC-2 Applying a perspective from the pane produces the same dock document as the top-bar preset of the same name (unit arm on the controller path, or the pane simply delegates to it).
- [ ] AC-3 No zone id present in the shipped dock document resolves to the `fm-pane-placeholder` branch (unit arm over the cockpit's zone list).
- [ ] AC-4 `check-visual-baselines` stamped after staging; unit + NL battery green with only the standing reds.

## Out of Scope

- Naming, renaming or deleting captured perspectives (a later leaf once capture is wired for real).
- The top-bar presets themselves.

## Related

Parent: #10. Sibling rail panes: #74 / PR #77 (Add agent), the Wake routes pane. The dock-layout perspective tools: `neomjs/neo` `src/dashboard/dock/persistence`.

Live latest-open sweep: checked the latest 20 open institution issues at 2026-09-02T15:52Z (#78 … #9); a `perspectives` search returns only the epics (#8, #9, #10, #19); no ticket for the pane's view. A2A: no claim on the cockpit's perspectives zone in the window. Structure-map gate: N/A. Structural pre-flight: a new view class is likely (`fleet/perspectives/...`) — named at PR time.

Origin Session ID: 91f83b9c-df95-4f72-a68f-d33f470792ac

Retrieval Hint: "cockpit perspectives rail pane placeholder lands with its own leaf empty state presets"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 91f83b9c-df95-4f72-a68f-d33f470792ac

## Timeline

- 2026-09-02T15:47:40Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-02T15:47:43Z @neo-fable-clio added the `bug` label
- 2026-09-02T15:47:43Z @neo-fable-clio added the `agent-os` label
- 2026-09-02T15:47:43Z @neo-fable-clio added the `ai` label
- 2026-09-02T17:12:50Z @neo-fable-clio cross-referenced by PR #86
- 2026-09-02T17:47:59Z @neo-fable-clio referenced in commit `c1fa672` - "fix(fleet): the Perspectives drawer reconciles its cards in place (#84)

The first cut rebuilt every card on each projection change (removeAll(true) + add) — an object-permanence anti-pattern the operator's review caught, and the actual cause of the drawer dropping out of the DOM right after a capture, which I had bisected as an engine race. Cards are now keyed by layoutId: existing instances get their active marker and apply verb moved in place, a new perspective inserts its card at its list position, a departed one removes its card, a moved one moves. With the churn gone the capture keeps the drawer open with its verdict and seats the switcher button in the same tick, so the deferred seating is withdrawn.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 91f83b9c-df95-4f72-a68f-d33f470792ac"
- 2026-09-02T18:14:36Z @tobiu referenced in commit `3345e5d` - "Merge pull request #86 from neomjs/agent/84-perspectives-pane

feat(fleet): the Perspectives rail pane lists, applies and captures saved layouts (#84)"
- 2026-09-02T18:14:36Z @tobiu closed this issue

