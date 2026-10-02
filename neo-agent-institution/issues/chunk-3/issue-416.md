---
id: 416
title: The mailbox pane's boot drain starves the Observatory's first read
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - performance
assignees:
  - neo-opus-vega
createdAt: '2026-10-02T08:31:24Z'
updatedAt: '2026-10-02T08:33:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/416'
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
blockedBy: []
blocking: []
---
# The mailbox pane's boot drain starves the Observatory's first read

## Context

ROADMAP row 3 (the Observatory) is `blocked` on the cold `get_graph_scene` read: #310 AC-6's first useful paint waits on it, and [D#19317 §7](https://github.com/neomjs/neo/discussions/19317) named it a defect of today's install, to be diagnosed on the read path before any byte is added to the scene. The #312 epic-resolution review lists it as the second untracked gap. This ticket is that diagnosis, and the cause is not on the read path. It is the cockpit's own mailbox pane.

Evidence, all from the plane's Memory Core server log and tool metrics (`mc-server-2026-10-02.log`, `get_memory_core_tool_metrics`), read 2026-10-02 08:10–08:30Z:

- 07:35:00.569Z: the installed Fleet Manager (viewer `@neo-opus-ada`) boots and issues its first `get_graph_scene`. In the same second its mailbox pane issues `list_messages {box: inbox, status: all, to: @neo-opus-ada, limit: 51, offset: 0}`.
- 07:35:00Z → 07:37:54Z: the pane requests the next window as soon as the previous one lands, 173 pages, offsets 0 → 8600, mean interval 1.01 s, no gap over 3 s. That is the pane's drain (`mailbox/Container.mjs` `applySnapshot`, merged as #40): "the honest end (hasMore: false) is the only stop".
- `get_graph_scene` is issued again at 07:36:16Z and 07:37:04Z (the client's 60 s deadline had passed each time). The metrics record three scene reads completing at 07:38:14–16Z in 11.0–11.9 s each, after the drain's last page. Over 48 h the tool's minimum is 2.2 s and its maximum 67 s.
- A `list_messages` page costs the Memory Core about one second regardless of offset (today's own-inbox probes at offset 4000: 0.9–1.5 s). The SQL part of a page is ~0.1 s (measured read-only inside the container: the newest-first join over 11,120 `DELIVERED_TO` rows sorts on a JSON field); the rest is per-row projection and receipts.
- The same healthchecks that take 0.2–0.6 s otherwise took 195 s, 155 s and 112 s during the drain. Everything on the server starved, not only the scene.

The 28 Sept observation on Institution `#307` (111.6 s and 65.0 s first reads, then 2.9–7.7 s) is the same shape: the reads that fail are the ones issued while the pane walks the inbox.

## The Problem

Every cold boot of the cockpit walks the operator's whole inbox through the Memory Core, one 50-row page per second, before the operator has scrolled anywhere. The inbox is 8,600 rows today and grows with every A2A message, so the boot cost is O(inbox) and already three minutes. During those minutes the single-process Memory Core serves little else, and the Observatory's first read, the read row 3's first-useful-paint check depends on, exceeds the client deadline and renders `graph-read-failed`.

The drain also stamps rows the operator never saw. The MCP adapter calls `MailboxService.listMessages` with `recordSeen: true` (`ai/mcp/server/memory-core/toolService.mjs:523`), which records a seen receipt for the bound identity on every surfaced row. The viewer is the subject here (the installed FM runs as `@neo-opus-ada`), so a boot marks all 8,600 rows as shown to that seat, and `mark_read({all: true})`'s "the unread messages you were actually shown" no longer means what it says for that seat.

## The Architectural Reality

- `apps/agentos/view/fleet/mailbox/Container.mjs:386-396`: `applySnapshot` fires `pageRequest {offset: page.offset + page.limit}` once per freshly landed snapshot while `snapshot.page.hasMore`. `OperatorContainer.onInboxPageRequest` relays it as `inboxPageRequest`; `cockpit/Controller.loadOperatorInbox({offset})` reads `bridge.fleetMailboxMirror({subjectAgentId, offset})`.
- Brain `ai/services/fleet/fleetMailboxMirrorAdapter.mjs`: one page per call, `limit + 1` probe, `page.hasMore`. The adapter is correct; the loop is the caller's.
- Brain `MailboxService.listMessages` (`ai/services/memory-core/MailboxService.mjs:3543-3653`): per page one `COUNT` over the match set, the page query, `_projectMailboxRow` per row, `attachRelatedPullRequestStates`, and with `recordSeen` the seen receipts. The Memory Core runs one Node process; a page is roughly one second of it.
- `apps/agentos/view/fleet/mailbox/Grid.mjs` is a `Neo.grid.Container` whose body pools rows (`bufferRowRange`) and fetches nothing. Its docblock and the memories grid's (`memories/RowsGrid.mjs:20`) both say the drain stands "until the engine lands its scroll-edge seam". The engine's `grid.Body` computes `visibleRows` and the mounted range in `updateMountedAndVisibleRows` and publishes only `isScrollingChange`; no scroll-edge event exists yet (neo issue filed alongside this one, linked below).
- The memories pane (`memories/Container.mjs:447-457`) drains the same way, per selected agent, not at boot. It is the sibling, not this ticket.

## The Fix

The pane loads one window at boot and the next window only when the operator reaches the loaded end.

1. `mailbox/Container.mjs`: `applySnapshot` no longer requests the next window on its own. It records `page.hasMore` and the next offset; the request moves to a `scrollEdge` handler that fires `pageRequest` when the grid's mounted range reaches the loaded end, at most one request in flight, and nothing after `hasMore: false`.
2. The grid gets the scroll-edge seam. Until the engine publishes it, the mailbox grid's body is a thin `Neo.grid.Body` subclass that, after `updateMountedAndVisibleRows`, fires `scrollEdge {startIndex, endIndex, count}` once per entry into the last `bufferRowRange` rows of the store. When the engine's event lands and the pin carries it, the subclass is deleted and the pane listens to the engine's event; the handler stays.
3. The honest end stays visible: the pane's existing state line says nothing new while rows show; the freshness chip and `page` bounds already name the loaded count. No paging chrome (operator direction 2026-08-28 stands).
4. The cockpit's boot order is unchanged. The scene read simply no longer queues behind 172 pages.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `pageRequest` from the mailbox pane | `mailbox/Container.mjs` (the pane); relayed unchanged by `OperatorContainer` → `cockpit/Controller.loadOperatorInbox` | Fired once per scroll-edge entry while the last snapshot's `page.hasMore` is true and no request is in flight; never from `applySnapshot` itself. Payload `{offset, source}` unchanged. | No scroll edge reached → one window stays loaded; `hasMore: false` → no request. | the pane's docblock | `container.spec.mjs`: fresh snapshot with `hasMore` fires nothing; scroll edge fires one; `hasMore: false` fires none; in-flight dedup |
| `scrollEdge` on the mailbox grid's body | the FM `grid.Body` subclass (interim), then the engine's event | Fires `{startIndex, endIndex, count}` when `endIndex + bufferRowRange >= store.count`, once per entry (re-armed when `count` grows). | A store with fewer rows than the window fires on first layout, so a short first page still reaches `hasMore`. | docblock | a body spec over `updateMountedAndVisibleRows` with a fake store count |
| Seen receipts for the viewer | Brain `toolService.mjs:523` (unchanged) | Only rows the operator scrolled to are surfaced, so only those are stamped. | — | — | follows from the first row; no Brain change |

## Acceptance Criteria

- [ ] A freshly landed first window with `page.hasMore: true` fires no `pageRequest` (red-first against the current pane).
- [ ] A scroll-edge entry fires exactly one `pageRequest` at `offset + limit`; a second edge entry while that request is in flight fires none; `hasMore: false` fires none.
- [ ] With a stub bridge, a cockpit boot records exactly one `fleetMailboxMirror` read before the first scene snapshot is applied.
- [ ] Row 51+ is still reachable by scrolling, through the same `applyBags` append path; the thread-collapse state of held rows survives an append (existing arms stay green).
- [ ] *(L3, this seat's plane, on the PR)* A cold launch of a checkout FM against the team plane: the Memory Core log shows one `list_messages` page for the viewer in the first minute, and the first `get_graph_scene` completes under the client deadline.
- [ ] *(L4, installed candidate at the operator's viewer; the receipt is row 3's own check, #312)* The same read from the installed FM. Not claimed by this PR.

## Out of Scope

- The memories pane's drain (sibling; same seam, separate leaf once the engine event exists).
- The Memory Core's per-page cost (~1 s per 50 rows, SQL ~0.1 s of it). A Brain defect-note is on the fold; a Brain ticket follows a profile, not this leaf.
- The engine's scroll-edge event itself (neo ticket below).
- Any change to the client timeout or the scene payload (D#19317 §7: not before the cause is named; it now is).

## Avoided Traps

- **Raising the client's 60 s deadline.** The reads would still queue behind three minutes of pages; the failure would move, not go.
- **Throttling the drain.** It still costs the Memory Core 173 pages per boot and still stamps 8,600 unseen rows.
- **Paging chrome.** Rejected on 2026-08-28; the scroll edge is the honest replacement for both the chrome and the drain.
- **Deferring the drain until after the scene lands.** The scene would paint, and the next three minutes of every other read would still starve.

## Related

#312 (the Observatory epic; this is its second untracked gap) · #310 AC-6 (first useful paint) · [D#19317 §7](https://github.com/neomjs/neo/discussions/19317) (the plane-crossing hypothesis now has its cause) · #40 (the drain's origin) · `#307` (the 28 Sept observation) · neomjs/neo-agent-brain `MailboxService.listMessages` and `fleetMailboxMirrorAdapter.mjs` · the engine scroll-edge ticket: neomjs/neo#19356

Live latest-open sweep: the 21 open Institution issues read via `list_issues(sort: created)` at 08:28Z and again at 08:33Z, the A2A inbox's last 30 messages at 08:26Z and 12 at 08:33Z, and one `query_raw_memories` on the symptom: no equivalent ticket, claim or prior decision. Own open assignments: #411 only.

Origin Session ID: 60d9be31-4233-40fd-9b07-6ee6a9ebf6cf
Retrieval Hint: "mailbox pane boot drain 173 pages get_graph_scene starved 60 s deadline scroll edge"


## Timeline

- 2026-10-02T08:31:24Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-02T08:31:26Z @neo-opus-vega added the `bug` label
- 2026-10-02T08:31:26Z @neo-opus-vega added the `agent-os` label
- 2026-10-02T08:31:26Z @neo-opus-vega added the `ai` label
- 2026-10-02T08:31:26Z @neo-opus-vega added the `performance` label
- 2026-10-02T08:32:22Z @neo-opus-vega cross-referenced by #19356

