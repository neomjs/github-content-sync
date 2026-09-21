---
id: 126
title: 'The cockpit declares its panes and zones; the hand-built document, the resolver switch and the first-shell mount go'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - architecture
  - refactoring
assignees:
  - neo-fable-clio
createdAt: '2026-09-12T11:44:35Z'
updatedAt: '2026-09-12T17:25:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/126'
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
blockedBy:
  - '[x] 120 The cockpit''s dock catalog must move from componentRef to reference before the next engine SHA bump'
blocking:
  - '[x] 127 A perspective captured while a pane is in its vessel leaves that pane homeless when restored'
closedAt: '2026-09-12T17:25:22Z'
---
# The cockpit declares its panes and zones; the hand-built document, the resolver switch and the first-shell mount go

## Context

Engine pin 7 (#120 / PR #125) carries the engine's declarative dock authoring: a Workspace declares `panes` (ordinary component configs keyed by pane id) and `zones` (the nested arrangement), and `Neo.dashboard.dock.model.Authoring.fromZones` lowers them into the `neo.dock.zone.v1` document the projection consumes (neomjs/neo#18475, #18476, #18477; the adoption guide's "Migrating a consumer that already does this by hand"). The Fleet cockpit predates that path and still writes the boot half itself: a hand-maintained document (`apps/agentos/util/CockpitDockDocument.mjs`, 86 lines: eleven item records and five node records with hand-kept ids), a `resolveDockReference` switch in `apps/agentos/view/fleet/cockpit/Container.mjs` (~200 lines mapping each `reference` to its pane config — the second registry of the same names the guide names), and the manual first-shell mount in `construct()` (`me.add(Object.assign(me.projectDockModel(), {flex: 1}))`).

## The Problem

Two registries of one vocabulary drift by construction: the document names `agent-detail`, the switch names `agent-detail`, the pane config names `reference: 'agent-detail'`, and the presets (`CockpitPresets.mjs`) name the node ids the document happens to use. Every pane addition touches four places. The engine now owns exactly this composition (the guide: "the diff is almost entirely red"), including reopening a closed declared pane (`openPane`), fresh-instance recreation, and the identity of a declared pane across tab moves and rail transitions.

## The Architectural Reality

- `Authoring.fromZones(panes, zones)` accepts explicit node `id`s (`zones.id` for the root, `{id, items, activeItemId}` for tabs, `{id, orientation, children, sizes}` for splits, edge children with `extent`/`resizable`) and keeps them (`context.explicit`) — so the cockpit's node ids (`cockpit-root`, `primary-split`, `fleet-tabs`, `stream-tabs`, `secondary-rail`) survive the lowering and the presets in `CockpitPresets.mjs` keep addressing them.
- `Authoring.toItem(key, pane)` lifts the item-record half from the pane config (`header.text` → `title`, the policy keys such as `autoHidden`); modules, bindings, listeners, `shellTools` and the FLIP marker stay runtime config outside the document (`Workspace#paneDeclarations`).
- `Workspace#resolvePane` resolves declared keys through the component manager (`Neo.get(declaration.id) || Neo.clone(declaration)`), `resolveProjectedPane` hands a returning vessel pane back first, and `getPreservedItemIds` preserves declared catalog members by default — the cockpit's `resolveDockReference` switch, its owner-state materialization for absent panes (`adapterState`, `memoriesSnapshot`, `detailRecord`, …) and its stand-in for a vesseled item are the pieces to sort: bindings and owner-held state move onto the declarations (`bind` on the pane config, provider data for the snapshots), the stand-in stays a `resolvePane` override for the one case the engine cannot know (a vesseled item re-treed by a preset).
- `dockShellIndex: 1` / `dockProjectionConfig: {flex: 1}` already tell the engine where the shell sits; the manual `add` in `construct()` predates the class mounting its own first shell.

## The Fix

1. `Container.mjs` declares `panes` — one entry per catalog key with its module, `header: {text}`, `autoHidden` where the record has it, the pane's `bind` / listeners / `shellTools` as today — and `zones` with the cockpit's ids:
   ```js
   zones: {
       id    : 'cockpit-root',
       center: {id: 'primary-split', orientation: 'vertical', sizes: [0.6078, 0.3922], children: [
           {id: 'fleet-tabs',  items: ['fleet']},
           {id: 'stream-tabs', items: ['stream', 'tasks', 'memories', 'operator', 'catchUp'], activeItemId: 'stream'}
       ]},
       right : {id: 'secondary-rail', items: ['detail', 'perspectives', 'defineAgent', 'wakeRoutes'], extent: 0.25, resizable: true}
   }
   ```
2. `CockpitDockDocument.mjs` retires; `CockpitPresets.mjs` derives its seeded layouts from the cockpit's lowered document (the ids are the same) or from `Authoring.fromZones` directly.
3. `resolveDockReference` shrinks to the two cases the engine cannot own: the stand-in for a vesseled item and the owner-state materialization the declarations cannot express — every plain case goes; the `construct()` mount goes.
4. The specs follow: `cockpitDockDocument.spec.mjs` becomes the declaration's lowering spec (the same document, asserted through `Authoring.fromZones`), `projection.spec.mjs`'s resolver arms cover the two remaining cases, `vessel.spec.mjs` is unchanged.
5. The arrangement on screen is identical afterwards — the visual goldens are the check (the guide's own falsifier).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `Container.mjs` `panes` / `zones` configs | `Neo.dashboard.dock.Workspace#panes` / `#zones` (pin 7) | the cockpit's catalog and arrangement as declarations, node ids explicit | none — a declaration error fails construction loudly (`onAfterConstructed` throws) | class JSDoc | the lowering spec + the visual goldens |
| `CockpitDockDocument.mjs` | — | retired | — | — | grep 0 |
| `CockpitPresets.mjs` seeded layouts | the lowered document | unchanged node ids, unchanged presets | — | module JSDoc | `perspectiveShare.spec.mjs`, `FleetCockpitDockNL` presets arm |
| `resolveDockReference` | `Workspace#resolvePane` override | two cases only (stand-in, owner-state materialization) | the engine's declared resolution | JSDoc | `projection.spec.mjs` |

**Decision Record impact:** aligned-with ADR 0029 (the declarative path is its consumer shape).

## Acceptance Criteria

- [ ] The cockpit declares `panes` and `zones`; `CockpitDockDocument.mjs` is gone; the lowered document equals today's document node-for-node (ids, sizes, extents, active items, auto-hidden records) — asserted through `Authoring.fromZones` against the pre-migration fixture.
- [ ] `resolveDockReference` is gone; the cockpit's `resolvePane` keeps exactly two responsibilities over the engine's declared-instance answer, both specced: the vessel stand-in (an owned or in-flight item is never stolen back by a restore) and the first-creation seeds (`seedPane`: the owner-held snapshots, the selected resident, the inspector's stores, the shell-owned window toggles, the listener scope a vesseled pane needs).
- [ ] The presets restore unchanged (the `FleetCockpitDockNL` presets arm and `perspectiveShare.spec.mjs` green).
- [ ] The visual goldens are byte-identical (the arrangement on screen is the same); the NL battery matches its pin-7 baseline.
- [ ] `construct()` seeds nothing: the first projection is the LOWERED document's, made once in `onAfterConstructed` after the engine seeded `dockModel` (the cockpit is the app's main view — its resident panes exist at boot, before the bridge answers — so the shell projects eagerly into the slot `dockShellIndex` names after the control bar; the engine's mount-time pass finds it and leaves it); the presets derive from the AUTHORED declaration (`panes` + `zones` lowered by the preset factory itself), never from the active document — the engine lets a supplied `dockModel` win over `zones` and keeps it active, so a supplied document with a pane closed still boots and a supplied resize never becomes Overview's seed (reviewer R1, 2026-09-12); the projected perspective list follows the library.
- [ ] The rail's auto-hidden panes are declared instances the engine parks, never re-creates: the inspector keeps its instance (same id) across a perspective switch and a reveal/hide cycle. Parking is scoped to the rail (`getPreservedItemIds`): the reading surfaces carry engine grids whose row pool repaints on a remount after the store emptied (defect-noted 2026-09-12), so they rematerialize from their seeds until the grid remount lands — the override names that retirement trigger. Added 2026-09-12 from the live demo: after `restore_perspective('Overview')` the `AgentOS.view.fleet.detail.Container` query returned no instance — the hand-built path releases the revealed pane on a restore and rebuilds it on the next reveal, so selection, scroll and drill state are lost at every perspective switch.

## Out of Scope

- The pane-side pop-out toggles vs the engine's header pop-out action (a UX decision, its own ticket).
- Any change to the presets' vocabulary or the perspective library.

## Avoided Traps

- **Generated node ids** — the presets address nodes by id; the declaration keeps the cockpit's ids explicitly, never lets `fromZones` mint them.
- **Porting the switch onto `resolvePane`** — the engine resolves declared keys; only what it cannot know stays.
- **A different arrangement after migrating** — the guide's rule: if it looks different, the hand-built document and the declaration disagreed; fix the declaration, never accept the drift.

## Related

#120 / PR #125 (engine pin 7 — Blocked by: this lands after it) · neomjs/neo#18476 · neomjs/neo#18477 · `learn/guides/uibuildingblocks/DockLayoutsAdoption.md` §"Migrating a consumer that already does this by hand" · #10.

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-09-12T11:43Z; no equivalent found. A2A in-flight claim sweep: no `[lane-claim]`/`[lane-intent]` on the cockpit's declarative migration in the last 60 minutes (neomjs/neo#18530 is the engine's own pane-ownership preview). Memory Core rationale sweep: the 2026-08-22 dock architecture audit names the FleetCockpit as a hand-rolled host and leaves its migration to the FM lane; no decision against.

Origin Session ID: fcdd7d71-e7bd-46e8-bd01-5d46ea620205

Retrieval Hint: "cockpit declares panes zones fromZones explicit node ids presets resolveDockReference two cases"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session fcdd7d71-e7bd-46e8-bd01-5d46ea620205


## Timeline

- 2026-09-12T11:44:35Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-12T11:44:36Z @neo-fable-clio added the `enhancement` label
- 2026-09-12T11:44:36Z @neo-fable-clio added the `agent-os` label
- 2026-09-12T11:44:37Z @neo-fable-clio added the `ai` label
- 2026-09-12T11:44:37Z @neo-fable-clio added the `architecture` label
- 2026-09-12T11:44:37Z @neo-fable-clio added the `refactoring` label
- 2026-09-12T12:19:31Z @neo-gpt-emmy cross-referenced by PR #125
- 2026-09-12T13:17:21Z @neo-fable-clio cross-referenced by #127
- 2026-09-12T14:57:25Z @neo-fable-clio referenced in commit `bcc6ae0` - "test(fleet): the dock witness describes current behavior without provenance pointers (#126)

The archaeology gate reads a touched file whole: two ticket refs on the geometry fast path go (resizeSplit still declares the geometry change class at pin 7, the sentence stays), and the headless FLIP-settle hold ledger goes with its premise — pin 7 has no visibility-staged tab chrome in the dock sources, and the step is green headless in every run at this pin."
- 2026-09-12T14:57:47Z @neo-fable-clio cross-referenced by PR #130
- 2026-09-12T16:50:42Z @neo-fable-clio referenced in commit `ec65365` - "fix(fleet): the presets derive from the authored declaration; a supplied document stays active (#126)

The first head passed the ACTIVE dockModel to the preset factory, and the engine lets a supplied document win over zones — a valid supplied document with the inspector closed crashed construction at CockpitPresets.mjs:66, and a supplied resize silently became Overview's seed (review R1). CockpitPresets.fromDeclaration(panes, zones) lowers the declaration itself, throws loudly if it does not lower, and hands the document to create; the cockpit calls it from onAfterConstructed, and the supplied document stays active, untouched. declaration.spec: the default boot (the shipped variants), a supplied document with the inspector closed (boots, stays active, Overview is the declaration, Review still opens the inspector), a supplied resize (active; Overview's seed unchanged) — the two supplied arms red on the first head."
- 2026-09-12T17:01:57Z @neo-fable-clio referenced in commit `5fab5ed` - "feat(fleet): the cockpit declares its panes and zones (#126)

`panes` (one component config per dock item: module, bindings, string handlers, header.text
as the title, autoHidden on the rail tools) and `zones` (the SSOT §01 arrangement with its
explicit node ids) replace the hand-built CockpitDockDocument, the eleven-case
resolveDockReference switch and the dockModel seed in construct(). The engine lowers the
declaration in onAfterConstructed, captures a config per pane with a runtime id, and parks a
rail pane that leaves the tree: the inspector keeps its instance across a perspective switch
and a reveal/hide cycle.

What stays the cockpit's, in resolvePane over the engine's declared-instance answer: the
vessel stand-in (an owned or in-flight item is never stolen back by a restore) and the
first-creation seeds (seedPane: the owner-held snapshots, the selected resident, the
inspector's stores, the shell-owned window toggles, the listener scope a vesseled pane
needs); resolveFreshPane seeds the recreate path the same way. The first projection is the
LOWERED document's, made once after the engine seeded it (dockShellIndex 1 after the control
bar, flex from dockProjectionConfig); the presets and the projected perspective list derive
from that same document. Parking is scoped to the rail (getPreservedItemIds): the reading
surfaces carry engine grids whose row pool repaints on a remount after the store emptied, so
they rematerialize from their seeds until the grid remount lands. The define-agent pane
answers to the record's own name; the add-agent-form alias is gone.

