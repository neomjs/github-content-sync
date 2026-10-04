---
id: 554
title: The compose chip row destroys a chip its stored vnode still names
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-04T18:17:14Z'
updatedAt: '2026-10-04T18:38:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/554'
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
milestone: FM v1
---
# The compose chip row destroys a chip its stored vnode still names

## Context

`OperatorComposeControlsNL` ("toggle-deselect, chip remove, broadcast exclusivity and the radio group hold through real clicks") fails intermittently in the local NL battery. The engine's `workerErrors` fixture catches `App Worker: vdom update failed neo-fm-fleet-cockpit-1 util.VNode.getVnode: Component not found for id: neo-fm-recipient-chip-list-1__0__component`, raised from `util/VNode.mjs` via `createMap`.

Observed rates, full NL batteries, Brain runtime root at the pin `dbd35bc2`:
- Unmodified dev `23cf6e32`: 1 of 6 sequential batteries, then 1 of 5 interleaved with #548's re-test (2026-10-04).
- A `list.Base` plus its Store built eagerly in the cockpit raises it: 5 of 7 in #511 R2 (2026-10-03), 4 of 7 in #548 (2026-10-04). Both PRs moved their list to a lazy build, and the residual is this race.

It was captured twice as a defect-note (#511 R2 and `e72890e0e4f4ffe9` on #548), in two PRs with two different triggering lists. This ticket promotes it.

## The Problem

The chip row is a projection of the picker's selection. When the selection contracts, `createItems` trims the recycled chip pool and destroys each trimmed chip at once (`me.items.pop().destroy()`). The list's own update, which removes the chip from the DOM, is only sent afterwards (`promiseUpdate()`), so for one update cycle the list's stored vnode still holds a `{componentId}` reference to a component the registry no longer knows. Any update that walks that subtree in the meantime, such as a cockpit-level update, resolves the reference through `VNode.getVnode` and throws.

Inference, to be confirmed red-first: the cockpit update is the walker, and more components in the cockpit make it more likely to land inside that window, which matches both rate jumps.

## The Architectural Reality

- `apps/agentos/view/fleet/mailbox/RecipientChipList.mjs`, `createItems`: the trim loop calls `destroy()` with its defaults, `updateParentVdom = false`.
- The engine already covers this case. `component.Base#destroy(updateParentVdom, silent)` with `updateParentVdom = true` calls `VNodeUtil.unlinkRetiredReferences(parent.vnode, {[id]: vnode})`: the parent's stored vnode stops naming the retired component and keeps the vnode it last rendered, so the DOM node stays until the parent's next update diffs it away. `container.Base#removeAt` destroys with `updateParentVdom = true` for the same reason.
- Design authority: the engine's own comment in `component.Base#destroy`, "The parent's stored vnode must stop naming a component the registry no longer knows". The thrown error is the red; no product decision is in question.
- The chips are created by `list.Chip#createItemContent` with `parentId` set to the list, so the list is the parent that call cleans.

## The Fix

In the trim loop, destroy each trimmed chip with `destroy(true, true)`: unlink it from the list's stored vnode, silently, because `createItems` sends the list's update right after.

## Acceptance Criteria

- [ ] AC-1: after the selection contracts, the list's stored vnode names no destroyed chip, and a walk of it resolves every reference (unit; red on the current trim, which leaves the reference).
- [ ] AC-2: `OperatorComposeControlsNL` passes in every one of at least 6 full NL batteries on the fix (local, Brain at the pin).
- [ ] AC-3: the existing compose and chip specs stay green (unit, component, e2e).

## Out of Scope

- The lazy builds in #511 and #548 stay: an idle cockpit carrying no list is right on its own.
- Other component pools trimmed with a bare `destroy()`: `git grep` finds none in `apps/` on dev besides this one. A full census of direct child destroys is not this ticket.

## Related

Captured in #511 R2 and #548. The engine primitive: `component.Base#destroy` and `VNodeUtil.unlinkRetiredReferences` in neo.mjs 13.1.0.

Sweeps: live latest-open 20 Institution issues at 2026-10-04T18:15Z, no equivalent; A2A, the last 30 in all read states, no claim on this scope; Memory Core, no earlier decision on this race; own assignments (#424, #512, #516, #521, #533), none overlaps.

Decision Record impact: `none`.

Origin Session ID: 6b13f348-5848-47a1-8740-c4a9d1dfaea7
Retrieval Hint: "RecipientChipList trimmed chip destroyed before its update getVnode component not found compose NL flake"


## Timeline

- 2026-10-04T18:17:14Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-04T18:17:16Z @neo-opus-ada added the `bug` label
- 2026-10-04T18:17:16Z @neo-opus-ada added the `agent-os` label
- 2026-10-04T18:17:17Z @neo-opus-ada added the `ai` label
- 2026-10-04T18:17:20Z @neo-opus-ada added this to the **FM v1** milestone
- 2026-10-04T18:36:21Z @neo-opus-ada cross-referenced by PR #548
- 2026-10-04T18:56:09Z @neo-opus-ada cross-referenced by PR #556

