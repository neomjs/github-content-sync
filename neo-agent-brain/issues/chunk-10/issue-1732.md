---
id: 1732
title: 'June–July messages lack routing edges, so even their sender is refused'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-15T00:59:51Z'
updatedAt: '2026-11-15T13:07:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1732'
author: neo-opus-vega
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
blocking:
  - '[x] 1733 Run the legacy-carrier rebuild and the message backfill on the plane'
closedAt: '2026-11-15T03:04:53Z'
---
# June–July messages lack routing edges, so even their sender is refused

## Context

Found while closing #35. #1725's plane receipt ([comment 6886860038](https://github.com/neomjs/neo-agent-brain/issues/1725#issuecomment-6886860038)) counted 7,190 `incomplete` messages that the embed backfill skips. I then ran a read-only census inside the Memory Core container on plane `2cb8e911` (2026-11-15; aggregate counts only, no content):

- **By send month:** 2026-06: 1,707 of 1,707 incomplete; 2026-07: 5,466 of 5,652; 2026-08 through 2026-11: 0 of 64,272. Every message since August is complete.
- **By missing piece:** no `SENT_BY`: 3,810; no `SENT_BY` and no `SENT_TO`: 3,362; no `SENT_TO`: 1; neither edge nor `sentAt`: 17.
- **What the nodes still carry:** 7,173 of the 7,190 have `from`, `to` and `sentAt` as node properties. The other 17 carry only `name`, `description` and concept fields (`semanticVectorId`, `logical_layer`, `gravity_well`, …) under a `MESSAGE:` id. They are not messages.

## The Problem

A message without `SENT_BY` / `SENT_TO` is closed to every mailbox read, its sender and recipient included:

- **`get_message` refuses.** `MESSAGE:3303c7aa-a12e-4440-8f42-88c7e77e0ffa` is a direct message @neo-opus-vega sent on 2026-06-13. Asked by @neo-opus-vega, it answers "Unauthorized: message … was not sent to or from @neo-opus-vega".
- **Outboxes miss them.** @neo-opus-vega's outbox listing begins at 2026-07-31, though the seat has sent messages since June.
- **`query_messages` cannot recall them.** The backfill skips incompletely routed messages by design.

So the swarm's first two months of coordination are in the graph, and no reader can open them.

## The Architectural Reality

- `MailboxService.getMessage` (`ai/services/memory-core/MailboxService.mjs`, around `:5096`) takes the routing facts from edges only (`getMessageRouteFacts`, `:2786`). It admits through `canReadMessage` (`helpers/messageAdmission.mjs:33`): the sender, direct recipients, broadcast deliveries and inbox grants. With no edges, `sentBy` is `null` and `sentTo` is `[]`, so every caller is refused.
- The three carriers belong to `repairMessageGraphIntegrity` (`:4215`), which rebuilds projections from accepted message-WAL records. The WAL keeps only its retention window (`backfillMessageEmbeddings.mjs`: "The embed drain indexes messages from the message WAL, which keeps only its retention window; the graph keeps every message"), so June and July are out of its reach. `getMessage`'s on-read repair calls it and finds nothing.
- `ai/scripts/maintenance/rebuildMessageEdges.mjs` rebuilds edges from the node's own fields, but only `IN_REPLY_TO`, `PART_OF_THREAD`, `REFERENCES_TICKET` and `TAGGED_CONCEPT`.

Design authority: `rebuildMessageEdges.mjs`'s JSDoc, "`repairMessageGraphIntegrity` repairs the three carriers only". Routing is the WAL repair's job, and nothing covers a message older than the WAL.

## The Fix

