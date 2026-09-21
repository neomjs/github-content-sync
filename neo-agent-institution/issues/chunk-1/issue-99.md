---
id: 99
title: 'Keyboard focus is lost when the roster re-sorts: cards and list items carry positional ids, so a reorder replaces the focused node'
state: CLOSED
labels:
  - bug
  - accessibility
  - agent-os
  - ai
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T13:37:58Z'
updatedAt: '2026-09-04T15:34:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/99'
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
closedAt: '2026-09-04T15:34:52Z'
---
# Keyboard focus is lost when the roster re-sorts: cards and list items carry positional ids, so a reorder replaces the focused node

## Context

Found while pinning engine `dev@205bc52f8a` (#98). With neomjs/neo#18269 merged, the roster's "Online first" sorter finally evaluates records instead of raw rows, so a joining agent that ranks ahead now sorts ahead — and the `FleetGridKeyboardA11y` reorder step turned red 3/3 (green 2/2 on `ab5235e69a`, the commit before the engine fix). The arm was passing on pin 5 for the wrong reason: a raw joiner had no `tierRank`, sorted LAST, and Charlie never changed index — the reorder it claims to witness never happened.

Live latest-open sweep: checked the latest 20 open issues at 2026-09-04T13:37Z; no equivalent found. A2A claim sweep (12 most recent messages, all read states, 13:03Z): no claim on the roster's card identity. Memory Core: no prior decision on card identity across a sort.

## The Problem

Measured on pin 6 with the arm's two id assertions replaced by probes (`FleetGridKeyboardA11yProbeNL`, scratch):

- Charlie's card id `neo-fm-fleet-roster-list-1__1__component` → `…__2__component`; its `li` `…__1` → `…__2`.
- The focused native Button (`Stop Charlie`, `data-ref="control-toggle"`) is **inactive** after the reorder: `toBeFocused()` fails — keyboard focus is lost the moment the roster re-sorts around the user.

Root cause, read in source: `Neo.list.Component#sortItems` nulls every pooled component's id (`item.setSilent({id: null})`) and reorders `items` to follow the records; the Animate plugin's `sortComponentList` then moves the DOM by transforms and, after `transitionDuration`, calls `createItems()`; the roster's `createItemContent(record, index)` re-seats the same card instance with `id: me.getComponentId(index)` — a POSITIONAL id. The instance survives (the roster's stated intent), but its DOM id is renumbered, so the vdom engine replaces the node — and the `li` id (`getItemId`) renumbers the same way. Focus cannot survive a node replacement.

## The Architectural Reality

