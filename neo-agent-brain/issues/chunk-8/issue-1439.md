---
id: 1439
title: Embed A2A messages through the existing drain
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-02T23:05:36Z'
updatedAt: '2026-11-03T00:58:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1439'
author: neo-opus-vega
commentsCount: 2
parentIssue: 35
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 1440 Backfill the message index from the graph''s MESSAGE nodes'
  - '[x] 1445 query_messages: semantic recall over messages, admitted exactly like get_message'
closedAt: '2026-11-02T23:58:26Z'
---
# Embed A2A messages through the existing drain

## Context

This is leaf A of #35, the data path that makes A2A messages searchable. The intake read and the three-leaf split are in [6590575731](https://github.com/neomjs/neo-agent-brain/issues/35#issuecomment-6590575731).

At `dev` `613df5d7` no content search over messages exists:
- `MailboxService.mjs` has no embedding.
- `list_messages` takes no text.
- There is no FTS5 index.

Leaf B (one admission module plus `query_messages`) needs vectors before it can exist. That makes this leaf the first.

## The Problem

Every accepted message is already durable and projected, but nothing embeds it. The embed drain knows one source, the memory WAL, and the vector store knows three collections (memories, summaries, temporal summaries). A message therefore never enters the one index that semantic recall reads.

## The Architectural Reality

- **Send.** `MailboxService.addMessage` appends the message to its own WAL (`appendWalMessage`, `MailboxService.mjs:3667`) and then projects it into the graph (`_projectMessageWalRecord`, `:3692–3700`). The WAL (`helpers/messageWalStore.mjs`) writes `message-wal-YYYY-MM-DD.jsonl` segments, keeps graph-projection markers beside them, and prunes reconciled segments (`appendWalMessage`, l.169–191).
- **The facts admission needs are already recorded at send.** `normalizeMailboxRoutingMarker` holds `sentBy`/`to` as `known` or `legacy-unknown`. `normalizeBroadcastCohortMarker` holds the broadcast cohort (`known` with a count, or `legacy-unknown`). The audience itself is the WAL record's send-time snapshot plus the `DELIVERED_TO` edges, which `getMessage` reads (`MailboxService.mjs:4886–4893`).
- **The drain.** `drainWalOnce` / `startDrainLoop({getCollection, getConfig, expectedDimension})` (`ai/daemons/embed/drainCycle.mjs:448`, `:601`) embed through `collection.add({ids, metadatas, documents})` with no client embeddings (l.267). They verify the stored vectors (l.299) and write embed markers. Their reader, `readPendingWalRecords` (`memoryWalStore.mjs:500`), is specific to the memory WAL.
- **Collections.** `AbstractVectorManager` (l.35/44/54) and `ChromaManager` (l.234/267/286) have one getter per collection, and `CollectionProxy` switches on `collectionType` (l.40). Export, import and truncate enumerate the collections explicitly (`DatabaseService.mjs:416`, `:555`, `:799`).
- **Retraction.** `delete_message` is a sender-only retraction that overwrites subject and body in place (the `deleteMessage` JSDoc). The old text must then leave the index too.

## The Fix

1. **A `messages` collection.** Add a getter beside the three existing ones, using the same embedding function and dimension as `memories`.
2. **Embed markers for the message WAL.** Write them to `message-wal-YYYY-MM-DD.embedded.jsonl` with a pending reader, mirroring the memory WAL's split: records and markers in separate files, one writer per file. A segment prunes only when both its graph and its embed markers are reconciled.
3. **A message source for the drain.** The drain embeds `subject + "\n\n" + body` with this metadata:
   - `from`, `to`, `kind` (`direct` | `broadcast`) and `threadId`;
   - the cohort disposition (`known` | `legacy-unknown`);
   - for a receipt-backed broadcast, its send-time recipients.

   A legacy row is written as legacy, never guessed. Recipients are one array key, `recipients`, filtered with `$contains`. This was measured on a throwaway 1.5.9 at 23:1xZ, 11-02, by writing a broadcast delivered to `@a`/`@b`, a broadcast delivered to `@c` and a direct message to `@a`, then filtering with `{$or: [{to: '@a'}, {recipients: {$contains: '@a'}}]}`. Exactly the first and third were admitted. One boolean key per recipient works too, but it grows one metadata key per identity.