Extend `rebuildMessageEdges.mjs` with the two carriers a node can still prove: `from` → `SENT_BY` and `to` → `SENT_TO`.
- Write them with the edge properties and target resolution the `MailboxService` projection gives them, former logins included.
- Keep the script's dry-run default, `--types` selection and `INSERT … WHERE NOT EXISTS` identity.
- Skip and count a target node that does not exist; never create one.
- A route decides who may read the message, so it is linked only from a node labelled `MESSAGE`, from a single string, to a target the projection routes to (senders `AgentIdentity` or `System`; destinations `AgentIdentity` or `BroadcastSentinel`), and only while the message holds no route of that type, which the insert itself checks. Anything else counts as `invalid` (added after review 5908454433 on PR #1734).
- Leave `DELIVERED_TO` alone. The send-time recipient snapshot is not on the node, so a rebuilt broadcast without deliveries reads as `legacy-unknown`, which `canReadMessage` already opens to every reader (the existing rule for legacy broadcasts).

On the plane after merge: rebuild dry run → `--apply`. Then the embed backfill's dry run reports the new `pending` and `projectedBytes`, and its `--apply` follows the Chroma sizing read.

Decision Record impact: none.

## Acceptance Criteria

- [x] A unit arm, red on `dev` first: a `MESSAGE` node with `from` and `to` and no carriers gains `SENT_BY` and `SENT_TO` equal to what the projection writes for the same message.
- [x] A unit arm: a present carrier is left alone, a missing target is skipped and counted, and a dry run writes nothing.
- [x] A unit arm: after the rebuild, `canReadMessage` admits the sender and the recipient of a rebuilt direct message and refuses a third identity that holds no inbox grant.
- [x] The module JSDoc names the carriers it rebuilds and why the WAL repair cannot.
- [x] (post-merge, owned by #1733; receipts [rebuild](https://github.com/neomjs/neo-agent-brain/issues/1733#issuecomment-6890194465) and [backfill](https://github.com/neomjs/neo-agent-brain/issues/1733#issuecomment-6894773858)) On the plane: the rebuild's dry-run and apply receipts; `get_message("MESSAGE:3303c7aa-a12e-4440-8f42-88c7e77e0ffa")` succeeding for @neo-opus-vega; then the embed backfill's dry run, the sizing read and its `--apply` receipt.

ACs 1–4: delivered by PR #1734, merged on 2026-11-15 as `2d07503d`; its AC Evidence table names each spec arm. AC-5 stays with #1733.

## Out of Scope

- The 17 `MESSAGE:`-keyed nodes that are not messages (a separate defect-note).
- `DELIVERED_TO` snapshots for legacy broadcasts.
- Any change to admission.

## Avoided Traps

- **Admitting by node properties in `getMessage`** (a fallback read of `from` / `to`). That would create two admission sources, and `query_messages`' `admissionWhere` would need the same fallback. Edges stay the one source.
- **Creating identity nodes** for `from` / `to` values that resolve to nothing.

## Related

#35 (closed: messages recallable by meaning) · #1440 (the backfill) · #1725 (its compensation; the receipt that surfaced this).

Live latest-open + closed sweep: checked the latest 20 open plus closed searches (`message routing edges SENT_BY legacy`; `get_message Unauthorized legacy message`) at 2026-11-15T01:0xZ; the hits #1440 and #563 govern other scope; no equivalent or prior exclusion. Backlog: 154 open (red); no open owning ticket, since #35 closed tonight.
A2A in-flight claim sweep: the latest 30 messages, all states, up to 00:58Z; no claim on message routing.
MC sweep: two `query_raw_memories` queries on the symptom (messages missing routing edges; get_message refusing an old message's sender); no prior decision.
Own-assignment sweep: 12 open; none on this surface.
Structure map: `ai/scripts/maintenance` owns the rebuild (sibling `backfillMessageEmbeddings.mjs`).
Not `good first issue` / `help wanted`: the receipts need the live plane, and the change decides who can read whose messages.

Origin Session ID: 4faded79-9c19-44ab-83d6-ece47e8a2768
Retrieval Hint: "June-July messages lack SENT_BY SENT_TO routing edges; get_message refuses the sender of an old message"

🌿 **Vega** · `@neo-opus-vega` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-15T00:59:51Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-15T00:59:52Z @neo-opus-vega added the `bug` label
- 2026-11-15T00:59:53Z @neo-opus-vega added the `ai` label
- 2026-11-15T00:59:53Z @neo-opus-vega added the `agent-os` label
- 2026-11-15T01:07:00Z @neo-opus-vega cross-referenced by #1733
- 2026-11-15T01:07:10Z @neo-opus-vega marked this issue as blocking #1733
- 2026-11-15T01:08:25Z @neo-opus-vega cross-referenced by PR #1734
- 2026-11-15T01:43:12Z @neo-opus-vega referenced in commit `de24d97` - "fix(memory-core): a rebuilt route is atomic and links only routable parties (#1732)

A route decides who may read a message, and two inputs widened the audience:
- The insert guarded (source, target, type), so a different recipient that
  another writer linked after the precheck got a second route. The route
  insert now guards (source, type) itself, and a lost race counts as present.
- An array-valued `to` led by `AGENT:*` became a legacy broadcast open to every
  reader. A route now needs a node labelled MESSAGE, a single string, and a
  target of a kind the projection routes to (AgentIdentity or System senders,
  AgentIdentity or BroadcastSentinel destinations). Anything else counts as
  `invalid`."
### @neo-opus-vega - 2026-11-15T02:28:24Z

**Handover (session sunset, 2026-11-15 02:2xZ; owner Vega, next session).**

- **PR #1734 at `de24d970`:** Emmy's RC 5908454433 is addressed (response 6887476436): a route insert now holds one route per message and type atomically, and only a genuine `MESSAGE` with a scalar identity of a routable kind gets one. The rebuild spec passes 16/16 with both controls, which fail on `fe1d54c4` for their behavior. Re-requested at green, 01:51Z. Next: Emmy's Round 2.
- **After the merge and a plane cut carrying it:** #1733's run, inside `neo-local-agent-os-mc-server-1`:
  1. the rebuild's dry run, then apply;
  2. `get_message("MESSAGE:3303c7aa-a12e-4440-8f42-88c7e77e0ffa")` as `@neo-opus-vega`;
  3. the backfill's dry run;
  4. the Chroma sizing read;
  5. the backfill's apply;
  6. receipts on #1733.
- **Census (read-only, aggregate):** 7,190 incomplete messages, all from June and July 2026. 7,173 still carry `from` / `to`; the other 17 are `ARTIFACT_TASK` / `ARTIFACT_PLAN` / `MEMORY` nodes (defect-note only).

— Vega (Opus 5.5, Claude Code) 🌿

- 2026-11-15T03:04:53Z @tobiu referenced in commit `2d07503` - "feat(memory-core): the edge rebuild links a pre-WAL message's sender and destination (#1732) (#1734)

* fix(memory-core): the edge rebuild links a pre-WAL message's sender and destination (#1732)

7,173 messages from June and July keep `from` and `to` on the node but have no
SENT_BY / SENT_TO edge. The WAL repair cannot rebuild them because they are older
than the WAL, so get_message refused even their sender, outboxes skipped them, and
the embed backfill could not index them.

rebuildMessageEdges now links the two carriers from the node's own fields. It
normalizes the identity as the projection does, links at most one carrier of each
type, and stamps a broadcast route `broadcastCohort: 'legacy-unknown'`, the
projection's stamp when no recipient snapshot exists. DELIVERED_TO stays
unrebuilt: the snapshot is not on the node.

* fix(memory-core): a rebuilt route is atomic and links only routable parties (#1732)

A route decides who may read a message, and two inputs widened the audience:
- The insert guarded (source, target, type), so a different recipient that
  another writer linked after the precheck got a second route. The route
  insert now guards (source, type) itself, and a lost race counts as present.
- An array-valued `to` led by `AGENT:*` became a legacy broadcast open to every
  reader. A route now needs a node labelled MESSAGE, a single string, and a
  target of a kind the projection routes to (AgentIdentity or System senders,
  AgentIdentity or BroadcastSentinel destinations). Anything else counts as
  `invalid`."
- 2026-11-15T03:04:54Z @tobiu closed this issue
- 2026-11-15T06:31:10Z @neo-opus-vega cross-referenced by #1753

