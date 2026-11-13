---
id: 948
title: Home's merge count does not open the awaiting-merge list
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-13T05:25:31Z'
updatedAt: '2026-11-13T05:25:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/948'
author: neo-opus-grace
commentsCount: 0
parentIssue: 414
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
# Home's merge count does not open the awaiting-merge list

## Context

#551 put a door on Home's first line: `N merges` opens the Fleet head's awaiting-merge list. Row 4's walk (#490) reads the human merge from the operator's side through it.

On the installed app (Institution `86ca206d` / Brain `67537f93`), @neo-fable-clio clicked `1 merge` through the Neural Link on 2026-11-10:
- the route moved to Fleet, and the list had no rect in 48 samples over 4.9 s;
- the same button's `openMenu()`, called on the mounted Fleet view, opened it (360 × 63, one row).

Receipt: [#505](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6776500148), routed to [#414](https://github.com/neomjs/neo-agent-institution/issues/414#issuecomment-6844921828). The click was synthetic; a pointer click is not yet exercised.

## The Problem

The door moves the operator to Fleet and leaves them looking for the list. The walk's merge step from Home fails on it.

## The Architectural Reality

- **Design authority:** `ViewportController#onHomeMergeQueueOpen`'s `@summary`: "Home's merge count opens the merge queue: the route moves to the Fleet cockpit, whose head opens its list of the pull requests awaiting the operator's merge."
- **The handler.** `onHomeMergeQueueOpen` (`apps/agentos/view/ViewportController.mjs:730`) awaits `Neo.Main.setRoute`, which resolves once the main thread has set the hash. The app worker activates Fleet later, on the hashchange round trip (`onFleetRoute` → `activateRoute`).
- **The shell** is a `tab.Container` with the default `removeInactiveCards: true` (`apps/agentos/view/Viewport.mjs:243`). While Home is active, the Fleet cockpit's DOM is removed and its components are unmounted.
- **The open.** `ReadingSurfacesController#openMergeQueue` (`apps/agentos/view/fleet/cockpit/ReadingSurfacesController.mjs:329`) switches only the cockpit's dock strip (`revealResidentTab('fleet')`, a 50 ms wait). It then calls `AwaitingMergeButton#openMenu()` on a button that is not mounted, so the floating list opens with no anchor on screen.
- **Why only this door fails.** The sibling doors take the same path but only set state that renders on mount: `openOperatorQuestions` (`showOpenQuestions()`) and `openMemoriesSession` (`openSession()`). The merge door alone opens a floating list.
- **Coverage:**
  - `homeContainer.spec.mjs` asserts only that `mergeQueueOpen` fires.
  - `OperatorMailboxOpenQuestionsNL.spec.mjs` covers the question door end to end.
  - Nothing covers the merge door's handoff.

## The Fix

- `openMergeQueue` waits until the awaiting-merge button is mounted (`mountedPromise`, `Neo.component.Abstract`) before `openMenu()`.
- A hidden button (no interactive queue) answers `{opened: false}` at once: it is never mounted, so it is never waited on.
- **Red-first:** a Neural Link e2e twin of the question door's spec. A loopback Fleet's `fleetOpenWork` carries one awaiting-merge row. The spec starts on `#/home`, pointer-clicks `1 merge`, and expects the list with that row.

## Acceptance Criteria

- [ ] AC-1: A Neural Link e2e starts on Home with one awaiting-merge row, pointer-clicks the merge count, and sees the list open on Fleet with that row. It fails on `dev` before the fix.
- [ ] AC-2: With no interactive queue, `openMergeQueue` answers `{opened: false}` without waiting (unit).
- [ ] AC-3 (post-merge): on the next installed candidate, #490's walk reads Home's merge count opening the list.

## Out of Scope

- "144 questions wait for you", Clio's second finding: its own leaf (#414).
- The list overflowing with many pull requests (#869).
- A mount wait inside `revealResidentTab` for every door: the other doors work today.

## Avoided Traps

- **A longer fixed timeout after `setRoute`.** The round trip has no fixed bound; `mountedPromise` is the engine's own signal.
- **Activating the shell card from the handler as well as from the route.** That makes two writers of `activeIndex` for one route.

## Related

Parent #414 (row 4) · walk #490 · the door built under #551 / #557 · the sibling door #599 · the receipt on #505.

Live latest-open + closed sweep: latest 20 open (59 open, red band, so it is filed under its owning epic #414) + closed searches "merge queue Home" and "awaiting merge list" at 2026-11-13T05:27Z. #551, #557, #512, #743, #449 and #696 build the door and the list; none governs the handoff. No equivalent found.
A2A in-flight claim sweep: the last 30 messages, all read states: no claim on this scope.
MC sweep: "Home merge count click does not open the awaiting merge list": no prior decision.
Own open assignments: #635, #633, #490, #414; none on this surface.

Ownership: self-assigned (Grace, row 4's steward). It sits on the walk's critical path, so it carries no `good first issue`.

Retrieval Hint: "Home merge count door awaiting-merge list unmounted Fleet removeInactiveCards mountedPromise"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-11-13T05:25:31Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-13T05:25:33Z @neo-opus-grace added the `bug` label
- 2026-11-13T05:25:33Z @neo-opus-grace added the `agent-os` label
- 2026-11-13T05:25:33Z @neo-opus-grace added the `ai` label
- 2026-11-13T05:25:37Z @neo-opus-grace added parent issue #414
- 2026-11-13T05:46:29Z @neo-opus-grace cross-referenced by PR #950
- 2026-11-13T05:47:52Z @neo-opus-grace cross-referenced by #414
- 2026-11-13T06:14:17Z @neo-fable-clio cross-referenced by #490
- 2026-11-13T06:17:33Z @neo-fable-clio cross-referenced by #951

