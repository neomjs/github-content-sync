---
id: 42
title: 'Map where controllers and state providers belong: the view-layer debt matrix'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - refactoring
assignees: []
createdAt: '2026-08-28T22:05:02Z'
updatedAt: '2026-09-23T09:23:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/42'
author: neo-fable-clio
commentsCount: 1
parentIssue: 24
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
# Map where controllers and state providers belong: the view-layer debt matrix

## Context

Operator direction (2026-08-29): feature velocity created deliberate debt, and the cleanup starts with knowing where it lives — "a quick win definitely would be to create a tech debt investigation ticket (e.g. exploring areas where controllers and state providers would make sense)." This is the measurement step between #24's Law 2 (controllers own business logic; providers own cross-view state) and the per-view repair leaves — nobody should start cutting before the map exists.

## The Problem

**Re-measured 2026-09-19 at `dev@6d953f1` — the premise moved, and the urgent part narrowed:**

- The 3,640-line cockpit container is gone. The #50 rebuild split it into `cockpit/Container.mjs` **999**, `cockpit/Controller.mjs` **998**, `cockpit/LivenessController.mjs` **993**, `cockpit/VesselContainer.mjs` 599 and `cockpit/StateProvider.mjs` — the first real provider class. Imperative reference writes in the container: 24 → **1**.
- 51 view files, 6 controller files (was 5), 1 `…StateProvider` class (was 0), 5 files using `bind:` (was 4).
- **The new structural risk is the cliff:** all three cockpit files sit within 7 lines of the 1,000-line error (`buildScripts/checkAppFileSizes.mjs`). PR #173 had to find a dead import to add an 8-line delegate. The next cockpit change by anyone errors CI.
- The old debt outside the cockpit is unchanged in kind: `accounts/Panel.mjs` 699, `roster/card/Container.mjs` 683, `detail/Container.mjs` 680, `memories/Container.mjs` 589, `system/Container.mjs` 509, `instances/ManagerContainer.mjs` 498, `catchup/Container.mjs` 470.

**Seam candidates in the trio, by measured method size** (a starting point for the matrix rows, not a prescription):

| File | Cluster | Methods (lines) | Candidate home |
|---|---|---|---|
| `Controller.mjs` | per-pane read owners | `loadTasks` 53 · `loadMemories` 49 · `loadSessionMemories` 49 · `loadCatchUp` 43 · `loadWakeRoutes` 42 · `composeOperatorMessage` 41 · `loadOperatorIdentity` 39 · `loadOperatorInbox` 37 · `markCatchUp` 33 · `openCatchUpLiveSurface` 30 — about 420 lines | a controller per pane (Law 2; `tasks/Controller.mjs` is the precedent). Open question for the row: who keeps the instance-switch generation fence |
| `Container.mjs` | pane seeding and resolution | `seedPane` 74 · `resolveProviderStore` 26 · `resolvePane` 21 · `getPreservedItemIds` 18 — about 160 lines | a declarative pane registry module: item id → component config |
| `Container.mjs` + `Controller.mjs` | perspectives | `activatePerspective` 51 · `capturePerspective` 42 · `publishPerspectives` 33 · `importPerspectiveArtifact` 25 · `afterSetActivePerspective` 25 · `exportPerspectiveArtifact` 18 — about 190 lines | one perspective owner beside `util/CockpitPerspectives.mjs` |
| `LivenessController.mjs` | the roster read | `loadRoster` 104 · `mapRosterRow` 36 · `reconcileRoster` 32 — about 170 lines | a roster read owner beside `roster/Controller.mjs` |

The investigation contract below stands. What changes is the order: **the trio's rows come first**, because they block every other cockpit PR.

*As filed —* Measured at `dev` (2026-08-29):

- **Zero `…StateProvider` classes** exist in `apps/agentos`; exactly two inline `stateProvider` configs (Viewport, FleetCockpit).
- **Only 4 view files use `bind:` at all** — across 25+ view classes, cross-view data flows almost entirely through imperative pushes (`getReference(...).set(...)`-class writes: 24 sites in the cockpit container alone).
- **5 controllers total** (Viewport, tasks, roster, roster/card, cockpit) — most surfaces route intents and lifecycle through their owning container instead.
- The plumbing concentration: `cockpit/Container.mjs` 3,640 lines (post the 2026-08-28 merges), `memories/Container.mjs` 710, `accounts/Panel.mjs` 699, `detail/Container.mjs` 676, `instances/ManagerContainer.mjs` 498, `catchup/Container.mjs` 470.

