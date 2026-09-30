---
id: 634
title: The fleet activity read drops a page offset before it reaches the mailbox
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T13:16:16Z'
updatedAt: '2026-09-30T15:02:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/634'
author: neo-opus-grace
commentsCount: 1
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
closedAt: '2026-09-30T15:02:12Z'
---
# The fleet activity read drops a page offset before it reaches the mailbox

## Context

Institution `#349` (FM v1 milestone, roadmap row 4 in `#335`) makes the cockpit's activity feed read older events on scroll. Its premise, that "the Institution can page without a Brain change for the A2A lane", does not hold at Brain `dev` `8a42800`. Verified at intake on 2026-09-30.

## The Problem

`ai/services/fleet/wireFleetActivityReadSource.mjs:43` binds the A2A slot as `params => readFleetA2AActivitySnapshot({listMessages, limit: params.limit})`, so only `limit` survives. The adapter itself already pages. `readFleetA2AActivitySnapshot` spreads `listArgs` into `listMessages(...)` and reports `pageOffset`, `totalCount` and `truncated` (`fleetA2AActivityAdapter.mjs:60-72`). Both bindings pass args through unchanged: `MailboxService.listMessages` in process, and `planeClient.listMessages` → `list_messages` in plane mode (`devFleetServer.mjs:309`, `:323`). An `offset` the cockpit sends is dropped at the wiring, so every "older" page returns the newest page again.

The composer has a second gap: it bounds both slots' events together (`boundEvents(contributions.flatMap(...), bound)`, `fleetActivityComposer.mjs:337-343`). The PR/lane slot has no offset. A history page would therefore mix the PR lane's newest events into the A2A page and cut A2A rows at the bound.

## The Architectural Reality

- `FleetControlBridge.fleetActivity(params)` → `activitySource.readActivitySnapshot(params)` (`fleetActivityComposer.mjs:330`). The composer normalizes `params.limit` at the wire boundary (`normalizeBound`) and asks every slot through `readSlot`, which contains each slot's failure.
- The wire envelope's `params` is free-form (`src/fleet/contract/wire.mjs:74`, `optional: ['params']`), so no protocol version or capability changes.
- The bridge's JSDoc says `params` carries `{limit, since, until}`. The wiring forwards neither `since` nor `until`, and the adapter applies both to mapped events after the fetch, not to the query.

## The Fix

1. `makeReadA2ASnapshot` forwards the page offset as `listArgs.offset`, omitted at 0, so today's request is unchanged.
2. `readActivitySnapshot` normalizes `params.offset` at the wire boundary (a non-negative integer, else 0), beside `limit`. It also accepts `params.slots`, a subset of `FLEET_ACTIVITY_SLOTS`: only those slots are read, and the composed capability covers only them. An absent, empty or all-unknown selection reads every slot, as today.
3. `FleetControlBridge.fleetActivity`'s JSDoc names what `params` actually carries: `limit`, `offset` and `slots`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleetActivity` `params.offset` | `wireFleetActivityReadSource.makeReadA2ASnapshot` (`:42-44`), `fleetActivityComposer.readActivitySnapshot` (`:330`) | A non-negative integer pages the A2A lane through `listMessages({offset})` | absent or invalid → 0, today's newest page | `FleetControlBridge.fleetActivity` JSDoc | unit: the offset reaches an injected `listMessages`; a plane-client stub receives it too |
| `fleetActivity` `params.slots` | `fleetActivityComposer.readActivitySnapshot` | A subset of `FLEET_ACTIVITY_SLOTS` reads only those slots; the capability composes over them | absent, empty or unknown-only → every slot (today) | same JSDoc | unit: `slots: ['a2a']` never calls the PR/lane reader |

## Acceptance Criteria

- [ ] AC-1 `fleetActivity({offset: 50, limit: 50})` reaches `listMessages` with `offset: 50` in both bindings, and the snapshot's A2A counts report that page (`pageOffset` 50, `total` complete).
- [ ] AC-2 `fleetActivity({slots: ['a2a'], offset: 50})` reads only the A2A slot. The PR/lane reader is not called, and the capability reflects the A2A slot alone.
- [ ] AC-3 Without `offset` or `slots`, the snapshot is unchanged (the existing composer and wiring specs stay green).
- [ ] AC-4 An invalid `offset` (negative, fractional, string) reads as 0, and unknown slot names are ignored.

## Out of Scope

Paging the PR/lane slot (its adapter slices by `limit` only). `since`/`until` as query bounds. The cockpit's history segment, head and goldens (Institution `#349`).

## Avoided Traps

- **A new wire verb for history.** The read is the same and `params` is free-form. A verb would duplicate the policy row, the bridge method and the client surface for one parameter.
- **Offsetting the PR/lane slot silently.** Its adapter would ignore the offset and return its newest events, which is the bug this ticket removes for the A2A lane.
- **A timestamp cursor.** `list_messages` has no time bound, and `since`/`until` filter after the fetch. Offset paging plus id dedup at the store is honest, because the store keys on `eventId`.

## Related

Institution `#349` (the consumer; blocked by this) · `#413` (the feed reads every corpus origin) · `#585` (plane-attached PR/lane activity) · `#563` (`list_messages` page cost)

Decision Record impact: none.
Structure-map gate: `npm run ai:structure-map -- --files --loc` at `8a42800` (exit 0). The existing owners are in `ai/services/fleet/` (`wireFleetActivityReadSource.mjs`, `fleetActivityComposer.mjs`), and no new file is needed.
Sweeps at 2026-09-30T13:15Z: the live latest 20 open Brain issues, with no equivalent (closest: `#459`, `#585` closed, `#413` closed). A2A claims in the last 60 minutes, all read states: none on this scope beyond my claim of the Institution consumer. Memory Core: no prior decision on activity paging. Own open Brain assignments: none on this surface.

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636
Retrieval Hint: "fleet activity offset paging A2A slot listArgs wireFleetActivityReadSource composer slots"

## Timeline

- 2026-09-30T13:16:17Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T13:16:17Z @neo-opus-grace added the `enhancement` label
- 2026-09-30T13:16:18Z @neo-opus-grace added the `ai` label
- 2026-09-30T13:16:18Z @neo-opus-grace added the `agent-os` label
- 2026-09-30T13:16:48Z @neo-opus-grace cross-referenced by #349
- 2026-09-30T13:27:04Z @neo-opus-grace cross-referenced by PR #636
- 2026-09-30T13:30:09Z @neo-opus-grace referenced in commit `3304a42` - "test(fleet): the sync-throw arm says what it proves, not who asked (#634)"
- 2026-09-30T14:55:00Z @tobiu referenced in commit `ff018a4` - "Merge pull request #636 from neomjs/grace/634-activity-page-offset

feat(fleet): the activity read pages the mailbox and reads only the lanes it names (#634)"
### @neo-opus-grace - 2026-09-30T15:02:12Z

Delivered by #636, merged to `dev` at ff018a4.

- 2026-09-30T15:02:13Z @neo-opus-grace closed this issue
- 2026-09-30T18:14:15Z @neo-gpt-emmy cross-referenced by PR #357

