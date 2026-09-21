---
id: 50
title: 'Rebuild the cockpit core to Neo idiom: declarative Container, pattern Controller, partial StateProvider'
state: CLOSED
labels:
  - ai
  - refactoring
assignees:
  - neo-fable-clio
createdAt: '2026-08-29T12:56:09Z'
updatedAt: '2026-08-29T20:19:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/50'
author: neo-fable-clio
commentsCount: 1
parentIssue: 22
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T20:19:34Z'
---
# Rebuild the cockpit core to Neo idiom: declarative Container, pattern Controller, partial StateProvider

## Context

Second cut under Epic #22 — REDEFINED after the operator's architecture review of the first two attempts (PR #51 closed, rated 20/100; PR #49's util shape retired by the same ruling). The verdict, recorded verbatim as this ticket's acceptance frame: moving lines between files is not decomposition; the cockpit ignores the config system's power. The violated-idiom checklist (operator, 2026-08-29): anonymous imports (tab/Container ntype registration, manager/Instance outside thread entrypoints) · SNAKE_CASE consts · inline regex in methods · a massive inline state provider · zero own reactive configs against a wall of class fields · fully imperative `buildWorkspaceItems()` where Neo excels declaratively (portal shows static items configs with injection points) · 2,981 LOC against the 1k HARD limit · `additionalThemeFiles` (strong veto) · controller class not following the class-name-equals-file-name pattern · the class starting at line 741 behind module-scope function walls · `xImpl(view)` + wrapper pairs ("pointless overhead").

## Operator architecture decisions (asked + answered, 2026-08-29)

1. **State model — PARTIAL provider:** only MULTI-consumer values move onto a dedicated `cockpit/StateProvider` class (portal `ViewportStateProvider` pattern): adapter states, degraded reasons, daemon state, activity counts — the values the spine banner, grid chrome and telltale all read. Per-pane snapshots (operator inbox, memories, drill, wakeRoutes, tasks, catchUp) stay CONTROLLER state with direct pane writes; fences stay controller-internal.
2. **Container = declaration + dock loop only:** static config (declarative items incl. lazy `module: () => import(...)`, layout, controller class, stateProvider class, own reactive configs with before/afterSet hooks) plus the dock-commit loop. Vessel/tear-out and chrome-sync leave for their own classes (later cuts).
3. **Fresh build:** PR #51's branch is deleted; the rebuild is a clean PR against this checklist.

## The Fix

1. `cockpit/StateProvider.mjs` — own class, the multi-consumer data above; consumers bind.
2. `cockpit/Controller.mjs` — REWRITTEN to the Neo pattern: `class Controller`, class body starts directly after a minimal camelCase module-const header (regex consts, fixture seeds if any), ALL read/liveness/loss-edge logic as direct methods (`this.component` inline — no Impl functions, no wrapper pairs), snapshots + fences as controller fields, `setData` for provider values.
3. `cockpit/Container.mjs` — declarative rebuild: static items (lazy imports for panes), own reactive configs replacing ad-hoc field flows where the view genuinely owns state, named imports only, no `additionalThemeFiles`, no inline provider, no `buildWorkspaceItems()`-style generation. **Hard target: < 1,000 LOC.**
4. Specs follow the architecture (drive controller/provider directly); behavior parity witnessed by the full suite staying green.

## Out of Scope

Vessel/tear-out + chrome-sync extraction (next cuts, now against this foundation) · the write verbs · the file-size guard sub (armed AFTER the container is under the bar).

## Acceptance Criteria

- [ ] `Container.mjs` < 1,000 LOC, fully declarative per decision 2; zero anonymous imports; zero `additionalThemeFiles`; no inline stateProvider.
- [ ] `Controller.mjs`: class = file name, class body opens after a minimal const header, no Impl/wrapper pairs, camelCase module consts, logic in methods.
- [ ] `StateProvider.mjs`: own class carrying exactly the multi-consumer values; banner/grid/telltale consume via bindings.
- [ ] Full agentos unit tree green; component battery 3/3; visual 6/6.
- [ ] The operator's checklist above re-audited against the final diff before review request.

## Related

Parent: #22 · retired attempts: PR #49 (util, merged then superseded), PR #51 (closed) · state follow-up context: #42.

Origin Session ID: 41859592-b7ee-4bce-bee3-f25644d9003b

Authored by Clio (Fable 5, Claude Code). Session 41859592-b7ee-4bce-bee3-f25644d9003b.


## Timeline

- 2026-08-29T12:56:11Z @neo-fable-clio added the `ai` label
- 2026-08-29T12:56:11Z @neo-fable-clio added the `refactoring` label
- 2026-08-29T12:56:17Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-29T13:00:10Z @neo-fable-clio cross-referenced by PR #51
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
- 2026-08-29T16:04:48Z @neo-fable-clio changed title from **Extract the liveness read class: activity, roster, brain health** to **Rebuild the cockpit core to Neo idiom: declarative Container, pattern Controller, partial StateProvider**
- 2026-08-29T17:56:29Z @tobiu cross-referenced by PR #52
### @tobiu - 2026-08-29T17:56:52Z

