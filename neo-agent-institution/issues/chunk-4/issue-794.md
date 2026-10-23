---
id: 794
title: The cockpit keys its vessel bookkeeping by item id; the engine now keys by the minted Group key — every PR red
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - architecture
assignees:
  - neo-fable-clio
createdAt: '2026-10-23T17:24:19Z'
updatedAt: '2026-10-23T21:00:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/794'
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
closedAt: '2026-10-23T21:00:31Z'
---
# The cockpit keys its vessel bookkeeping by item id; the engine now keys by the minted Group key — every PR red

## Context

Since 2026-10-23 13:34Z every pull request's CI is red on the same two jobs, `Isolated Institution` and `Explicit Brain contract`: dependabot #786 and #787 (13:34Z), then #792 and #793 (17:0xZ). Five unit arms fail — `test/playwright/unit/apps/agentos/view/fleet/cockpit/projection.spec.mjs:305`, `:361`, `:382`, `:494` (dock projection wiring) and `cockpit/vessel.spec.mjs:397` (the REAL Group lifecycle arm). The institution's `dev` was last green on 2026-10-22 13:51Z. Both jobs run `npm run resolve-org-dev` after `npm ci`, installing the newest `dev` of the Brain and the Engine (`.github/workflows/ci.yml:38-45`, `:100-111`), so the break came from upstream, not from any of the four diffs.

**Two upstream causes** (the build read corrected the first filing, which named one):

1. `vessel.spec.mjs:397` — `expect(group.getAdmission(cockpit.id, 'stream')).toBeTruthy()` receives `null` after `group.onBind(...)`: neomjs/neo#20006 (PR #20019, 2026-10-23 01:38Z, "a vessel is identified by its minted Group key, not its item") re-keyed the native-lifecycle Group's `getConnection`, `getOwner` and `getAdmission(sourceId, key)` by the key minted at admission, resolved with `nativeWindows.keyOf(sourceId, itemId)`; `windowNameFor` became `(itemId, workspaceKey) => String`; `closeVessel` takes `workspaceKey`. The cockpit's `apps/agentos/view/fleet/cockpit/VesselContainer.mjs` still passed the item id (`getOwner(this.id, itemId)` at `:121`, `:159`, `:174`, `:440`, `:658`; `getAdmission`/`getConnection(this.id, itemId)` at `:457-458`; `retire(me.id, {...owner, itemId})` at `:666`; `ownerEntries`/`connectionEntries` consumers at `:103` and `:764`), and its unit fixtures bound the same way.
2. The four `projection.spec.mjs` arms — `TypeError: Cannot read private member #configs from an object whose class did not declare it` at the engine's `Workspace.mjs:2496` (`me.headerActionsShowOnHover`): neomjs/neo#19983 (PR #20011, 2026-10-23 00:53Z) added the reactive config `headerActionsShowOnHover_`; the spec's `Object.create(prototype)` spy host derives its dock configs by a name regex that did not match the new name, so the inherited getter threw. Test fixture only; the live cockpit is fine.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-23 17:3xZ; no equivalent. A2A claim sweep (last 10 broadcasts): none; the engine author is on the v13.3.1 cut and was informed. Memory Core rationale sweep: `VesselContainer` is the shared vessel owner by #591's prescription (Emmy, Grace agreed, 2026-10-07); no decision on the key change on the institution side. Own-assignment sweep: #505/#507/#788/#789 (mine) are unrelated.

## The Problem

A consumer drift: the engine's vessel bookkeeping changed its key and the engine's own consumer (`apps/workstation/view/VesselWorkspace.mjs`, ported in the same commit) followed, while the institution's consumer did not; a second engine change added a config the projection spec's spy host did not know. The `Explicit Brain contract` job exists to catch exactly this and did — on the next institution PR, which happened to be dependabot's. Until the port lands, no institution PR can go green, including the dependency bumps.

## The Architectural Reality

