---
id: 948
title: Home's merge count does not open the awaiting-merge list
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-13T05:25:31Z'
updatedAt: '2026-11-13T14:36:07Z'
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
closedAt: '2026-11-13T14:36:07Z'
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
- 2026-11-13T13:53:28Z @neo-opus-grace referenced in commit `91a6a00` - "fix(agentos): Home's merge door ends its request when the Fleet visit ends (#948)

The door waited on the button's mount promise, which the engine keeps across an unmount, so a request whose visit ended before the mount opened the list on a later ordinary Fleet visit. The shell's controller now hands the cockpit a signal scoped to the visit (it aborts when the shell activates another keeper-view), and the cockpit's wait answers not opened when that signal aborts or the button hides first, releasing both listeners."
- 2026-11-13T13:53:28Z @neo-opus-grace referenced in commit `9775f99` - "test(agentos): restamp the two controller inputs of the Darwin goldens (#948)

npm run test-goldens passed all 8 capture specs at this head and no golden changed, so only the entries of ViewportController.mjs and ReadingSurfacesController.mjs move; the engine line stays."
- 2026-11-13T14:36:07Z @tobiu referenced in commit `7fc84e5` - "fix(agentos): Home's merge count opens the list once Fleet is mounted (#948) (#950)

* fix(agentos): Home's merge count opens the list once Fleet is mounted (#948)

The shell drops an inactive view's DOM, so the door opened the floating
list on an unmounted button whenever the open beat the route's mount.
openMergeQueue now waits for the button's mountedPromise; a hidden
button is never mounted and answers at once. The visual stamp moves
only this input's entry: all 8 Darwin capture specs pass at this head.

* fix(agentos): Home's merge door ends its request when the Fleet visit ends (#948)

The door waited on the button's mount promise, which the engine keeps across an unmount, so a request whose visit ended before the mount opened the list on a later ordinary Fleet visit. The shell's controller now hands the cockpit a signal scoped to the visit (it aborts when the shell activates another keeper-view), and the cockpit's wait answers not opened when that signal aborts or the button hides first, releasing both listeners.

* test(agentos): restamp the two controller inputs of the Darwin goldens (#948)

npm run test-goldens passed all 8 capture specs at this head and no golden changed, so only the entries of ViewportController.mjs and ReadingSurfacesController.mjs move; the engine line stays."
- 2026-11-13T14:36:08Z @tobiu closed this issue

