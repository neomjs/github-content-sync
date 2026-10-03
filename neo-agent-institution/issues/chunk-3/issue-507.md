---
id: 507
title: The default perspective gives each important view a good home
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-10-03T12:04:32Z'
updatedAt: '2026-10-03T12:04:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/507'
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
---
# The default perspective gives each important view a good home

## Context

Operator, 2026-10-03, refining epic #505's "room" question: *"dock layouts, splitters, tear-out and popups => views are adjustable. however, initial views should still be on a decent quality level. mental model: it is super nice that we can customize views structure and placements, but this is not an excuse to make the default views structure not good."* This leaf owns the default.

## The Problem

The cockpit's default perspective (*Overview*, `apps/agentos/util/CockpitPerspectives.mjs`) was composed when the lower dock held one or two streams. It now parks six views — Activity, Tasks, Memories, Mailbox, Catch up, Golden Path — in a single tab strip under the roster at the default split, and keeps Agent Detail, Perspectives, Add agent and Wake routes as auto-hidden rail members on the right (extent 0.25). The result on the installed app at the operator's window size: the roster has room; every other important view is either one tab among six in a strip too short to read in, or a reveal overlay that disappears on the next click. The operator can fix any of it by hand — and must, every time, which is the defect: a default is the product's opinion about what matters, and the current one has none.

## The Architectural Reality

- `CockpitPerspectives.mjs` declares the catalog: `fleet-tabs` (roster), `stream-tabs` (the six), the right edge zone, and three named perspectives (Overview `zones: {}`, Focus `sizes: [0.85, 0.15]`, Review `sizes: [0.45, 0.55]` with a detail column). The engine's dock model (zones, tabs nodes, splits, edge zones, `autoHidden`, `extent`) is the vocabulary; a new default is a catalog change plus the default-split numbers, not a new mechanism.
- Perspectives are saved and restored per operator (`capture_perspective` / `restore_perspective` exist on the Neural Link); a changed default must not overwrite a saved custom perspective — it applies to first runs and to the explicit *Reset to default* only.
- The visual suite carries goldens of the default layout (`test/playwright/visual`); the default-perspective change re-captures them from a full run, and the dock e2e arms that assert the catalog (`FleetCockpitDrillNL`, bar composition) follow.

## The Fix

1. **A design page first** — `apps/agentos/design/default-perspective.html`: the operator's window size as the frame, the inventory of #505 placed by daily weight, each view's default home and size, the four questions answered per view, with captures of the current default beside the proposal. The operator approves the page (the default is his product opinion; Tier 4 — aesthetics are human-owned); the design seat authors it.
2. **The perspective change** — `CockpitPerspectives.mjs` *Overview* implements the approved page: which views are open at first run, in which zones, at which split; which stay rail members. Direction the page starts from, to be confirmed by the read, not assumed: the roster keeps the top; the lower zone splits into two side-by-side homes so a reading view (Memories or Mailbox) and a stream (Activity) are both visible without a tab change; Tasks, Catch up and Golden Path stay tabs beside them; Agent Detail opens as a column on selection (the Review perspective's `detailColumn`) instead of a reveal overlay.
3. **Reset and survival** — a saved custom perspective survives the new default; *Reset to default* restores the approved layout; first run shows it.
4. Goldens re-captured from a full visual run; the dock e2e arms updated to the new catalog.

## Acceptance Criteria

- [ ] AC-1 The design page exists on dev with captures of the current default and the proposal at the operator's window size, the inventory placed, and the four questions answered per view; the operator's approval is recorded on this ticket before AC-2 starts.
- [ ] AC-2 First run (no saved perspective) renders the approved default: unit arm on the catalog + NL/e2e arm on the rendered dock topology (`get_dock_topology`).
- [ ] AC-3 A saved custom perspective is untouched by the new default; *Reset to default* restores the approved layout (NL/e2e arm).
- [ ] AC-4 Goldens re-captured from a full visual run; `baseline-inputs.txt` re-stamped.
- [ ] AC-5 (post-merge, installed) On the next #12 cut the operator launches the app fresh and reads one memory and one mail without touching a splitter; one screenshot receipt on this ticket.

## Out of Scope

- New views or new pane content (each view's own leaf under #505).
- The dock engine (zones, splitters, tear-out) — it is the vocabulary, not the subject.

## Avoided Traps

- Deciding the default from the dev server: the read happens on the installed app with the team's data at the operator's window size.
- "The operator can rearrange it": the leaf exists because that is not an answer.
- A default that forces the detail column open for everyone: the detail opens on selection; the default shows the fleet.

## Related

#505 (epic — parent), #506 (the Memories reading pane — its default home is decided here), #312 (the Observatory — its own home by its own row), the Perspectives pane (`apps/agentos/view/fleet/perspectives/Container.mjs`), #12 (the installed cut for AC-5).

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-03 11:56Z (plus #503–#506 since); no equivalent. A2A in-flight claim sweep at 12:00Z: none on the default perspective. Memory Core rationale sweep: the Focus/Review perspectives (`CockpitPerspectives.mjs`) are the prior decisions on alternative layouts; none decided the default against an inventory. Own-assignment sweep: none of my open tickets owns the default perspective. Structure map: N/A — Institution view layer; owning file `apps/agentos/util/CockpitPerspectives.mjs` + `apps/agentos/design/`.

Retrieval Hint: "default perspective Overview six tabs lower strip design page default home per view operator approval reset to default"

Origin Session ID: 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

## Timeline

- 2026-10-03T12:04:33Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-03T12:04:34Z @neo-fable-clio added the `enhancement` label
- 2026-10-03T12:04:34Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T12:04:34Z @neo-fable-clio added the `ai` label
- 2026-10-03T12:04:34Z @neo-fable-clio added the `design` label
- 2026-10-03T12:04:41Z @neo-fable-clio added parent issue #505
- 2026-10-03T12:21:39Z @neo-opus-grace cross-referenced by #505
- 2026-10-03T12:23:25Z @neo-opus-vega cross-referenced by #485
- 2026-10-03T12:32:26Z @neo-fable-clio cross-referenced by #508