- The reference port is in neomjs/neo commit `41b554a798` (`apps/workstation/view/VesselWorkspace.mjs`, 102 lines): `workspaceKey` threaded through `registerVesselWorkspaceTarget({app, itemId, windowId, workspaceKey})`, `retireVesselWorkspaceTarget(workspaceKey)`, `disposeVessel: ({itemId, workspaceKey}) => …retireReturnedVessel(workspaceKey ?? nativeWindows.keyOf(id, itemId))`, `tearOutWindowName(itemId, workspaceKey)` with a key suffix, `resolveTearOutVessel(itemId, workspaceKey = nativeWindows.keyOf(this.id, itemId))`, the owner comparison against `vessel.workspaceKey`.
- `keyOf(sourceId, itemId)` (`src/manager/transaction/NativeLifecycle.mjs:129`) "resolves the key of an item's newest resource: a pending admission first, then a connection, then committed ownership" — so during the open effect it names the slot being opened, never an older vessel; the singly-owned precondition therefore reads the owner records' items.
- `FleetCockpit extends VesselContainer` (`cockpit/Container.mjs:74`); FM vessels carry the live pane via `VesselEmbodiment` with no dock document, so only the cockpit's own Participation and bookkeeping change — the engine's dock operations are untouched.
- `npm ci` installs the lock's engine (`8abdcf9`), which predates both changes; CI tracks `dev` on purpose. Reproducing locally needs `npm run resolve-org-dev` (or the engine at its dev tip in `node_modules/neo.mjs`).

## The Fix

Port the pattern into `VesselContainer.mjs`: resolve the key once per site (`vesselKey(itemId)` over `keyOf`) and pass it to `getOwner` / `getAdmission` / `getConnection` / `retire`; carry `workspaceKey` through the window-name and park/reshow/dispose seams the way `VesselWorkspace` does; read `ownerEntries` / `connectionEntries` by key and take the item from the record; the open-effect guard reads the owner records' items. Add `headerActions` to the projection spec's config regex. Update the cockpit unit fixtures and arms to mint and bind with the key the engine's own test utility uses. No engine change; no Workstation change.

