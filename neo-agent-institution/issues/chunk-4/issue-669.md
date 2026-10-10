---
id: 669
title: 'VesselContainer imports the dock factory neo #19564 removed'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - regression
assignees:
  - neo-opus-ada
createdAt: '2026-10-10T19:50:55Z'
updatedAt: '2026-10-10T20:27:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/669'
author: neo-opus-ada
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
blocking:
  - '[x] 602 Loading older Mailbox rows resets the reading position'
closedAt: '2026-10-10T20:27:45Z'
---
# VesselContainer imports the dock factory neo #19564 removed

## Context

Institution `dev` CI has been red since its first run after neomjs/neo#19564 merged: run 38079855713 at 19:27:14Z. The 18:36Z run passed. Both jobs fail at module link:

```
SyntaxError: The requested module '../../../../../node_modules/neo.mjs/src/dashboard/dock/window/VesselEmbodiment.mjs' does not provide an export named 'createDockVesselProxyEmbodiment'
```

CI runs `npm run resolve-org-dev` after `npm ci`, so it installs neo's current `dev` whatever the lock records. Every Institution PR stays red until this consumer adopts the class.

## The Problem

neomjs/neo#19564 (resolving neomjs/neo#19540) turned the dock's tear-out factories into registered Neo classes. `createDockVesselProxyEmbodiment` became `class VesselProxyEmbodiment`, in its own file. The PR body named this consumer but put the swap off to the Institution's next lock move, because it missed that CI resolves `dev`. Ada authored #19564 and owns this fix.

## The Architectural Reality

- `apps/agentos/view/fleet/cockpit/VesselContainer.mjs` imports the factory and calls it with two seams, `resolvePane` and `resolveProxyConfig`.
- neo `src/dashboard/dock/window/VesselProxyEmbodiment.mjs` declares both seams as configs with the same names. The methods the cockpit calls (`restoreByWindow`, plus the `dragEmbodiment` hand-off) live on its prototype, and its `destroy()` releases what it owns. `VesselContainer#destroy` already calls `destroy()`.
- `FleetCockpitTearOutNL` and `FleetPermanenceMatrixRow4NL` reach `tearOutHandlers.*` through Neural Link paths. `InstanceService#callMethod` calls `scope[methodName].call(scope)`, so `this` is the `TearOut` instance and those specs need no change.
- `test/playwright/unit/apps/agentos/view/fleet/cockpit/vessel.spec.mjs` overrides `cockpit.tearOutHandlers.heldPane` and `heldPanes`. Before, the factory closures called those functions lexically, so the overrides reached only outside readers. Now `TearOut`'s own calls dispatch through `this` and see them too.

## The Fix

1. `VesselContainer.mjs`: import `VesselProxyEmbodiment` and create it with `Neo.create(VesselProxyEmbodiment, {resolvePane, resolveProxyConfig})`.
2. Run every suite Institution CI runs against neo `dev`. Adapt `vessel.spec.mjs` only if the override semantics change its outcome, and name the reason.

## Acceptance Criteria

- [ ] `git grep createDockVesselProxyEmbodiment` finds nothing in the Institution.
- [ ] Institution CI is green on the PR, with `resolve-org-dev` installing neo `dev` at or after `1c43d51e26`.
- [ ] The suites that load `VesselContainer` pass locally against neo `dev`; any test change states its reason.

## Out of Scope

- The lock move: CI installs `dev` regardless.
- The packaged FM candidate (#12): it pins Engine `5698517f`, which is unaffected.

## Avoided Traps

- **A compatibility re-export in neo:** `createDockVesselProxyEmbodiment = cfg => Neo.create(VesselProxyEmbodiment, cfg)` would turn CI green with one engine commit. It would also re-add the factory surface neomjs/neo#19540 retired, whose AC-1 is that no factory export is left. The consumer swap is one import and one call.

## Related

neomjs/neo#19540, neomjs/neo#19564 (the breaking merge; its Post-Merge Validation names this swap), #602 and #666 (PR lanes blocked by the red), #12.

Sweeps:
- Live latest-open sweep: the latest 20 open Institution issues at 19:49:17Z, plus the latest 5 at 19:50:33Z. No equivalent.
- A2A in-flight sweep, 30 messages: no competing claim. Sophie had accepted the swap at 19:47Z, and Ada took it back at 19:49Z once CI proved red already.
- Memory sweep: no prior decision.
- Own-assignment sweep: #516 and #424, both unrelated.

Origin Session ID: 8321fa60-be87-4732-a514-8bfe197fb299
Retrieval Hint: "VesselContainer createDockVesselProxyEmbodiment removed export resolve-org-dev Institution CI red 19564"

## Timeline

- 2026-10-10T19:50:55Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-10T19:50:57Z @neo-opus-ada added the `bug` label
- 2026-10-10T19:50:57Z @neo-opus-ada added the `agent-os` label
- 2026-10-10T19:50:57Z @neo-opus-ada added the `ai` label
- 2026-10-10T19:50:57Z @neo-opus-ada added the `regression` label
- 2026-10-10T19:54:30Z @neo-opus-ada cross-referenced by PR #670
- 2026-10-10T19:57:07Z @neo-opus-ada cross-referenced by PR #19564
- 2026-10-10T20:07:48Z @neo-gpt cross-referenced by PR #672
- 2026-10-10T20:11:40Z @neo-opus-ada referenced in commit `157a7ef` - "test(visual): restamp the baseline inputs for VesselContainer's class swap (#669)

VesselContainer.mjs is a style-owning input, so its swap moved the stamp. On Darwin at neo dev 9e48135295, FleetCockpitVisual passes 51/51 and AgentCardSynthesisRenderNL passes with no snapshot change; the stamp records the new blob and that engine."
- 2026-10-10T20:23:09Z @neo-gpt-sophie marked this issue as blocking #602
- 2026-10-10T20:27:45Z @tobiu referenced in commit `46ccb42` - "fix(cockpit): VesselContainer creates the engine's VesselProxyEmbodiment class (#669) (#670)

* fix(cockpit): VesselContainer creates the engine's VesselProxyEmbodiment class (#669)

neomjs/neo#19564 replaced createDockVesselProxyEmbodiment with the registered class Neo.dashboard.dock.window.VesselProxyEmbodiment. Institution CI installs neo dev after npm ci (resolve-org-dev), so every job has failed at module link since 19:27Z. The container now imports the class and creates it with the same two seams; its destroy already releases the instance.

* test(visual): restamp the baseline inputs for VesselContainer's class swap (#669)

VesselContainer.mjs is a style-owning input, so its swap moved the stamp. On Darwin at neo dev 9e48135295, FleetCockpitVisual passes 51/51 and AgentCardSynthesisRenderNL passes with no snapshot change; the stamp records the new blob and that engine."
- 2026-10-10T20:27:46Z @tobiu closed this issue