4. **Retraction drops the vector.** `delete_message` deletes the message's row from `messages`. That covers both orders. A retraction can land before the drain reaches the message, and the WAL still holds the original text, so the drain must skip a retracted message rather than embed that text.
5. ~~Backfill~~ → split to #1440 (23:2xZ, 11-02). It runs where the graph is (the MC container; the host graph is the orphan, #61), reads the cohort from `DELIVERED_TO` edges rather than the WAL, and keeps its own cursor: a different runtime, and an independently valuable leaf.
6. **The index is derived, so it is not exported.** `truncate` takes `messages` opt-in, and `purgeTestCollections` reclaims `test-message-*`. Backups leave it out on purpose: 4096-d vectors would add about 16 KB per message to every daily bundle, for data #1440 rebuilds from the graph. (Backup health was re-read at 23:08Z: healthy, last 14:51Z, restorable.)

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Error | Docs / Schema | Verification Evidence |
|---|---|---|---|---|---|
| `messages` Chroma collection (new) | #35 Fix §1 | One row per accepted message: document = subject + body, metadata as in Fix 3 | An embed failure leaves the message readable but unsearchable; the drain retries it like a memory record | `collections.message{Prod,Test}` leaves + collection getter; opt-in `truncate`; not exported (Fix 6) | drain spec + config spec + post-merge plane check |
| `add_message` result (`ai/mcp/server/memory-core/openapi.yaml`) | #35 AC-1 | Gains the `visibility` block that `add_memory` returns (`semanticQueryable: false` and `state: 'embed-deferred'` until drained) | Unchanged when the drain is down: the send still succeeds on the WAL append | `openapi.yaml` response schema | MailboxService spec |
| `delete_message` (retraction) | `deleteMessage` JSDoc | Also deletes the message's vector | The retraction is never blocked by vector-store unavailability; the drain reconciles the delete on its next cycle | `openapi.yaml` description line | retraction spec |

Decision Record impact: `none`. The drain gains a second source within its existing design.

## Acceptance Criteria

- [ ] AC-1: The drain embeds each accepted message into `messages` with Fix 3's metadata. `add_message` latency is unchanged, because the embed stays off the send path. Its result reports `visibility` in `add_memory`'s vocabulary: `{semanticQueryable: false, state: 'embed-deferred'}`. At send time the embed has always not yet happened, so no drain depth is read on the send path.
- [ ] AC-2: A receipt-backed broadcast's metadata carries exactly its send-time recipients, and a legacy broadcast carries `legacy-unknown`. Covered by a spec over both.
- [ ] AC-3: A retracted message has no vector, whether the retraction lands before or after the drain embeds it (spec covers both orders).
- [ ] AC-4: `truncate` drops `messages` when asked (opt-in), the destructive-operation guard and the purge script protect `neo-agent-messages`, and `purgeTestCollections` matches `test-message-*`. The index stays out of exports by design (Fix 6).
- [ ] AC-5 (post-merge, plane): after the cut, the embed daemon's message loop logs drain cycles, and a message sent after the cut appears in `neo-agent-messages` with its metadata.

The backfill AC moved to #1440 when the ticket was split (11-02); the ACs above are renumbered to stay contiguous.

## Out of Scope

- Any read surface, and the admission module: leaf B.
- Archive filtering. Archive is a recipient-side view, not admission, so it belongs to leaf B's query.
- The memory-mining tool-order line: leaf C, in neo-agent-skills.

## Avoided Traps

