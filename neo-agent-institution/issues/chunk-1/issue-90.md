---
id: 90
title: 'Engine pin bump to 35d468b51f: dock lock action, vessel lifecycle, tab paint'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - dependencies
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T09:27:30Z'
updatedAt: '2026-09-04T11:44:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/90'
author: neo-fable-clio
commentsCount: 0
parentIssue: 10
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-04T11:44:22Z'
---
# Engine pin bump to 35d468b51f: dock lock action, vessel lifecycle, tab paint

## Context

Institution `dev` (`e8abb4e`) pins `neo.mjs` at `4e0e9dd8c3` (PR #82, the fourth bump of the demo chain after #74 / PR #77 and #76 / PR #79). Engine `dev` is at `35d468b51f`, **71 commits** later: 21 `src/` files (+2126 / −444) and 33 `resources/scss` files (+521 / −307). The cockpit reads none of it until the pin moves. Grouped by what the cockpit's surfaces consume (engine ticket numbers):

- **Dock header actions** — neomjs/neo#18185 (lock joins the default engine action set), neomjs/neo#18154 (an operation declares what it can change, so a lock stops restaging the shell), neomjs/neo#18175, neomjs/neo#18239 (setting a reactive config is the change check), neomjs/neo#18241 (an action owns its own toggle mapping — `toolbar/ActionButton.mjs`, new class).
- **Tear-out / vessel lifecycle** — neomjs/neo#18153 (six PRs: the engine resolves and opens its own tear-out vessel, pop-out withheld when no vessel can be opened, a failed adoption reports and returns the item, pop-out survives a commit), neomjs/neo#18197 (a projection refuses a context that dropped the tear-out opt-ins), neomjs/neo#18169 (`resolveFreshPane` delegates), neomjs/neo#18233 (a cross-window participation answers its own seams from workspace state), neomjs/neo#18115 / neomjs/neo#18112 (the native titlebar drop: hold-to-fill, park on the move).
- **Rail reveals** — neomjs/neo#18116 / neomjs/neo#18117 (a dismissed reveal slides back under its strip, a retargeted one slides in from the edge), neomjs/neo#18134 (a dismissed reveal keeps its pane until the exit ends), neomjs/neo#18250 (the pre-projection sweep awaits the rail's in-flight reveal release).
- **Maximize / restore** — neomjs/neo#18122 (maximize fills the dock host inset by a gap, on a shadow), neomjs/neo#18164 (a pane whose home collapsed comes home to it), neomjs/neo#18177 (an edge-zone home records its own extent), neomjs/neo#18255 (a restored layout brings back the tab you were on), neomjs/neo#18142 / neomjs/neo#18143 (preview geometry, rejected-phase settle).
- **Tab paint** — neomjs/neo#18141 / neomjs/neo#18181 (every theme states the inline header's paint; the classic families give it its own ground), neomjs/neo#18171 (a pressed tab's title and glyph take the contrast colour), neomjs/neo#18145 (five PRs: the tab, grid, button and portal selectors leave the theme value files, which now declare variables only).
- **Grid** — neomjs/neo#18099 / neomjs/neo#18104 (a native drag out of a grid ends the drag-to-scroll gesture; the kinetic tail clears its state), neomjs/neo#18198 (the row and column windows carry clone descriptors, so two grids stop sharing one).
- **Theme resolution** — neomjs/neo#18105 / neomjs/neo#18106 (an item config's own theme survives creation; `getTheme` answers the closest theme through the component chain), neomjs/neo#18102 (tooltip value sheets at `:where()` weight).
- **Layout** — neomjs/neo#18201 (`layout.Card#loadModule` honours the in-flight marker it already sets).
- The remaining commits are tests, build, docs, CI and the workstation example.

## The Problem

The engine is consumed as a GitHub-SHA dependency, so `dev` advancing is invisible to the institution by design. This bump is the largest of the chain, and the first one that moves the seam the cockpit overrides: `dashboard/dock/Workspace.mjs` changed by 855 lines in the range, and the cockpit sits on it in two places — `apps/agentos/view/fleet/cockpit/Container.mjs` (`resolvePane`, `onDockZoneDocumentChange`, `enableDockTearOutLifecycle: true`) and `VesselContainer.mjs`, which extends `Neo.dashboard.dock.Workspace` and overrides `openTearOutVessel` / `closeTearOutVessel`. neomjs/neo#18153's "the engine resolves and opens its own tear-out vessel" and neomjs/neo#18197's opt-in check land exactly there, so the pin is also the first read of those overrides against the engine that now owns the default path. The same range adds a `lock` action to the default engine action set, which the cockpit's dock headers will show unless the cockpit's action set says otherwise — a product read, not a mechanical one.

## The Architectural Reality

- `package.json` → `dependencies["neo.mjs"]` = `github:neomjs/neo#<sha>`; `package-lock.json` carries the resolved commit; both are visual-baseline stamp inputs (`test/playwright/visual/__screenshots__/baseline-inputs.json`, `npm run check-visual-baselines`, the lock's `neo.mjs` entry is the stamp's `engine` axis).
- The same shape as PR #82, PR #79 and PR #77: pin, lock, stamp. Product code moves only if the live read or the battery shows a consumer-side adaptation the new engine requires; an engine defect goes to an engine ticket, not into this PR.
- Battery: `test-e2e:nl` (34 witnesses) under the Brain root, with the two stated reds (`FleetGridScaleNL`, #78; the `FleetCockpitDockNL` presets arm, the headless FLIP-settle hold from #66) — any other red is the bump's.
- The multi-window transaction design (engine D#18224, `manager.Transaction` / `TransactionGroup`) and the `Workspace.mjs` split (D#18150) are in design; nothing here anticipates them.

## The Fix

Pin `neo.mjs` at `35d468b51f` (full SHA), refresh the lock, restamp, run unit + the NL battery under the Brain root, then read the cockpit live in both themes: the dock headers' action set, maximize, a rail reveal, the tab header paint with a pressed tab, the fleet grid, one tear-out round trip, and the Perspectives pane applying a saved layout.

## Acceptance Criteria

- [ ] AC-1 `package.json` + `package-lock.json` pin `neo.mjs` at `35d468b51f` (full SHA); `check-visual-baselines` green on the pushed tree.
- [ ] AC-2 Unit suite green; `test-e2e:nl` under the Brain root reads 32/34 with only the two stated reds (#78, the #66 hold). Any other red is diagnosed in the PR: consumer-side adaptation lands here, an engine defect gets its engine ticket and a stated residual.
- [ ] AC-3 Live read on the cockpit in neo-dark and neo-light: the dock headers' action set is the one the cockpit intends (the new `lock` action either belongs on the product surface or is configured off, decided and stated), maximize fills the host inset on a shadow, a rail reveal slides in from its strip and back under it, the inline tab header paints flat with the pressed tab in contrast colour, the fleet grid renders and scrolls.
- [ ] AC-4 A tear-out round trip on the cockpit: a pane pops out into a vessel window and comes home, through the cockpit's `VesselContainer` overrides on the new engine path (neomjs/neo#18153 / neomjs/neo#18197); the pop-out action is withheld where no vessel can be opened.
- [ ] AC-5 The Perspectives rail pane (PR #86) still lists, applies and captures; a restored layout brings back the active tab (neomjs/neo#18255).

## Out of Scope

- Consumer adoption of the multi-window transaction work (D#18224) or of the `Workspace.mjs` split (D#18150) — both in design.
- neomjs/neo#18265 (a second SharedWorker window drops the replayed `registerRemote`) — an engine lane in flight; the pin carries what has landed at `35d468b51f`, and the reload-with-popup case stays a known engine defect until that lands and the next pin picks it up.
- Cockpit SCSS reconciliation beyond a defect the live read exposes (a drifted paint gets its own leaf, as #81 ruled).

## Related

#81 / PR #82 (pin 4) · #74 / PR #77 (pin 3) · #76 / PR #79 (pin 1) · #10 (parent) · #85 (open, the roster legend) · engine D#18150 · engine D#18224 · neomjs/neo#18153 · neomjs/neo#18185 · neomjs/neo#18255

Live latest-open sweep: checked the latest 20 open institution issues at 2026-09-04T09:26Z (#87 … #11) plus the full 24 at 09:22Z; no pin-bump ticket open, #81 / #74 / #76 closed. A2A sweep (last 60 min, all read-states): no claim on the institution pin — the only pin mention is Vega's role FYI naming the WebStudio consumer's pin, a different repo. Memory Core rationale sweep: `query_raw_memories("fleet manager engine pin bump neo-agent-institution …")` surfaced no prior decision on this range (miss recorded); own-assignment sweep: #10 (the parent epic) is my only open institution assignment. Structure-map gate: N/A (no `ai/` touch). Structural pre-flight: N/A (no new `.mjs`).

Origin Session ID: e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

Retrieval Hint: "institution engine pin bump 35d468b51f lock action tear-out vessel override tab paint"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

## Timeline

- 2026-09-04T09:27:30Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T09:27:32Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T09:27:33Z @neo-fable-clio added the `agent-os` label
- 2026-09-04T09:27:33Z @neo-fable-clio added the `ai` label
- 2026-09-04T09:27:33Z @neo-fable-clio added the `dependencies` label
- 2026-09-04T09:55:22Z @neo-fable-clio cross-referenced by PR #91
- 2026-09-04T09:57:11Z @neo-fable-clio cross-referenced by #92
- 2026-09-04T10:08:33Z @neo-fable-clio cross-referenced by PR #93
- 2026-09-04T10:28:27Z @neo-fable-clio cross-referenced by PR #94
- 2026-09-04T11:21:52Z @neo-fable-clio cross-referenced by PR #95
- 2026-09-04T11:23:15Z @neo-fable-clio referenced in commit `363ee32` - "docs(test): the dock-loop witness states its geometry contract — a resize keeps the splitter (#90)"
- 2026-09-04T11:44:22Z @tobiu referenced in commit `6ade255` - "Merge pull request #91 from neomjs/agent/90-engine-pin-35d468b51f

chore(agentos): engine pin to dev@35d468b51f — the cockpit reads the lock action, the vessel lifecycle and the tab paint (#90)"
- 2026-09-04T11:44:22Z @tobiu closed this issue
- 2026-09-04T13:04:46Z @neo-fable-clio cross-referenced by #98
- 2026-09-04T14:23:43Z @neo-fable-clio cross-referenced by PR #102
- 2026-09-12T10:10:47Z @neo-fable-clio cross-referenced by #120
- 2026-09-18T10:40:56Z @neo-fable-clio cross-referenced by #151
- 2026-09-18T14:17:27Z @neo-fable-clio cross-referenced by #157