The precedent (`apps/portal/view/**`, cited by #24) carries a controller per surface and providers at view roots; the app-work gate ("providers stay at view roots") states the same law the tree does not yet follow.

## The Fix (the investigation, not the repair)

One systematic pass over `apps/agentos/view/**` producing the **debt matrix** as a ticket comment, with one row per surface:

| surface | LOC | owns business logic in the view? (named methods) | cross-view state it pushes/receives imperatively (named seams) | controller recommended? | provider recommended (which state)? | priority (blast/effort) |

Method contract:
1. **Evidence per row** — every "owns business logic" claim names the methods/lines; every imperative seam names both ends (writer → reader). No vibes.
2. **Recommendation discipline** — a controller/provider is recommended only where #24 Law 2's split applies (intents/lifecycle/reads → controller; genuinely CROSS-VIEW state → provider). A leaf owning purely local display state gets an explicit "none needed" row — the investigation must be allowed to find health.
3. **Selection-state case study** — the known worst seam (selected agent as owner-held fields feeding detail/memories/mailbox through pushes) gets a full trace as the matrix's calibration example.
4. **Cut the leaves** — file one repair leaf per coherent cluster under #24 (bundled by surface, never per-method scraps), each citing its matrix rows; the #22 cockpit decomposition consumes the cockpit rows as its planning input.
5. **Do not repair anything in this ticket** — measurement only; the leaves carry the code.

## Decision Record impact

None — applies #24's operator-stated laws; produces inputs for existing tickets (#22) and new leaves.

## Acceptance Criteria

- [ ] The debt matrix covers every file in `apps/agentos/view/**` (a "none needed" verdict is a valid, explicit row).
- [ ] Every recommendation row carries named evidence (methods/seams with both ends).
- [ ] The selected-agent seam is traced end-to-end as the calibration example.
- [ ] Repair leaves are filed under #24 (bundled per surface), each citing its rows; #22 receives the cockpit rows.
- [ ] No code changes in this ticket's scope.

## Out of Scope

- The repairs themselves (the filed leaves own them).
- The mechanical topology move and naming migrations (#24's own ordered leaves).
- Brain-side architecture (Emmy/Vega's planning lane) and DockLayouts adoption (#39).

## Related

Parent: #24 (Law 2 is the yardstick) · Consumer: #22 (cockpit decomposition planning input) · Precedent: `apps/portal/view/**` · Sibling context: #39 (DockLayouts adoption, Euclid/Mnemosyne) — the investigation notes where dock-hosted surfaces change ownership, but does not block on it.

Live latest-open sweep: checked latest 20 open issues at 2026-08-29T00:15Z — no equivalent (nearest: #22 decomposition, #24 epic; neither is the measurement pass). A2A in-flight claim sweep: recent window clean, no overlapping lane-claim.

Origin Session ID: 4f07d934-f43f-4406-98a0-9562e122e471

Retrieval Hint: `query_raw_memories("controller state provider debt matrix investigation agentos view")`

Authored by Clio (Fable 5, Claude Code). Session 41859592-b7ee-4bce-bee3-f25644d9003b.


## Timeline

- 2026-08-28T22:05:04Z @neo-fable-clio added the `enhancement` label
- 2026-08-28T22:05:04Z @neo-fable-clio added the `ai` label
- 2026-08-28T22:05:05Z @neo-fable-clio added the `architecture` label
- 2026-08-28T22:05:05Z @neo-fable-clio added the `refactoring` label
- 2026-08-28T22:05:05Z @neo-fable-clio added parent issue #24
- 2026-08-29T13:23:23Z @tobiu cross-referenced by #50
- 2026-08-29T13:23:25Z @tobiu referenced in commit `1d3b244` - "refactor(agentos): home the cockpit read families on the view CONTROLLER (#50)

Cut 2 of Epic #22 — redirected mid-lane by the operator's architecture
ruling: passing 'owner' into a util file and triggering logic on it is
a functional mixin, not a responsibility move; it keeps dodging view
controllers. The named home for this logic is FleetCockpitController.

- BOTH read families move onto the controller: the #48 fenced snapshot
  template (readFencedSource + SOURCE_READS, six pane feeds) AND this
  ticket's liveness class (loadActivity / loadRoster / loadBrainHealth
  as named methods — their variance is structural, not templatable),
  plus the loss edges degradeWiredSurface / clearDegradedReason and the
  file-local helpers (boundedRead, toSafeDegradedReason).
- AgentOS.util.CockpitSourceReads is DELETED — the util shape lasted
  one review cycle and is not public history to preserve.
- The controller reaches view state through this.component; every
  collaborator call stays owner-virtual (the #49 RA-1 law). View state
  remains view-held in this cut; state.Provider migration is the #42
  follow-up.
- The view keeps thin delegates with unchanged public signatures for
  its existing callers (startLiveness, boot, vessel paths).
- Specs migrated mechanically to the new address: prototype fakes now
  drive FleetCockpitController.prototype with {component: fake} — the
  fake shapes and every assertion are untouched; the law spec drives
  controller verbs directly.

Exact numbers at head: Container.mjs 3,339 -> 2,981 (-358); Controller
350 -> 1,186; the dev-side util (+414 via #49) is gone. Epic total:
FleetCockpit 3,640 -> 2,981 across two cuts. Behavior frozen: 735/0
unit (identical pre-cut set + the law witnesses), component 3/3,
visual 6/6 untouched goldens."
- 2026-08-29T13:23:47Z @neo-fable-clio cross-referenced by PR #51
- 2026-08-29T18:58:41Z @neo-gpt cross-referenced by PR #52
- 2026-09-02T17:12:50Z @neo-fable-clio cross-referenced by PR #86
- 2026-09-19T10:32:05Z @neo-fable-clio cross-referenced by #170
- 2026-09-19T13:30:14Z @neo-gpt-emmy cross-referenced by PR #173
### @neo-fable-clio - 2026-09-23T09:23:44Z

**Seam cut record (2026-09-23, PR for #181):** the Brain-health read left `LivenessController` as `apps/agentos/util/BrainHealthRead.mjs` — the `DeploymentStateRead` shape (static `load(owner)` / `apply(owner, response)`, the controller keeps `loadBrainHealth` / `applyBrainHealth` as handles); the roster-derived consumer refresh moved into the new `util/TargetBinding.mjs` with the target-binding rule. LivenessController 993 → 960 lines. The matrix here stands: Container 999, Controller 998 remain the next cuts.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session f34cbeb6-fd44-4060-b31f-e05332e62aee

- 2026-09-23T09:23:56Z @neo-fable-clio cross-referenced by PR #184

