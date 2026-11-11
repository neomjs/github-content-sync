---
id: 923
title: The ObservatoryCanvas spec's bare host has no canvasId — every PR red
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - regression
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-11-11T20:08:56Z'
updatedAt: '2026-11-11T21:26:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/923'
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
blocking: []
closedAt: '2026-11-11T21:16:48Z'
---
# The ObservatoryCanvas spec's bare host has no canvasId — every PR red

## Context

Since engine `2ca156c0` (neomjs/neo#20792, merged 19:21Z), `Isolated Institution` fails 3 unit tests in `test/playwright/unit/apps/agentos/view/fleet/goldenpath/observatoryCanvas.spec.mjs`:

```
TypeError: Cannot read private member #configs from an object whose class did not declare it
```

CI installs the engine at `dev` (`npm run resolve-org-dev`), so every open PR is red on that required check. It first showed on #893's rerun at 20:00Z (run 43030374050, attempt 2). That rerun was meant to collect the E2E half of the fix, neomjs/neo#20793, which had just merged.

**One-variable receipt.** Run locally, Brain `c27e8b9b` for both:
- engine `8bd84b8e` (the parent of `2ca156c0`): 4 passed.
- engine `e53db3cb` (`dev`): 3 failed, 1 passed, with the same TypeError.

Live latest-open + closed sweep, 20:09Z:
- I checked the latest 20 open issues (58 open, red band).
- Closed searches: `canvasId`, `observatoryCanvas`, `bare host`, `private member`, and the baseline pin. No equivalent and no prior exclusion; #794 is the precedent.
- The 30 newest A2A messages, all read-states, carry no claim.
- MC has no prior decision.
- My own assignments are #917, #516 and #424, none overlapping.

## The Problem

`makeCanvas()` builds the host as `Object.create(ObservatoryCanvas.prototype)`, and it shadows the reactive configs it reads with own data properties. Since #20792, every renderer call goes through `SharedCanvas#getCanvasId()`. That method reads `this.canvasId`, a config the bare host does not shadow. The getter therefore reaches `Neo.core.Base`'s `#configs` on an object the constructor never initialized.

#20792 gave its own three bare-host specs an own `canvasId`. This consumer spec was outside its reach.

## The Architectural Reality

- `src/app/SharedCanvas.mjs` `getCanvasId()` (engine `dev`): `if (!me.canvasId) me.canvasId = me.id`. Both the read and the write go through the reactive config.
- `src/core/Base.mjs`: `#configs = {}`, a private field that only construction initializes.
- The bare host is deliberate. The spec's JSDoc calls it "a prototype host whose own data properties shadow the reactive configs", so the pointer handlers run as the class's own methods without a worker.

**The two required checks deadlock.**
- Ruleset 21641205, updated 2026-11-11, requires `Shared PR Baseline / PR baseline`. Only the v1.x reusable baseline emits that context, and `dev` pins `@v0.1.84`.
- #893 moves the pin to `v1.3.7`, but it cannot carry this spec fix: it is a Dependabot branch, and a foreign push breaks its automerge.
- A spec-only PR would lack the `PR baseline` context.

Neither change can merge alone.

## The Fix

One PR that carries both changes:

1. In `observatoryCanvas.spec.mjs`, add `canvasId: 'canvas-1'` to `makeCanvas()`'s shadowed own properties, as #20792 did for the engine's bare hosts.
2. In `.github/workflows/shared-pr-baseline.yml` and `dependabot-automerge.yml`, move `@v0.1.84` → `@v1.3.7`. This is #893's exact change, so the PR emits `PR baseline`. Dependabot then closes #893 as already applied.

## Acceptance Criteria

- [x] `observatoryCanvas.spec.mjs` passes 4/4 against engine `dev` (`e53db3cb` or later) with the Brain at `dev`. Green in #924's `Isolated Institution` and in `dev` CI run 43101669337 (engine `781af05f`).
- [x] The PR is green on all four required contexts, `Isolated Institution` and `Shared PR Baseline / PR baseline` included. #924 merged green at `32306bb0`.
- [x] *(post-merge)* Institution `dev` CI is green on its next run. **Met 2026-11-11:** run 43101669337 on `a8393687` (engine `dev` `781af05f`, Brain `c27e8b9b`): `Isolated Institution` and `Explicit Brain contract` are green; unit 2117, components 8, E2E 17.

## Out of Scope

- Making `SharedCanvas#getCanvasId()` tolerate an unconstructed host. The engine's own bare-host specs shadow `canvasId` instead.
- #914's engine lock, which points at a pre-fix commit. I recreate #914 after this lands.
- Refreshing #915 and #916 on the new `dev`.

## Related

- neomjs/neo#20792 is the cause. neomjs/neo#20793 fixed the E2E half and has merged.
- #893 is the pin bump this PR absorbs.
- #794 is the precedent: an engine change turned every PR red, and the fix landed here.

Self-assigned (Ada). It blocks every open PR, so it is not a good first issue.

Origin Session ID: 9a5ba5cf-1097-4257-b5fa-42948b2161b4
Retrieval Hint: "observatoryCanvas bare host canvasId private member configs every PR red PR baseline pin deadlock"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-11T20:08:56Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-11T20:08:58Z @neo-opus-ada added the `bug` label
- 2026-11-11T20:08:58Z @neo-opus-ada added the `agent-os` label
- 2026-11-11T20:08:58Z @neo-opus-ada added the `ai` label
- 2026-11-11T20:08:58Z @neo-opus-ada added the `regression` label
- 2026-11-11T20:08:59Z @neo-opus-ada added the `testing` label
- 2026-11-11T20:12:29Z @neo-opus-ada cross-referenced by PR #924
- 2026-11-11T20:36:13Z @neo-fable-clio cross-referenced by PR #916
- 2026-11-11T21:16:47Z @tobiu referenced in commit `a839368` - "test(fleet): the Observatory spec's bare host shadows canvasId (#923) (#924)

* test(fleet): the Observatory spec's bare host shadows canvasId (#923)

Engine #20792 routes every renderer call through SharedCanvas#getCanvasId(), which reads the reactive canvasId config. makeCanvas() builds the host with Object.create(prototype), so that read reached core.Base's #configs on an unconstructed object and threw. The host now shadows canvasId like the other configs it reads, as #20792 did for the engine's bare hosts.

* ci: call the shared PR baseline at Skills v1.3.7 (#923)

The dev ruleset now requires the aggregated 'Shared PR Baseline / PR baseline' context, which only the v1.x reusable baseline emits. Same change as Dependabot's #893, carried here because neither that branch nor a spec-only branch can pass both required checks alone."
- 2026-11-11T21:16:49Z @tobiu closed this issue
- 2026-11-11T21:27:36Z @neo-opus-ada cross-referenced by PR #20793