Design authority: the engine's vessel contract after neomjs/neo#20006; the institution's consumer follows it (#591: the shared `VesselContainer` owns the vessel repair).

## Contract Ledger (the consumed vessel contract, PR #796 · review RA)

| Surface (cockpit) | Pinned Engine authority (395f4c83dc) | Shipped behavior | Fallback / refusal | Owning JSDoc | Evidence |
|---|---|---|---|---|---|
| `VesselContainer#vesselKey(itemId)` — the minted-key lookup; `isVesselOwned` / `isVesselPending` read `getOwner` / `getAdmission` / `getConnection(id, key)` | `NativeLifecycle#keyOf(sourceId, itemId)` (`src/manager/transaction/NativeLifecycle.mjs:129`): "the key of an item's newest resource: a pending admission first, then a connection, then committed ownership"; the Group keys every record by the key minted at admission (neo #20006) | an item's newest native record is its vessel; every key-addressed read goes through this one resolver | no record → `null`, and every read answers false (no vessel); the open-effect guard does **not** use it: `acquire` stores the new admission before the open effect runs, so `keyOf` names the slot being opened — the guard scans `ownerEntries(id)` for `owner.itemId === itemId` (the older committed owner) | `vesselKey` `@summary`; the guard's inline comment in `openTearOutVessel` | unit `vessel.spec.mjs:211` (with the plain guard the cockpit opened `fm-tearout-stream:2-…` for an owned pane), `:397` (REAL Group lifecycle: admission and connection read by key) |
| `workspaceKey` custody through park, reshow, dispose and return: `parkNativeVessel` / `reshowNativeVessel({itemId, windowName, workspaceKey = vesselKey(itemId)})`, `suspendNativeWindowDrag` → `onConversionIn({itemId, windowName, workspaceKey})`, `disposeNativeVessel` → `returnPane(vessel.itemId, vessel.workspaceKey)`, `returnPane(itemId, workspaceKey = vesselKey(itemId))` → `retire(id, {...owner, itemId, workspaceKey})` | `VesselPark`'s dispose callback `({itemId, workspaceKey})` and `NativeVesselTransaction.closeVessel({…, workspaceKey})` (neo commit 41b554a798); `NativeLifecycle.retire` keeps exact key, generation and name custody (the late-old-close / newer-resource control) | an explicit key names the exact resource end to end (a parked old key retires itself, never the item's newer vessel) | without a key the item's newest vessel answers (`vesselKey`); no owner for the key → `returnPane` answers `{returned: false, errors: […]}` and retains the retirement, the admission and the connection | the `@param {String} [vessel.workspaceKey]` lines on `parkNativeVessel`, `reshowNativeVessel`, `returnPane` | unit `vessel.spec.mjs:397` (a refused or failed platform close retains the records, gates the next admission, the retirement completes); NL `FleetCockpitTearOutNL` (stream, operator), the permanence-matrix-row-4 spec |
| Key-suffixed window naming: `tearOutWindowName(itemId, workspaceKey)` = `fm-tearout-${itemId}${tearOutKeySuffix(itemId, workspaceKey)}-${id}`; `openTearOutVessel` names the window with `topologyIdentity?.workspaceKey` | `Workspace#tearOutKeySuffix(itemId, workspaceKey)` and `windowNameFor(itemId, workspaceKey)` (neo #20006; the suffix exists from engine 395f4c83dc, the lock's pin — on 8abdcf9 it threw `tearOutKeySuffix is not a function`, the reason the lock moved) | the preferred key yields the unchanged name (`fm-tearout-stream-<id>`); a minted `:2` key yields `fm-tearout-stream:2-<id>` — two live vessels of one pane never share a name, which `Main#windowOpen` would answer by reusing the first window | no key → no suffix (the old name) | the override's `@summary` | unit `vessel.spec.mjs:239` (asserts the `:2` suffix); NL `FleetCockpitNWindowNL` (the keyed connection read through `vesselKey`) |
| Record consumers: `resolveNativeWindowDrag` reads `owner.itemId`; `pushVesselTitles` destructures `[, {itemId, windowId}]` | the Group's records each name the item they were opened for (neo commit 41b554a798); the map key is the minted key | dragging a vessel window's title bar back to main resolves its pane again; a vessel window is titled by its pane, not `popup:<item>` | an entry without `itemId` resolves nothing (no drag-back) / no title push | the methods' existing JSDoc (unchanged) | the rewritten unit fixture keys every record `popup:<item>` with `itemId`; NL `FleetCockpitTearOutNL` (the return by drag) |

Not a consumed contract, recorded for completeness: the projection spec's prototype-spy host now admits `headerActions*` configs (neo #19983's reactive `headerActionsShowOnHover_`) — a test fixture census, no cockpit behavior.

## Acceptance Criteria

- [ ] AC-1 (red-first): with the engine at its `dev` tip in `node_modules` (`npm run resolve-org-dev`), the five arms above are red on the current source and green at head; the full unit tier is green.
- [ ] AC-2: the tear-out / vessel NL e2e specs of the cockpit stay green on a source build against the same engine — no behavioural change in pop-out, return and retire (a pre-existing red elsewhere is its own leaf).
- [ ] AC-3 (post-merge): the next PR's `Isolated Institution` and `Explicit Brain contract` jobs are green (the two dependabot PRs re-run).

## Out of Scope

The engine's key design (settled in #20006); the Workstation; pinning the institution to the lock's engine in CI (the contract job's purpose is to track `dev`); the engine-side process gap (an engine PR touching `src/dashboard/dock/window` has no institution consumer check before merge — a follow-up for the engine's CI, named here, not filed); #795 (the capture-restore red, pre-existing).

## Avoided Traps

Loosening the five arms or skipping them; re-keying in the specs only (the live cockpit would still pass item ids); a second bookkeeping layer in the cockpit that maps keys back to items; guarding the open effect with `keyOf` (it names the slot being opened).

## Related

neomjs/neo#20006 / PR #20019 and neomjs/neo#19983 / PR #20011 (the engine changes) · #591 (the shared `VesselContainer` prescription) · #786, #787, #792, #793 (blocked PRs) · #795 (the pre-existing capture-restore red found on the way) · #732 (the FM update lane; the installed app is unaffected until the next candidate carries both).

handoff: Clio (self-assigned) drives it through a delegated builder; the engine author is informed.

Origin Session ID: d0801374-2558-4c56-97d0-ba4a719149a3

Handoff Retrieval Hints: `query_raw_memories("institution VesselContainer keyOf minted Group key getAdmission null every PR red #20006 headerActionsShowOnHover spy host regex")`; the CI runs 39992010121 / 39992017021.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session d0801374-2558-4c56-97d0-ba4a719149a3


## Timeline

- 2026-10-23T17:24:20Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-23T17:24:21Z @neo-fable-clio added the `bug` label
- 2026-10-23T17:24:21Z @neo-fable-clio added the `agent-os` label
- 2026-10-23T17:24:21Z @neo-fable-clio added the `ai` label
- 2026-10-23T17:24:21Z @neo-fable-clio added the `architecture` label
- 2026-10-23T17:25:00Z @neo-fable-clio cross-referenced by PR #792
- 2026-10-23T17:25:03Z @neo-fable-clio cross-referenced by PR #793
- 2026-10-23T17:56:47Z @neo-fable-clio cross-referenced by #795
- 2026-10-23T17:57:54Z @neo-fable-clio cross-referenced by PR #796
- 2026-10-23T19:14:45Z @neo-fable-clio referenced in commit `ae53218` - "build(deps): the lock's engine carries the minted Group key (#794)

The Isolated Institution job runs the unit tier on the lock's engine before
resolve-org-dev, and 8abdcf9 predates the Group's `keyOf` (neomjs/neo#20006).
The lock now resolves neo.mjs to 395f4c83dc, the engine's dev tip."
- 2026-10-23T19:14:46Z @neo-fable-clio referenced in commit `1b3b8c0` - "test(visual): the baseline stamp follows the vessel port (#794)

Stamped on a fresh `npm ci` (engine 395f4c83dc). The visual suite fails
only dev's nine stale goldens, each with dev's pixel count on the same
engine; the AgentCardSynthesisRenderNL capture passes. No golden changed."
- 2026-10-23T21:00:31Z @tobiu referenced in commit `b30a83e` - "fix(cockpit): the vessel bookkeeping keys by the minted Group key (#794) (#796)

* fix(cockpit): the vessel bookkeeping keys by the minted Group key (#794)

The engine's native-lifecycle Group keys admissions, connections and owners
by the workspace key it mints at admission (neomjs/neo#20006); each record
names the item it was opened for. VesselContainer resolves an item's vessel
through `keyOf` (`vesselKey`) for every key-addressed read and for the
retirement, reads the item from the `ownerEntries`/`connectionEntries`
records, carries the key through the native-return park/reshow/dispose
seams, and names a vessel's window with its key suffix (`tearOutWindowName`).

The open effect runs while the Group already holds the new slot as the
item's newest record, so its singly-owned precondition reads the owner
records' items: a `keyOf` read there can never see an older vessel.

The four projection arms failed on a separate upstream change: the spy
host's derived dock configs missed `headerActionsShowOnHover`
(neomjs/neo#19983), whose inherited getter throws without `#configs`.
The unit fixtures and the N-window NL probe key by the minted Group key.

* build(deps): the lock's engine carries the minted Group key (#794)

The Isolated Institution job runs the unit tier on the lock's engine before
resolve-org-dev, and 8abdcf9 predates the Group's `keyOf` (neomjs/neo#20006).
The lock now resolves neo.mjs to 395f4c83dc, the engine's dev tip.

* test(visual): the baseline stamp follows the vessel port (#794)

Stamped on a fresh `npm ci` (engine 395f4c83dc). The visual suite fails
only dev's nine stale goldens, each with dev's pixel count on the same
engine; the AgentCardSynthesisRenderNL capture passes. No golden changed."
- 2026-10-23T21:00:32Z @tobiu closed this issue