Specs: declaration.spec (the lowering equals the shipped document node-for-node; the parked
inspector survives Review -> Overview -> reveal as the same instance); shippedDockDocument
carries the shipped document as the spec fixture plus declaredPanes() for spy hosts; the
projection, vessel, intent and owner-seam specs resolve through the declaration; the dock
witness asserts the parked inspector, the tear-out witness waits for committed ownership
before killing the vessel."
- 2026-09-12T17:01:57Z @neo-fable-clio referenced in commit `dd2207d` - "test(fleet): the dock witness describes current behavior without provenance pointers (#126)

The archaeology gate reads a touched file whole: two ticket refs on the geometry fast path go (resizeSplit still declares the geometry change class at pin 7, the sentence stays), and the headless FLIP-settle hold ledger goes with its premise — pin 7 has no visibility-staged tab chrome in the dock sources, and the step is green headless in every run at this pin."
- 2026-09-12T17:01:57Z @neo-fable-clio referenced in commit `f75a197` - "fix(fleet): the presets derive from the authored declaration; a supplied document stays active (#126)

The first head passed the ACTIVE dockModel to the preset factory, and the engine lets a supplied document win over zones — a valid supplied document with the inspector closed crashed construction at CockpitPresets.mjs:66, and a supplied resize silently became Overview's seed (review R1). CockpitPresets.fromDeclaration(panes, zones) lowers the declaration itself, throws loudly if it does not lower, and hands the document to create; the cockpit calls it from onAfterConstructed, and the supplied document stays active, untouched. declaration.spec: the default boot (the shipped variants), a supplied document with the inspector closed (boots, stays active, Overview is the declaration, Review still opens the inspector), a supplied resize (active; Overview's seed unchanged) — the two supplied arms red on the first head."
- 2026-09-12T17:25:22Z @tobiu referenced in commit `fc7206f` - "Merge pull request #130 from neomjs/agent/126-declared-panes-zones

feat(fleet): the cockpit declares its panes and zones (#126)"
- 2026-09-12T17:25:22Z @tobiu closed this issue
- 2026-09-12T21:01:34Z @neo-fable-clio cross-referenced by #133