## Scope update + checklist re-audit (AC 5) — PR #52

**Scope correction:** the Out-of-Scope line deferring vessel/chrome extraction was my own mis-estimate when drafting this ticket — the 1k hard limit (AC 1) is not reachable with the vessel family in place (~1.86k after every other cut). `VesselContainer` + `LivenessController` therefore land in PR #52 as decision-2's "own classes". The write verbs + file-size guard subs stay out as written.

**Per-item disposition of the verdict checklist against the final diff:**

| Item | Disposition |
|---|---|
| anonymous imports (`tab/Container`, `manager/Instance`) | named `TabContainer` import with the engine-gap note (DockLayoutAdapter emits the ntype without importing it); `manager/Instance` only in spec files, which stand in for the thread entrypoint |
| SNAKE_CASE consts | gone (`sampleActivity`, `livenessPollDefault`, `livenessReadTimeoutDefault`; regex consts camelCase) |
| inline regexes in methods | module header of `LivenessController` |
| massive inline state provider | own `StateProvider` class |
| zero own reactive configs / wall of class fields | `detailRecord_` + `presetError_` as reactive configs with afterSet hooks; vessel fields moved to `VesselContainer`, read-state fields to the controllers |
| imperative `buildWorkspaceItems()` | deleted — chrome is ONE config literal; runtime injection limited to the preset switcher reconcile + the dock projection shell. (Lazy `module: () => import()` deliberately not used: every chrome slot and resident pane projects at boot, so laziness would defer nothing.) |
| 2,981 LOC / 1k hard limit | all seven files < 1,000 (987 / 902 / 861 / 828 / 257 / 69 / 65) |
| `additionalThemeFiles` veto | gone — chrome slots are real component classes, SCSS auto-loads via the class-name convention |
| controller class-name pattern | `class Controller` in `Controller.mjs`; `LivenessController` / `VesselContainer` follow Law 1 (topology conformance suite green) |
| class starting at line 741 | `Controller` class body opens at line 28 after a bare import header |
| `xImpl(view)` + wrapper pairs | none — logic sits in controller methods with `this.component` |

Verification: unit 724/724 · component 3/3 · visual 6/6 (unchanged goldens) · docs-json regenerated. Engine findings (child-provider formulas, setData leaf-drilling, engine-config-only binds) are documented in the PR body + provider class doc.


- 2026-08-29T18:10:56Z @tobiu referenced in commit `df828fc` - "test(agentos): rebind the cross-repo contract twins to the rebuilt seats (#50)"
- 2026-08-29T18:19:41Z @tobiu referenced in commit `d719dc0` - "chore(agentos): restamp the visual-baseline input identity (#50)"
- 2026-08-29T19:11:58Z @tobiu referenced in commit `cbd7fc2` - "refactor(agentos): reconcile the review seams — parent-owned dot, first-class chip channels, static chrome (#50)"
- 2026-08-29T19:17:23Z @tobiu referenced in commit `ca18e5e` - "chore(agentos): restamp the baseline inputs for the review-reconciliation head (#50)"
- 2026-08-29T20:00:14Z @tobiu referenced in commit `59faa10` - "refactor(agentos): the derivations become real pull-based formulas; lazy wake pane; probe residue out (#50)"
- 2026-08-29T20:01:10Z @tobiu referenced in commit `2ca4dee` - "docs(agentos): the source-exact pull-based formula model in the provider JSDoc (#50)"
- 2026-08-29T20:12:48Z @tobiu referenced in commit `e6418cf` - "refactor(agentos): drop the probe-binding residue; source-exact witness doc; index-true stamp (#50)"
- 2026-08-29T20:19:34Z @tobiu referenced in commit `02896fe` - "Merge pull request #52 from neomjs/agent/50-cockpit-rebuild

refactor: rebuild the cockpit core to Neo idiom (#50)"
- 2026-08-29T20:19:34Z @tobiu closed this issue
- 2026-08-29T22:13:03Z @tobiu cross-referenced by #55
- 2026-08-29T22:16:42Z @tobiu cross-referenced by PR #56
- 2026-08-29T22:43:57Z @neo-fable-clio cross-referenced by #22
- 2026-09-01T20:57:22Z @neo-fable-clio cross-referenced by #66
- 2026-09-01T20:58:26Z @neo-fable-clio cross-referenced by PR #65
- 2026-09-01T23:24:21Z @neo-fable-clio cross-referenced by #74
- 2026-09-18T15:30:31Z @neo-fable-clio cross-referenced by #161
- 2026-09-18T15:34:00Z @neo-fable-clio cross-referenced by PR #162
- 2026-09-18T16:03:26Z @neo-fable-clio cross-referenced by PR #165
- 2026-09-19T10:32:05Z @neo-fable-clio cross-referenced by #170
- 2026-09-19T15:35:42Z @neo-fable-clio cross-referenced by #42

