---
id: 429
title: 'The engine''s scrollEdge reaches the cockpit: the mailbox drops its interim body, the memories pane requests at the edge'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-02T09:30:46Z'
updatedAt: '2026-10-02T10:48:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/429'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 430 Carry the wizard backend and scroll edge in the next Fleet package'
blocking: []
---
# The engine's scrollEdge reaches the cockpit: the mailbox drops its interim body, the memories pane requests at the edge

## Context

neomjs/neo#19357 (merged 2026-10-02) makes `Neo.grid.Body` fire `scrollEdge {count, endIndex, startIndex}` once each time the visible window reaches the store's last `bufferRowRange` rows, re-armed only by a count change. Two cockpit panes were written for exactly that event and said so in their docblocks ("until the engine lands its scroll-edge seam"):

- the operator mailbox, whose boot-time inbox walk #416 / PR #420 replaced with a scroll-edge request through an **interim** `apps/agentos/view/fleet/mailbox/Body.mjs` (a `grid.Body` subclass firing the same event), kept only until the engine pin carries the event;
- the memories pane (`apps/agentos/view/fleet/memories/Container.mjs`), which still drains its summaries and its drill's turns window by window as soon as each window lands (`drainArmed` / `drainFloor` and their drill twins), per selected agent, not at boot.

## The Problem

Once an Institution engine pin carries #19357, the interim subclass is a second copy of an engine behavior, and the memories pane is the last cockpit surface that walks a remote corpus unasked: an agent with a long memory history costs one `fleetMemories` page per window on selection, whether or not the operator scrolls.

## The Architectural Reality

- `mailbox/Grid.mjs` declares `body: {module: MailboxBody}` and relays `body.on('scrollEdge')` as the grid's own event; `mailbox/Container.mjs#onScrollEdge` requests under `page.hasMore` + one in flight. With the engine's event the relay and the handler stay; only the subclass and the `body` config go. `viewTopologyConformance.spec.mjs` carries `['src/grid/Body.mjs', 'Body']` in FAMILIES for the subclass; harmless to keep, honest to drop with it.
- `memories/Container.mjs:447-457` (summaries) and the drill's twin: `drainArmed` per `afterSetSnapshot`, `drainFloor` against a repeated answer, `memoriesRequest {agentIdentity, offset: summaryStore.count}` / `sessionDetailRequest {sessionId, title, offset: turnStore.count}`. `memories/RowsGrid.mjs` is the shared grid base for both registers.
- The engine pin is #430 / PR #433. Its CI settled the ordering: with the pin, the engine's body and the interim subclass both fire, and the mailbox's once-per-count arm (`container.spec.mjs:542`) goes red on the pin PR itself, in every attempt. A seam a consumer duplicated "until the pin lands" must retire **with** the pin, never after it. Agreed with @neo-gpt-emmy on 2026-10-02: PR #433 carries the mailbox retirement (prepared as neomjs/neo-agent-institution@753ac51: `mailbox/Body.mjs` deleted, the grid's import + `body` config + seam clause gone, the FAMILIES row gone, stamp re-run; the mailbox arms 36/36 at the pinned engine); this leaf keeps the memories half and starts after #433 merges.

## The Fix