- **Embedding inside `addMessage`.** The send stays a WAL append plus projection; the embed rides the drain (#35 AC-1).
- **Backfilling from the WAL.** Reconciled segments are pruned.
- **Storing admission as `to ∈ {me, AGENT:*}`.** That loses receipt-backed cohorts and would leak them through leaf B's `where`.

## Related

- Parent: #35.
- #157: the operator-authored sibling; a superseded-by-#35 recommendation is on it.
- Grace's sizing probe on #35 (08-25).

Not `good first issue`: verifying it needs the plane's drain and backup wiring, and the metadata it writes is the input to an admission filter.

*Sweep attestations:*
- *Live latest-open + closed sweep:* latest 20 open, plus a closed search ("message embedding OR embed messages OR mailbox semantic OR message collection"), at 2026-11-02T23:00Z and re-checked at 23:05Z. Closed hits read: #563 (list pagination) and #511 (REM GC vectors), neither equivalent.
- *A2A in-flight sweep:* `list_messages({status:'all', limit:30})` at 23:00Z; no claim on message embedding.
- *Memory Core sweep:* `query_raw_memories` on the problem's nouns found no prior attempt.
- *Own-assignment sweep:* the only overlap is the parent, #35.
- *Structure map (§1c):* run in Brain this session (exit 0). The owners are `ai/daemons/embed/` (drain), `ai/services/memory-core/helpers/` (WAL stores) and `managers/` (collections).

Ownership: self-assigned (Vega).

Origin Session ID: afcce2e4-5eb9-4fa6-8895-4730263a042d

Retrieval Hint: "A2A message embedding drain messages collection backfill admission metadata receipt-backed broadcast"


## Timeline

- 2026-11-02T23:05:36Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-02T23:05:37Z @neo-opus-vega added the `enhancement` label
- 2026-11-02T23:05:38Z @neo-opus-vega added the `ai` label
- 2026-11-02T23:05:38Z @neo-opus-vega added the `agent-os` label
- 2026-11-02T23:27:26Z @neo-opus-vega cross-referenced by #1440
- 2026-11-02T23:27:56Z @neo-opus-vega changed title from **Embed A2A messages through the existing drain, with a bounded backfill** to **Embed A2A messages through the existing drain**
- 2026-11-02T23:29:50Z @neo-opus-vega referenced in commit `8f14243` - "test(memory-core): cover the message collection in the purge and guard parity specs (#1439)"
- 2026-11-02T23:30:55Z @neo-opus-vega cross-referenced by PR #1443
- 2026-11-02T23:47:13Z @neo-opus-vega cross-referenced by #1445
- 2026-11-02T23:58:26Z @tobiu referenced in commit `bcad758` - "feat(memory-core): embed A2A messages through the existing drain (#1439) (#1443)

* feat(memory-core): embed A2A messages through the existing drain (#1439)

Every accepted message was durable and projected, but nothing embedded it, so semantic
recall stopped at the mailbox. The embed daemon now drains the message WAL into a new
`messages` collection through the same retry, verify and compensation logic as add_memory.

- messageWalStore: embed markers (`message-wal-K.embedded.jsonl`), `toMessageEmbedRecord`
  (subject + body; from/to/kind/sentAt/threadId and, for broadcasts, cohort + recipients
  from the record's send-time routing), a cached pending reader, and pruning that waits on
  both graph and embed markers.
- drainCycle: a store seam (MEMORY_WAL_STORE default, MESSAGE_WAL_STORE) instead of a
  parallel pipeline; the message store has no pruner because the WAL writer prunes.
- embed daemon: a second loop for the message WAL; no extra lock (the messages dir's lock
  belongs to the message daemon, whose graph markers this loop never writes).
- collections.message{Prod,Test} + formula, getters, proxy branches, guard and purge lists.
- MailboxService: add_message reports visibility (embed-deferred); a retraction marks the
  WAL record retracted before deleting the vector, the purge-tombstone order.
- DatabaseService: opt-in `messages` truncate; the derived index is not exported.
- CollectionProxy/StorageRouter comments: ticket refs rewritten as prose (archaeology guard).

* test(memory-core): cover the message collection in the purge and guard parity specs (#1439)"
- 2026-11-02T23:58:27Z @tobiu closed this issue
### @neo-opus-vega - 2026-11-03T00:03:09Z

**Post-merge validation (Vega): AC-5 FAILS on the plane. The defect is mine; the fix is in progress.**

The plane cut to `dev` `41d360ab` went through at 00:02Z: four services healthy, Chroma and ingress untouched. But the plane runs no embed daemon:
- Memory Core drains both WALs in-process: `NEO_MEMORY_WAL_IN_PROCESS_DRAIN=true` and `NEO_MESSAGE_WAL_IN_PROCESS_DRAIN=true` in the mc-server env, started at `ai/mcp/server/memory-core/Server.mjs:343` / `:379`.
- #1443 wired the message embed loop only into the standalone `ai/daemons/embed/daemon.mjs`. The shared loop host's own JSDoc names both hosts (`drainCycle.mjs:605`), and I checked one.
- So on this plane no message is embedded.

**The fix:** wherever the memory embed drain runs, the message embed loop runs beside it. That means the in-process branch in `Server.mjs` starts it too. A spec will cover the in-process host, and the cut is redone. AC-5 is re-observed after that cut.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-03T00:04:53Z @neo-opus-vega cross-referenced by #1448
- 2026-11-03T00:05:20Z @neo-opus-vega cross-referenced by PR #1449
### @neo-opus-vega - 2026-11-03T00:58:08Z

**Post-merge validation: AC-5 observed on the plane** at `cf17e6c1`, which includes #1449's fix. The cut finished at 00:53Z on 11-03, with all four services healthy at that revision.

- **Embed marker.** A message sent after the cut, `MESSAGE:9fecb8c5-b4aa-4493-948a-db3f8915ea2b` (00:54:40.664Z, a direct DM), has an embed marker in `message-wal-2026-11-03.embedded.jsonl` (`embeddedAt` 00:55:00.685Z, 20 s later).
- **Index row.** Its row is in `neo-agent-messages`:
  - metadata `{from: '@neo-opus-vega', to: '@neo-gpt-sophie', kind: 'direct', sentAt: 1793667280664}`;
  - the document is subject plus body;
  - the vector is 4096-d.
- **Send receipt.** `add_message` returned `visibility: {semanticQueryable: false, state: 'embed-deferred'}`.
- **Catch-up.** The drain is working through the retained WAL: 25,617 records back to 10-04. By 00:56:07Z it had embedded 480, at about 2.2 per second, so it needs roughly 3 h.
- **Memory saves keep pace.** Meanwhile all 35 of today's memory records carry embed markers.
- **Sizing.** Chroma's anonymous memory was 12.31 GiB of its 16 GiB cap. The retained WAL adds about 0.4 GiB of vectors.
- **History.** The graph holds 60,130 MESSAGE nodes. #1440's body now carries that measurement and a sizing step before its `--apply`.

The first observation, at `bcad7585`, failed because the plane hosts the drain in-process. #1448 fixed that.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-03T01:13:45Z @neo-opus-vega cross-referenced by PR #1450
- 2026-11-03T02:04:44Z @neo-opus-vega cross-referenced by PR #1451

