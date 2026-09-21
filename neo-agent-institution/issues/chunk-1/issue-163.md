---
id: 163
title: 'Engine pin 12 — dev@d85060795d: 10 commits, the perspective document'
state: CLOSED
labels:
  - enhancement
  - ai
  - dependencies
assignees:
  - neo-fable-clio
createdAt: '2026-09-18T15:46:36Z'
updatedAt: '2026-09-18T16:19:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/163'
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
closedAt: '2026-09-18T16:19:35Z'
---
# Engine pin 12 — dev@d85060795d: 10 commits, the perspective document

## Context

The Institution pins the engine at `github:neomjs/neo#78c3e2f916` (pin 11, 2026-09-18). Engine `dev` is at `d85060795d` — 10 commits later (`git log --oneline 78c3e2f916..d85060795d | wc -l`). #127 is blocked on one of them: neomjs/neo#18904 (`dock.Workspace#getPerspectiveDocument()` — a window-scope perspective records a pane that is away in its vessel in its home instead of as closed).

## The Problem

Without the pin the cockpit's capture verb has nothing to read, and the Neural Link capture keeps recording the hole #127 describes.

## The Architectural Reality

Drift check — the range's `src` files against what the Fleet Manager consumes (`git diff --name-only 78c3e2f916 d85060795d -- src`, 12 files):

| Engine change | FM consumer | Verdict |
|---|---|---|
| `dashboard/dock/Workspace.mjs`, `dashboard/dock/model/Operations.mjs`, `ai/client/DockService.mjs` (neomjs/neo#18904) | the cockpit is a `dock.Workspace`; `capture_perspective` reaches it through `DockService` | additive read; the Neural Link capture changes behaviour for a vesseled pane — that IS #127's fix |
| `button/Base.mjs` (neomjs/neo#18894: `toggleMenu()` awaits `menuListReady`) | `fleet/instances/SwitcherButton.mjs#toggleMenu` overrides it and calls `super` | composes: the override builds its menu through the `menu` config, so `menuListReady` exists; its own `!menuList` early return stays valid |
| `vdom/Helper.mjs` (neomjs/neo#18902), `manager/VDomUpdate.mjs` (neomjs/neo#18901) | every view | core ordering fixes — the tiers are the check |
| `main/DomAccess.mjs`, `component/Base.mjs` (neomjs/neo#18891: `measure`) | none (`git grep "\.measure("` over `apps`: no hit) | no-op |
| `grid/Container.mjs`, `grid/Row.mjs`, `grid/plugin/CellEditing.mjs`, `main/DomEvents.mjs` (neomjs/neo#18892, neomjs/neo#18897) | the FM's grids do not edit cells (no `CellEditing` in `apps`) | no-op; `registerPreventDefaultKeys` is additive |

## The Fix

`package.json` + lock to `github:neomjs/neo#d85060795d…`, a clean install (`rm -rf node_modules/neo.mjs && npm install`), verified by a code marker (`foldPlacements` in `node_modules/neo.mjs/src/dashboard/dock/model/Operations.mjs`), then the tiers: unit (isolated + Brain-contract mode), components, visual alone, isolated e2e, the Neural Link battery, `build-all`. The visual stamp is re-written only if its inputs moved.

## Acceptance Criteria

- [ ] The installed engine carries the marker; manifest and lock name the same SHA.
- [ ] Unit (both modes), components, visual (alone), isolated e2e and `build-all` are green; goldens unchanged, or every re-rendered golden read by eye.
- [ ] The Neural Link battery shows no red that is not already recorded on this host at pin 11 (control run named in the PR).

## Out of Scope

#127's consumer change (its own PR on this pin). Simplifying `SwitcherButton#toggleMenu` now that the engine awaits the lazy menu.

## Related

#127 (blocked on this); #157 (pin 11).

Live latest-open sweep: all 19 open issues of neomjs/neo-agent-institution at 2026-09-18T15:46Z — none equivalent; one open PR (#162, mine, another surface). A2A in-flight claim sweep: no pin claim in the last hour. Memory Core rationale sweep: the pin recipe and its traps are recorded from pins 7–11 (stale `node_modules` after `npm install`, stamp after `git add`, visual suite alone); nothing argues against pinning per consumer need. Own-assignment sweep: #127 (the consumer), #160/#161 (other surfaces).

Origin Session ID: 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59
Retrieval Hint: "engine pin 12 perspective document drift check SwitcherButton toggleMenu"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59

## Timeline

- 2026-09-18T15:46:37Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T15:46:39Z @neo-fable-clio added the `enhancement` label
- 2026-09-18T15:46:39Z @neo-fable-clio added the `ai` label
- 2026-09-18T15:46:40Z @neo-fable-clio added the `dependencies` label
- 2026-09-18T15:52:45Z @neo-fable-clio cross-referenced by PR #164
- 2026-09-18T15:57:43Z @neo-fable-clio referenced in commit `f5f3604` - "chore(deps): engine pin 12 — dev@d85060795d, the perspective document (#163)"
- 2026-09-18T16:19:36Z @tobiu referenced in commit `d3f17ec` - "Merge pull request #164 from neomjs/agent/163-engine-pin-12

chore(deps): engine pin 12 — dev@d85060795d, the perspective document (#163)"
- 2026-09-18T16:19:36Z @tobiu closed this issue

