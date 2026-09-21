---
id: 98
title: 'Engine pin bump to 205bc52f8a: the projection inside the mutation, the SharedWorker join replay, the dock restore seams'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - build
  - dependencies
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T13:04:44Z'
updatedAt: '2026-09-04T15:15:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/98'
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
closedAt: '2026-09-04T15:15:52Z'
---
# Engine pin bump to 205bc52f8a: the projection inside the mutation, the SharedWorker join replay, the dock restore seams

## Context

The engine's `dev` moved seven commits past pin 5 (`35d468b51f`, #90 / PR #91) to `205bc52f8a`, and one of them is this product's own downstream fix: neomjs/neo#18269 (neo PR #18273, merged 2026-09-04T13:01Z) — the unfiltered projection written inside the mutation and eager stores hydrating before the splice — filed from the Fleet Manager's roster measurement (#78 / PR #95). The rest of the delta, from `git log 35d468b51f..205bc52f8a`:

- `2ee23d7801` fix(worker): a window joining a running SharedWorker receives the replayed remotes (#18265) — the tear-out / pop-out join path this cockpit exercises.
- `ab5235e69a` feat(dock): a workspace answers the multi-window restore seams (#18253).
- `0835abecb9` fix(grid): clearing isLoading re-projects, so a load committed while loading is not lost (#18259).
- `f58d233db4` ci: the shared PR baseline (#18271); `27e19edda7` docs(themes) (#18145); `0f01489e38` test(buildScripts) (#17922) — no product surface.

Live latest-open sweep: checked the latest 20 open issues at 2026-09-04T13:03Z; no pin ticket open (#90 and #81 are the closed predecessors). A2A claim sweep (12 most recent messages, all read states): no pin claim. Memory Core sweep: the pin recipe and its two recorded misses (pin 4's stale goldens under a green stamp; the interactive theme build) are the only prior decisions.

## The Problem

PR #95 ships the roster's consumer discipline with a stated residual owned by neomjs/neo#18269: `FleetGridScaleNL` passes its store half and fails its DOM half (12 cards rendered for 6 rows — the engine's raw/record duality), and `onRosterStoreLoad` carries a microtask deferral marked "removable once the Engine mirrors inside the mutation". Until the engine that fixes this is pinned, the README's battery paragraph keeps one stated red, and every dock/tear-out fix on `dev` since pin 5 is invisible to this product.

## The Architectural Reality

- The engine is a GitHub-SHA dependency: `package.json` `"neo.mjs": "github:neomjs/neo#<sha>"` + `package-lock.json`; the visual-baseline stamp (`buildScripts/checkVisualBaselines.mjs`) carries an `engine` axis read from the lock, so a pin is a stamp input.
- An engine pin re-renders every golden (pin 4 shipped stale goldens under a green stamp): the visual suite (`test-visual`, seven arms) and the synthesis capture (`AgentCardSynthesisRenderNL`) are re-captured by deletion, not `--update-snapshots` (pixelmatch is blind in the dark range), and each diff is read by eye.
- The theme build is interactive: `npm run build-themes -- -n -e dev -t all` before any visual run (the harness refuses stale CSS).
- The microtask deferral lives in PR #95's `apps/agentos/view/fleet/roster/Controller.mjs`; retiring it is a change on top of #95, so the pin branches from `dev` after #95 merges (no stacked PR — a stacked child gets no head CI).

## The Fix

Bump the pin to `205bc52f8ad49a8f89e0e50c9714ba2df7a433f4` (`npm install neo.mjs@github:neomjs/neo#205bc52f8ad49a8f89e0e50c9714ba2df7a433f4`), fix whatever the unit tree and the NL battery turn red under the new engine, re-capture the visual + synthesis goldens (by eye), retire `onRosterStoreLoad`'s microtask deferral (the fold decides synchronously again, as the engine now guarantees the projection inside the mutation), restamp after staging, and rewrite the README battery paragraph — no stated red left if the witness is green in full.

## Acceptance Criteria

- [ ] AC-1 `package.json` + `package-lock.json` pin `205bc52f8ad49a8f89e0e50c9714ba2df7a433f4`; `npm run test-unit` green.
- [ ] AC-2 `test-e2e:nl` under the Brain root: `FleetGridScaleNL` green in full (store half and DOM half — 6 cards for 6 rows), the rest of the battery at its pin-5 state or better; every red attributed.
- [ ] AC-3 Visual + synthesis goldens re-captured by deletion under a fresh non-interactive theme build, every diff read by eye and named in the PR (what moved, why); `check-visual-baselines` green on the pushed tree as its own command.
- [ ] AC-4 The microtask deferral in `onRosterStoreLoad` is retired and the unit arm that pinned it (the batch that crosses the threshold) asserts the synchronous fold; README battery paragraph truth-synced.

## Out of Scope

- Consuming the new dock restore seams (#18253) or the SharedWorker join replay (#18265) beyond what the existing battery exercises — separate leaves if a witness asks for them.
- Any engine change: a red that needs the engine goes back as a neo ticket, the pin ships with the red stated (the pin-5 shape).

## Related

#10 (parent) · #90 / PR #91 (pin 5) · #81 (pin 4) · #78 / PR #95 (the residual this pin retires) · neomjs/neo#18269 / PR #18273 (the engine fix) · neomjs/neo#18265 / PR #18266 · neomjs/neo#18253.

Origin Session ID: e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

Retrieval Hint: `query_raw_memories("engine pin 6 205bc52f8a institution goldens re-capture microtask deferral retired FleetGridScaleNL green")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3


## Timeline

- 2026-09-04T13:04:46Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T13:04:46Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T13:04:46Z @neo-fable-clio added the `agent-os` label
- 2026-09-04T13:04:46Z @neo-fable-clio added the `ai` label
- 2026-09-04T13:04:46Z @neo-fable-clio added the `build` label
- 2026-09-04T13:04:47Z @neo-fable-clio added the `dependencies` label
- 2026-09-04T13:04:47Z @neo-fable-clio added the `testing` label
- 2026-09-04T13:38:00Z @neo-fable-clio cross-referenced by #99
- 2026-09-04T13:44:43Z @neo-gpt cross-referenced by PR #95
- 2026-09-04T13:49:12Z @neo-fable-clio cross-referenced by #78
- 2026-09-04T13:51:25Z @neo-fable-clio referenced in commit `729a5ff` - "docs(readme): the battery's stated red names its owner — the engine pin #98 carries the merged fix (#78)"
- 2026-09-04T14:06:06Z @neo-fable-clio cross-referenced by PR #101
- 2026-09-04T14:23:43Z @neo-fable-clio cross-referenced by PR #102
- 2026-09-04T14:57:52Z @neo-fable-clio cross-referenced by #103
- 2026-09-04T15:01:11Z @neo-fable-clio referenced in commit `3422c0a` - "docs(agentos): the roster's two projection explanations name the pin-6 invariant; the battery's reds name their owners (#98)"
- 2026-09-04T15:15:52Z @tobiu referenced in commit `a8b469e` - "Merge pull request #102 from neomjs/agent/98-engine-pin-205bc52f8a

chore(deps): engine pin 6 — dev@205bc52f8a: the projection inside the mutation reaches the Fleet Manager, FleetGridScaleNL goes green in full (#98)"
- 2026-09-04T15:15:53Z @tobiu closed this issue
- 2026-09-04T17:33:03Z @neo-fable-clio cross-referenced by #107
- 2026-09-12T10:10:47Z @neo-fable-clio cross-referenced by #120
- 2026-09-12T11:42:54Z @neo-fable-clio cross-referenced by PR #125
- 2026-09-18T10:40:56Z @neo-fable-clio cross-referenced by #151
- 2026-09-18T14:17:27Z @neo-fable-clio cross-referenced by #157

