---
id: 136
title: 'Engine pin 9 — dev@337c9fd5d1: the registers select nothing through viewConfig, the field/config guard sweeps the cockpit, the tear-out strand gets its receipt'
state: CLOSED
labels:
  - agent-os
  - ai
  - dependencies
assignees:
  - neo-fable-clio
createdAt: '2026-09-13T19:10:54Z'
updatedAt: '2026-09-14T01:13:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/136'
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
closedAt: '2026-09-14T01:13:24Z'
---
# Engine pin 9 — dev@337c9fd5d1: the registers select nothing through viewConfig, the field/config guard sweeps the cockpit, the tear-out strand gets its receipt

## Context

Pin 8 (`7e8b32e421`, #133 / PR #134) → engine dev `337c9fd5d1`: 37 engine commits, 25 files under `src` (dashboard 12, selection 5, grid 4, list 1, core 1). Carried for this app: neo#18670 — the Animate list plugin measures its row height when the owner declares no itemHeight (the roster's engine half, #128); neo#18678 — a staged vessel stand-in names the item it keeps a slot for (the fix for the regression neo#18667 carried); neo#18684 — live headers prepared for whole-stack return; neo#18661 (with neo#18668) — the grid View owns the selection model and `grid.Container#viewConfig: {selectionModel: null}` is the engine-side opt-out, the named retirement trigger of #131's transparent rebinding; neo#18629 — the engine refuses a class field that shadows a config (and the reverse) at class setup; neo#18667 — the projection refuses a tab pairing it cannot trust (the mechanism neo#18621 attributed to the cockpit's tear-out strand, closed unverified against this repo's battery); neo#18644 — a projection repair joins the refresh tail and stands down behind a newer commit; neo#18660, neo#18664, neo#18635 — a reordered tab keeps naming its item, one engine routine closes a tear-out vessel, a lease for a never-bound vessel asks the host; neo#18623, neo#18627, neo#18643, neo#18636 — the WorkspaceSet publishes its membership, perspective names come from the declared list and a snapshot carries its origin, a declared perspective may be a lowered document, reserved names are refused.

## The Problem

Three consumer debts wait on this pin. (1) `resources/scss/src/apps/agentos/fleet/memories/Container.scss:171-178` neutralizes the paint of a RowModel the registers never wanted — its own paragraph names the retirement trigger, "an engine-side opt-out", which exists now. (2) The neo#18629 guard throws at class setup for a field / config name collision; the cockpit declares some 25 class fields (`roster/List.mjs:96 navigator` shadows a `list.Base` FIELD, which is allowed; the rest are unverified until the unit tier runs at the pin — a collision would stop the app from booting). (3) neo#18621's L3 receipt is this repo's 1-of-11 tear-out strand at pin 8; nobody but this repo can run it against the fix.

## The Fix

1. `package.json` `neo.mjs` → `github:neomjs/neo#337c9fd5d10a9647efe9c487f54e677f94b468cb`; `rm -rf node_modules/neo.mjs && npm install` (a github-SHA dependency stays stale under a plain install, exit 0); the lock regenerated; the installed SHA printed next to every receipt.
2. The unit tier at the pin; every neo#18629 collision it surfaces is fixed at the field (rename, or the config it should have been); every perspective-surface drift (neo#18627 / #18643 / #18623) fixed where the cockpit reads it.
3. `apps/agentos/view/fleet/memories/RowsGrid.mjs` and `apps/agentos/view/fleet/mailbox/Grid.mjs` (read-only surfaces, both): `viewConfig: {selectionModel: null}`; the two `--grid-rowmodel-selected-*` lines and the paragraph that justified them leave `memories/Container.scss`.
4. Goldens: `npm run build-themes -- -n -e dev -t all`, the visual suite run alone, a by-eye diff read of every changed golden (an engine pin re-renders them).
5. `test/playwright/e2e/agentos/FleetCockpitTearOutNL.spec.mjs` ×11 at the pin — the strand's own battery — with the count of runs carrying a "Dock projection failed" row posted on neo#18621; if a run still strands, the drift read `bar.sortZoneConfig.dockItemIds` beside `body.items.map(c => c.dockItemId)` at the vessel's death names the production writer.

## Contract Ledger

Not applicable: a consumer pin; no public surface of this repository changes shape (`viewConfig` is the engine's seam, ledgered on neo#18626).

## Acceptance Criteria

- [ ] AC-1 `package.json` pins `337c9fd5d10a9647efe9c487f54e677f94b468cb`, the lock agrees, and the installed `node_modules/neo.mjs` reports that SHA.
- [ ] AC-2 The unit tier is green at the pin; every neo#18629 collision it surfaced is fixed at the field, named in the PR.
- [ ] AC-3 The memories registers and the mailbox grid construct without a selection model (`view.selectionModel === null`, no `neo-selection-*` wrapper cls, a card click marks no row); the transparent rebinding is gone from the SCSS; the registers' visual goldens show no selection paint.
- [ ] AC-4 Visual goldens re-rendered from a full visual run; the PR names every changed golden and what changed in it.
- [ ] AC-5 The tear-out battery ×11 at the pin, the strand count posted on neo#18621 (0 = the L3 receipt for neo#18667; >0 = the drift read attached).
- [ ] AC-6 The cockpit's e2e NL specs (FleetCockpitDockNL, FleetCockpitTearOutNL, the focus invariant) green at the pin, headless, Brain runtime root set.

## Out of Scope

#128 (the roster's measured rows — its engine half rides this pin; the consumer change is its own PR next); #127; #129; the untracked Witness A recorder rig (the tracked battery is the receipt this ticket names).

## Avoided Traps

- A plain `npm install` after the SHA edit keeps the old engine and exits 0 — remove `node_modules/neo.mjs` first, print the installed SHA next to every "at the pin" result.
- Regoldening from an isolated visual run — a visual suite beside another Playwright suite starves the webfont; rebuild the themes after every SCSS touch, regolden only from full runs.
- Reading a pin's new red as a regression before a probe — arms have been green on a false premise for months; bisect the pin range first.

## Related

#131 (closed; this pin retires its rebinding) · #133 / PR #134 (pin 8) · #120 (pin 7, the precedent) · #128 · neo#18626 / PR neo#18661 · neo#18629 · neo#18621 / neo#18667 · neo#18644.

Live latest-open sweep: the latest 20 open issues of neomjs/neo-agent-institution at 2026-09-13T19:04Z — no engine-pin ticket (#127 / #128 / #129 are mine, other surfaces). A2A in-flight claim sweep: no claim on the Institution's engine pin in the last 60 minutes. Memory Core rationale sweep: my 2026-09-12 plan named this pin (neo#18644 + neo#18629 + the grid change) — no contrary decision. Own-assignment sweep: #127, #128, #129, #10.

Origin Session ID: a1d12cdc-a975-4f69-a14e-50cc28343691
Retrieval Hint: "Institution engine pin 9 viewConfig selectionModel null registers field config guard tear-out strand receipt"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session a1d12cdc-a975-4f69-a14e-50cc28343691

Revised 2026-09-14 01:05Z: the projection unit spec followed the engine's split seam — `resolveItem` yields the cockpit's raw pane contract (module, header text, reference), `prepareItem` stamps the item identity (`dockItemId` on the config, the data and the header) — the arm asserted the stamped shape off `resolveItem` (green at pin 8, red 3/3 at the new target). Revised 2026-09-14 00:50Z: the pin target moved from `123bbe14c1` to `337c9fd5d1` (9 more engine commits: neo#18670 merged at 00:45Z — the roster's engine half — with neo#18678, the stand-in fix for the regression neo#18667 carried, and neo#18684); one pin for both consumers instead of two.


## Timeline

- 2026-09-13T19:10:54Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-13T19:10:55Z @neo-fable-clio added the `agent-os` label
- 2026-09-13T19:10:55Z @neo-fable-clio added the `ai` label
- 2026-09-13T19:10:56Z @neo-fable-clio added the `dependencies` label
- 2026-09-14T00:38:49Z @neo-fable-clio cross-referenced by #18621
- 2026-09-14T00:40:14Z @neo-fable-clio cross-referenced by PR #137
- 2026-09-14T00:45:15Z @neo-fable-clio referenced in commit `94a9b74` - "docs(fleet): the register grids' docblocks drop their ticket archaeology (#136)

The source-comment archaeology gate reads a touched file whole: the two grids' docblocks carried six ticket
references from their first landing (the view-layer conformance ticket's law 0, the mailbox sketch tickets, the
engine's scroll-edge seam ticket). The prose now names the things, not the numbers; the history stays in the
commits that introduced them."
- 2026-09-14T00:58:27Z @neo-fable-clio changed title from **Engine pin 9 — dev@123bbe14c1: the registers select nothing through viewConfig, the field/config guard sweeps the cockpit, the tear-out strand gets its receipt** to **Engine pin 9 — dev@337c9fd5d1: the registers select nothing through viewConfig, the field/config guard sweeps the cockpit, the tear-out strand gets its receipt**
- 2026-09-14T00:58:27Z @neo-fable-clio referenced in commit `9ad6abe` - "chore(deps): the pin moves to dev@337c9fd5d1 — the roster's engine half and the stand-in fix ride the same pin (#136)

Nine more engine commits since the first target: the Animate list plugin measures its row height when the owner
declares no itemHeight (the roster's engine half), a staged vessel stand-in names the item it keeps a slot for (the
fix for the regression the pairing fix carried), live headers prepared for whole-stack return. One pin for both
consumers instead of two. The projection spec's absent-pane arm followed the engine's split seam: `resolveItem`
yields the cockpit's raw pane contract (module, header text, reference) and `prepareItem` stamps the item identity
(`dockItemId` on the config, the data and the header) — the arm had asserted the stamped shape off `resolveItem`.
The visual-baseline stamp is re-taken here, after the docblock rewrites moved the app inputs past the first one.
Unit 811/811, visual 14/14, FleetMemoriesNL + FleetCockpitDockNL 4/4, FleetCockpitTearOutNL ×11 = 22/22 at the pin."
- 2026-09-14T01:11:30Z @neo-gpt cross-referenced by #10
- 2026-09-14T01:13:24Z @tobiu referenced in commit `0d5eed2` - "Merge pull request #137 from neomjs/agent/136-engine-pin-9

chore(deps): engine pin 9 — dev@337c9fd5d1; the registers select nothing through viewConfig, the field/config guard found nothing to sweep (#136)"
- 2026-09-14T01:13:25Z @tobiu closed this issue
- 2026-09-18T10:40:56Z @neo-fable-clio cross-referenced by #151
- 2026-09-18T14:17:27Z @neo-fable-clio cross-referenced by #157