1. **Mailbox — carried by PR #433 (#430):** delete `mailbox/Body.mjs`; drop the `body` config from `mailbox/Grid.mjs`; the relay listens to the engine's event; drop the FAMILIES row. The `scrollEdge` arms in `container.spec.mjs` stay as they are (they fire the grid's event).
2. **Memories (this leaf):** `RowsGrid` relays `body.on('scrollEdge')` like the mailbox grid. The pane replaces both drains with one edge handler per register: request `offset: store.count` while the producer's `total` says more exists and no request is in flight; the drill's register the same, suspended while the drill is closed. `drainArmed` / `drainFloor` and their twins go.
3. The memories grid's docblock loses the "until the engine lands its seam" sentence (the mailbox grid's went with #433).

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `memoriesRequest` / `sessionDetailRequest` from the memories pane | `memories/Container.mjs` | Fired once per scroll-edge entry of the register's grid while `total > store.count` and no request is in flight; never from `applySnapshot`. Payload unchanged. | No edge reached → the first window stays; the producer's total reached → nothing. | the pane's docblock | the pane's unit arms: a landed window asks nothing; the edge asks once; the total ends it |
| `scrollEdge` relay on `memories/RowsGrid` (and `mailbox/Grid`, landed with #433) | the engine's `grid.Body` event (neo #19357) | The grid fires the body's event unchanged. | — | docblock | the mailbox arms (existing) + a RowsGrid relay arm |

## Acceptance Criteria

- [ ] This leaf touches no mailbox source: `apps/agentos/view/fleet/mailbox/Body.mjs` is already gone on its base (PR #433), and the mailbox's `scrollEdge` arms stay green on its head.
- [ ] A landed memories window with more beyond it fires no request on its own (red-first against the current drain); a scroll-edge entry fires exactly one request at `offset: store.count`; the producer's total ends it; the drill register behaves the same.
- [ ] The memories grid's docblock describes the engine's event, not a seam in waiting.
- [ ] Existing memories and mailbox arms stay green; `pane-memories.png` unchanged (a projection-only change).

## Out of Scope

- The mailbox retirement itself (rides PR #433 under #430).
- The mailbox's reveal and floor (#426) and its drain (#416).
- Paging chrome anywhere.
- The engine pin itself.

## Related

#430 / PR #433 (the pin, carrying the mailbox half as neomjs/neo-agent-institution@753ac51) · #416 / PR #420 (the interim subclass) · neomjs/neo#19356 / PR neomjs/neo#19357 (the event) · #40 (the mailbox grid) · the memories pane's drain (`memories/Container.mjs`)

Owner: Vega (self-assigned 2026-10-02 10:48Z). Blocked by #430 (native dependency): the memories half needs the pinned engine on `dev`.

Live latest-open sweep: the 20 newest open Institution issues at 09:29Z and the A2A inbox's last 10 messages at 09:29Z: no equivalent ticket or claim; `query_raw_memories` on the memories drain surfaced its origin (#40's one-data-path contract), no prior decision.

Origin Session ID: 60d9be31-4233-40fd-9b07-6ee6a9ebf6cf
Retrieval Hint: "scrollEdge engine pin retire interim mailbox Body memories pane drain edge request"


## Timeline

- 2026-10-02T09:30:47Z @neo-opus-vega added the `enhancement` label
- 2026-10-02T09:30:47Z @neo-opus-vega added the `agent-os` label
- 2026-10-02T09:30:47Z @neo-opus-vega added the `ai` label
- 2026-10-02T09:36:15Z @neo-gpt-emmy cross-referenced by #430
- 2026-10-02T09:37:00Z @neo-opus-vega cross-referenced by PR #420
- 2026-10-02T09:37:59Z @neo-opus-vega cross-referenced by #19359
- 2026-10-02T09:41:20Z @neo-opus-vega cross-referenced by PR #19360
- 2026-10-02T10:39:28Z @neo-gpt-emmy cross-referenced by PR #433
- 2026-10-02T10:43:56Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-02T10:48:39Z @neo-opus-vega marked this issue as being blocked by #430
- 2026-10-02T10:54:31Z @neo-gpt-emmy referenced in commit `73dafb6` - "refactor(agentos): the mailbox grid rides the engine's scrollEdge, the interim body goes (#429)

With the engine pin carrying neomjs/neo#19357, Neo.grid.Body fires scrollEdge itself; the interim subclass fired it a second time, and the mailbox's once-per-count arm went red on the pin PR. The grid relays the engine's event unchanged; the subclass, its body config and its FAMILIES row are gone."
- 2026-10-02T10:57:53Z @neo-opus-vega referenced in commit `0c4946f` - "refactor(agentos): the memories pane requests at the engine's scroll edge, its drains go (#429)

Both memories registers relay grid.Body's scrollEdge through RowsGrid; the pane asks for the next window once per edge entry at the rendered depth while the producer's total says more exists and no window is in flight, the summary register quiet while a drill owns the zone. The four drain fields and both drain blocks are gone; the rendered key is written before the bags seat because a corpus shorter than one window announces its edge inside that set. Unit arms red-first against the drain (5), 1177/1177 at the pinned engine."
- 2026-10-02T11:02:33Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-02T11:35:21Z @tobiu referenced in commit `f2dd081` - "feat(deps): carry the first-run backend and grid scroll edge (#430) (#433)

* feat(deps): carry first-run backend and grid scroll edge (#430)

* fix(harness): wait for the empty-answer header before first paint (#430)

* refactor(agentos): the mailbox grid rides the engine's scrollEdge, the interim body goes (#429)

With the engine pin carrying neomjs/neo#19357, Neo.grid.Body fires scrollEdge itself; the interim subclass fired it a second time, and the mailbox's once-per-count arm went red on the pin PR. The grid relays the engine's event unchanged; the subclass, its body config and its FAMILIES row are gone.

---------

Co-authored-by: Neo Opus Vega <neo-opus-vega@neomjs.com>"
- 2026-10-02T11:37:42Z @neo-opus-vega referenced in commit `53a9919` - "refactor(agentos): the memories pane requests at the engine's scroll edge, its drains go (#429)

Both memories registers relay grid.Body's scrollEdge through RowsGrid; the pane asks for the next window once per edge entry at the rendered depth while the producer's total says more exists and no window is in flight, the summary register quiet while a drill owns the zone. The four drain fields and both drain blocks are gone; the rendered key is written before the bags seat because a corpus shorter than one window announces its edge inside that set. Unit arms red-first against the drain (5), 1177/1177 at the pinned engine."
- 2026-10-02T11:37:44Z @neo-opus-vega cross-referenced by PR #434
- 2026-10-02T12:35:29Z @neo-opus-vega referenced in commit `d030633` - "fix(agentos): an edge the memories list announces behind an open drill is replayed once on return (#429)

A hidden register keeps its last geometry, so a summary continuation landing while a drill owns the zone announces its new count on that store set and the engine latches it; a plain refusal left the list short of its corpus with no edge left to reach. The pane remembers the refused edge and replays it once after the drill closes; nothing pages behind the drill. Sophie's source witness on 53a9919; two arms red against it."
- 2026-10-02T12:37:52Z @neo-opus-vega cross-referenced by #19361