- `apps/agentos/view/fleet/roster/List.mjs` extends `Neo.list.Component`; its pool is by index, "re-seated via `record` on reuse (the calendar component-list pattern)". That pattern recycles components by position by design — fine for a calendar, wrong for a surface that promises "the same AgentCard and exact native Button move" (the file's own JSDoc, the A11y arm's contract, #17553).
- `getComponentId(index)` has no other caller in the engine or the roster; the `li` id comes from `list.Base#createItem` → `getItemId(getRecordId(record))`, which `list.Component` resolves positionally.
- The Animate plugin needs stable node ids across a sort to MOVE nodes; today the move works visually only because the post-transition `createItems()` re-creates ids after the transform settles.

## The Fix

Give roster cards and their list items record-stable ids: in `roster/List.mjs`, override `sortItems` to reorder the pool without nulling ids, and assign `id` in `createItemContent` only at creation (derive it from the record key — `${listId}__${agentId}__component` — not the index); override `getItemId` to key by record id so the `li` is stable too. If the engine's `list.Component` should offer this as a mode (`stableItemIds: true`), that is a neomjs/neo leaf filed from this one; the roster does not wait for it.

## Acceptance Criteria

- [ ] AC-1 `FleetGridKeyboardA11y`'s reorder step is green on engine `dev@205bc52f8a` or later: Charlie's card id and `li` id unchanged across the reorder, the focused native Button still focused, `aria-selected` intact, selection unchanged.
- [ ] AC-2 A unit arm in `roster/container.spec.mjs` (or a sibling) asserts the card id and the `li` id of an existing agent survive `store.add` of a joiner that sorts ahead — red-first on the current roster.
- [ ] AC-3 The Animate move still animates (no rebuild flash): the existing "same instance, no rebuild" arm stays green.

## Out of Scope

- The engine's `list.Component` positional pattern itself (calendar and others rely on it) — a `stableItemIds` mode is a separate neo leaf if wanted.
- The pin (#98) ships with this arm failing-honest and this ticket as its owner.

## Related

#98 (the pin that surfaced it) · #17553 (the A11y contract the arm pins) · neomjs/neo#18269 (the engine fix that made the sort truthful) · #78 / PR #95 (the sibling consumer correction).

Ownership: unowned at filing — the pin lane (#98) comes first; the author takes it next unless a peer claims it.

Origin Session ID: e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

Retrieval Hint: `query_raw_memories("roster keyboard focus lost reorder list.Component positional ids sortItems createItemContent pin 6")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3


## Timeline

- 2026-09-04T13:37:59Z @neo-fable-clio added the `bug` label
- 2026-09-04T13:38:00Z @neo-fable-clio added the `accessibility` label
- 2026-09-04T13:38:00Z @neo-fable-clio added the `agent-os` label
- 2026-09-04T13:38:00Z @neo-fable-clio added the `ai` label
- 2026-09-04T13:38:00Z @neo-fable-clio added the `testing` label
- 2026-09-04T13:55:50Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T14:06:06Z @neo-fable-clio cross-referenced by PR #101
- 2026-09-04T14:15:32Z @neo-fable-clio referenced in commit `dd1f2ad` - "test(agentos): the card-identity arms sort by name, so the joiner leads on every pinned engine (#99)"
- 2026-09-04T14:23:43Z @neo-fable-clio cross-referenced by PR #102
- 2026-09-04T14:40:06Z @neo-fable-clio referenced in commit `908c63c` - "fix(agentos): roster cards and list items key by record, so a re-sort moves the focused node instead of replacing it (#99)"
- 2026-09-04T14:40:06Z @neo-fable-clio referenced in commit `776c201` - "test(agentos): the card-identity arms sort by name, so the joiner leads on every pinned engine (#99)"
- 2026-09-04T14:40:06Z @neo-fable-clio referenced in commit `300a0c8` - "fix(agentos): roster cards retire with their records, ids live in disjoint DOM-safe namespaces, and the sort moves the same nodes through the plugin (#99)"
- 2026-09-04T14:57:48Z @neo-fable-clio referenced in commit `0dc4c00` - "fix(agentos): roster cards retire on the mutation that removed their records — the store's event, never a projection read inside load (#99)"
- 2026-09-04T14:57:52Z @neo-fable-clio cross-referenced by #103
- 2026-09-04T15:25:57Z @neo-fable-clio referenced in commit `1b1604d` - "fix(agentos): roster cards and list items key by record, so a re-sort moves the focused node instead of replacing it (#99)"
- 2026-09-04T15:25:57Z @neo-fable-clio referenced in commit `0e492d0` - "test(agentos): the card-identity arms sort by name, so the joiner leads on every pinned engine (#99)"
- 2026-09-04T15:25:57Z @neo-fable-clio referenced in commit `5a68ff2` - "fix(agentos): roster cards retire with their records, ids live in disjoint DOM-safe namespaces, and the sort moves the same nodes through the plugin (#99)"
- 2026-09-04T15:25:57Z @neo-fable-clio referenced in commit `4a31607` - "fix(agentos): roster cards retire on the mutation that removed their records — the store's event, never a projection read inside load (#99)"
- 2026-09-04T15:25:57Z @neo-fable-clio referenced in commit `3e434b4` - "test(agentos): the card-identity arms observe the production rebuild — no manual createItems between a store event and its assertion (#99)

Round 2 of the PR #101 review: the four createItems(true) rescues after store.add left the identity and retirement arms; the store's own load, fired inside the mutation, rebuilds the list synchronously, so a rebuild the chain did not perform stays visible. Restamp: the rebase onto the pin-6 dev took dev's stamp while List.mjs still differs."
- 2026-09-04T15:25:57Z @neo-fable-clio referenced in commit `436cd6d` - "docs(agentos): README battery truth — one stated red (#103); the A11y reorder step is green on the pin-6 Engine (#99)"
- 2026-09-04T15:34:52Z @tobiu closed this issue
- 2026-09-04T15:34:52Z @tobiu referenced in commit `3d6c36a` - "Merge pull request #101 from neomjs/agent/99-roster-stable-card-ids

fix(agentos): roster cards and list items key by record, so a re-sort moves the focused node instead of replacing it (#99)"

