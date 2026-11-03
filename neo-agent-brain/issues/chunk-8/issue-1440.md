---
id: 1440
title: Backfill the message index from the graph's MESSAGE nodes
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-02T23:27:25Z'
updatedAt: '2026-11-03T09:39:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1440'
author: neo-opus-vega
commentsCount: 3
parentIssue: 35
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1439 Embed A2A messages through the existing drain'
blocking: []
closedAt: '2026-11-03T04:48:24Z'
---
# Backfill the message index from the graph's MESSAGE nodes

## Context

This is the history half of #35's data path, split out of #1439. #1439 embeds every message accepted from now on: its drain reads the message WAL, keyed by the WAL's embed markers. Everything older than the WAL's retention window was pruned from the WAL long ago, and only the graph still holds it. On 11-03 the graph held 60,130 MESSAGE nodes, and the WAL retained 25,617 records back to 10-04, which #1439's drain embeds on its own. Without this leaf, semantic recall would start blind to every coordination fact older than the cut.

It is a separate leaf, not part of #1439, because it runs differently:
- **Where it runs.** It must run where the graph is, inside the Memory Core container. The host graph is the orphan (#61).
- **Where admission comes from.** It reads each message's admission facts from graph edges rather than from a WAL record's send-time routing.
- **What it needs of its own.** A receipt, and a sizing read before it writes.

## The Architectural Reality

- **The record shape and metadata are fixed by #1439.** `toMessageEmbedRecord` (`ai/services/memory-core/helpers/messageWalStore.mjs`) maps a message to `{id, document, metadata}`. The metadata is: `from`, `to`, `kind`, `sentAt`, `threadId`, and for broadcasts `cohort` (`known` | `legacy-unknown`) plus `recipients` (the array key filtered with `$contains`).
- **The graph carries the same facts as edges.** `SENT_BY` and `SENT_TO` give the route. A receipt-backed broadcast's audience is its `DELIVERED_TO` edges; a legacy broadcast has none. That is the distinction `getMessage` draws (`MailboxService.mjs`, `getBroadcastDeliveryEdge` / `hasBroadcastDeliveryEdges`). Retracted messages carry `properties.retracted: true` and placeholder text.
- **Delivery edges cannot tell a legacy broadcast from a known-zero one.** Both have no delivery edges. The cohort is stamped on the `SENT_TO → AGENT:*` edge (`toBroadcastCohortEdgeProperties(getMessageWalBroadcastCohort(record))`, `MailboxService.mjs:3855–3858`), so the graph mapper reads it there. The node's `sentAt` is an ISO string where the WAL mapper writes epoch ms, so the graph mapper normalizes it. Both points come from Maryam's #1443 review.
- **The target is `aiConfig.collections.message`** (`neo-agent-messages`), through `StorageRouter.getMessageCollection()`.
- **Sizing (measured 11-03 00:56Z).** At 4096-d a message costs 16 KiB of vector, so the full corpus is about 0.92 GiB once indexed. Before any message was embedded, Chroma's anonymous memory was 12.31 GiB of its 16 GiB cap. Skipping the ids the WAL drain already wrote keeps this run to about 34.5k messages, roughly 0.53 GiB.

## The Fix

1. **A shared mapping core.** Factor the metadata build in `toMessageEmbedRecord` into one function that takes the route, the cohort and the properties. The WAL mapper and the graph mapper both call it, so the two sources cannot disagree on what admission reads.
2. **The backfill script.** `ai/scripts/maintenance/backfillMessageEmbeddings.mjs` sits beside `defragChromaDB.mjs` and `purgeTestCollections.mjs`; the structural pre-flight is a sibling match. It:
   - walks `MESSAGE` nodes in `sentAt` order and skips retracted ones;
   - skips ids the collection already holds (one `get` per batch), so it embeds only what the WAL drain never saw;
   - keeps no cursor file: the collection is the cursor, so an interrupted run resumes by skipping what it already wrote;
   - upserts bounded batches, so a re-run is idempotent;
   - deletes again a message retracted while its batch was written, as the drain compensates a mid-embed retraction;
   - prints a receipt `{embedded, pending, skipped, failed}`, plus `projectedBytes` on a dry run;
   - is a dry run unless `--apply` is passed, and never runs on boot.
3. **Restore runbook.** A graph restore is followed by a re-run of this script. The message index is derived and is not exported, per #1439's truncate JSDoc.
4. **Retraction lookup.** `_retractMessageFromIndex` finds the record with `readPendingMessageWalEmbedRecords({dir, ids: [messageId], limit: 1})` instead of the graph-marker index. It then marks only a record still pending embed, since the vector delete covers an embedded one, and it no longer depends on projection markers. This also comes from Maryam's #1443 review.
5. ~~**Drain-loop hardening** (Julia's #1449 review).~~ Dropped at intake on 11-03, per ADR 0019 B3: "the SSOT guarantees the tree; let it fail loud". An overlay is a thin child over the canonical template with deep-merge inheritance, so no deployment can lose the `memoryWal` or `messageWal` slice. A guard would defend against a state the config makes impossible.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Error | Docs / Schema | Verification Evidence |
|---|---|---|---|---|---|
| `backfillMessageEmbeddings.mjs` (new maintenance script) | #35 AC-3 | Runs a bounded, resumable, idempotent upsert of the graph's MESSAGE nodes into `collections.message`, printing a receipt | A failed batch is counted; a re-run retries it, since the collection still lacks it | the script's JSDoc + the restore runbook line | spec over a seeded graph + fake collection; post-merge plane receipt |

Decision Record impact: `none`.

## Acceptance Criteria

- [ ] AC-1: The backfill walks MESSAGE nodes in send order in bounded batches. A re-run is idempotent: ids the collection holds are skipped, so an interrupted run resumes without a cursor file. Covered by a spec.
- [ ] AC-2: For the same message, the graph mapper and #1439's WAL mapper produce identical metadata: direct, receipt-backed broadcast, known-zero broadcast and legacy broadcast. The cohort is read from the `SENT_TO` edge and `sentAt` is normalized to epoch ms. Covered by a parity spec.
- [ ] AC-3: Retracted messages, and ids the collection already holds, are skipped and counted; neither is re-embedded. A message retracted while its batch is written is deleted again.
- [ ] AC-4: It is a dry run by default and writes only with `--apply`. It prints the receipt `{embedded, pending, skipped, failed}` and never runs on boot. The dry run's receipt adds `projectedBytes`.
- [ ] AC-5 (post-merge, plane): before `--apply`, record Chroma's anonymous memory plus the dry run's projection against its cap, the input #1416's sizing rule reads. Then run inside the Memory Core container. The receipt's embedded plus skipped count equals the graph's MESSAGE count.
- [ ] AC-6: A retraction appends its marker only for a record still pending embed, located without the graph-marker index. Covered by a spec.
- ~~AC-7: A `getConfig()` that throws, or returns no slice, is absorbed as a failed cycle, and the loop keeps rescheduling.~~ Dropped with Fix 5 (ADR 0019 B3).

## Out of Scope

- The live data path for new messages: #1439.
- The read surface: #35's leaf B.

## Avoided Traps

- **Backfilling from the WAL.** Its reconciled segments are pruned, so it is not the history.
- **A host-side run.** The host graph is the orphan (#61); run where the serving graph is.
- **Exporting the derived index into backups.** 4096-d vectors cost about 16 KB per message in every bundle, for data this script rebuilds.

## Related

- Parent: #35. Sibling: #1439 (the data path).
- #61: the orphan host graph.

Not `good first issue`: it runs against the plane's serving graph, and the metadata it writes feeds an admission filter.

*Sweep attestations:*
- *Live latest-open + closed sweep:* latest 6 open since the 23:05Z full 20, plus a closed and open search ("backfill message OR message index OR message embeddings"), at 2026-11-02T23:26Z. Hits: #1439 (the split source), #35 (parent), #80 (vector-rebuild failure receipts, a different store and failure); no equivalent.
- *A2A in-flight sweep:* the last 12 messages, all statuses, at 23:26Z; no claim on a message backfill.
- *Memory Core sweep:* the #1439 sweep at 23:00Z covers the same nouns; no prior attempt.
- *Own-assignment sweep:* the only overlap is #1439.
- *Structure map (§1c):* run this session (exit 0); the owner is `ai/scripts/maintenance/` (sibling match).

Ownership: self-assigned (Vega).

Origin Session ID: afcce2e4-5eb9-4fa6-8895-4730263a042d

Retrieval Hint: "backfill message index graph MESSAGE nodes cursor receipt DELIVERED_TO cohort parity"



## Timeline

- 2026-11-02T23:27:25Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-02T23:27:27Z @neo-opus-vega added the `enhancement` label
- 2026-11-02T23:27:27Z @neo-opus-vega added the `ai` label
- 2026-11-02T23:27:27Z @neo-opus-vega added the `agent-os` label
- 2026-11-02T23:27:34Z @neo-opus-vega added parent issue #35
- 2026-11-02T23:27:57Z @neo-opus-vega cross-referenced by #1439
- 2026-11-02T23:30:55Z @neo-opus-vega cross-referenced by PR #1443
- 2026-11-02T23:47:13Z @neo-opus-vega cross-referenced by #1445
- 2026-11-02T23:47:21Z @neo-opus-vega marked this issue as being blocked by #1439
- 2026-11-03T00:42:46Z @neo-opus-vega cross-referenced by PR #1449
- 2026-11-03T01:13:45Z @neo-opus-vega cross-referenced by PR #1450
- 2026-11-03T01:53:07Z @neo-opus-vega referenced in commit `85b0b69` - "fix(memory-core): only a broadcast can be open to everyone in the admission filter (#1440)

admissionWhere's legacy branch now requires kind broadcast, like its known-zero branch, so a direct row can never be admitted by a cohort tag; queryMessages' JSDoc states that recall never repairs a projection gap. Both from Maryam's review of PR #1450."
- 2026-11-03T02:04:41Z @neo-opus-vega referenced in commit `8ce9945` - "feat(memory-core): backfill the message index from the graph (#1440)

backfillMessageEmbeddings walks the graph's MESSAGE nodes in send order and embeds each one the collection lacks, with the metadata the WAL drain writes: both mappers now share toMessageEmbedMetadata. The collection is the cursor, so a re-run resumes without a cursor file; a dry run projects the vector bytes before any write. A retraction marks only a record still pending embed, found without the graph-marker index."
- 2026-11-03T02:04:42Z @neo-opus-vega referenced in commit `45c6ec5` - "fix(memory-core): only a broadcast can be open to everyone in the admission filter (#1440)

admissionWhere's legacy branch now requires kind broadcast, like its known-zero branch, so a direct row can never be admitted by a cohort tag; queryMessages' JSDoc states that recall never repairs a projection gap. Both from Maryam's review of PR #1450."
- 2026-11-03T02:04:44Z @neo-opus-vega cross-referenced by PR #1451
- 2026-11-03T04:48:24Z @tobiu referenced in commit `cea6adc` - "feat(memory-core): backfill the message index from the graph (#1440) (#1451)

* feat(memory-core): backfill the message index from the graph (#1440)

backfillMessageEmbeddings walks the graph's MESSAGE nodes in send order and embeds each one the collection lacks, with the metadata the WAL drain writes: both mappers now share toMessageEmbedMetadata. The collection is the cursor, so a re-run resumes without a cursor file; a dry run projects the vector bytes before any write. A retraction marks only a record still pending embed, found without the graph-marker index.

* fix(memory-core): only a broadcast can be open to everyone in the admission filter (#1440)

admissionWhere's legacy branch now requires kind broadcast, like its known-zero branch, so a direct row can never be admitted by a cohort tag; queryMessages' JSDoc states that recall never repairs a projection gap. Both from Maryam's review of PR #1450."
- 2026-11-03T04:48:25Z @tobiu closed this issue
### @neo-opus-vega - 2026-11-03T04:53:37Z

**AC-5, post-merge on the plane** at `15dbfb38`. The cut finished at 04:52Z on 11-03 with all four services healthy at that revision.

**Dry run, inside the Memory Core container:**

| messages | already indexed | pending | retracted | incomplete | projectedBytes |
|---|---|---|---|---|---|
| 60,313 | 25,790 | 27,305 | 28 | 7,190 | 447,365,120 (≈0.42 GiB) |

**Sizing against the cap.** Chroma's cgroup holds 12.86 GiB anonymous against its 16 GiB `memory.max`. After the apply that is about 13.28 GiB, about 2.7 GiB under the cap. `--apply` started at 04:53Z in the background and logs to `/app/.neo-ai-data/backfill-1440.log`. The final receipt follows here.

**The 7,190 incomplete messages, classified:**
- **3,379 aren't messages.** They are graph entity nodes with a `MESSAGE:` id prefix (`logical_layer`, `gravity_well`, `strategic_weight` …), with no route edges. Skipping them is correct.
- **3,810 are real messages from June–July 2026 without a `SENT_BY` edge.** They have a subject and body, a `SENT_TO` edge, and `from`/`to` on the node. They stay out of the index: `getMessage` itself treats them as unrouted, so indexing them would break admission parity. The designed fix is to restore their sender carrier through the graph integrity repair and then re-run the backfill. The collection is the cursor, so the re-run embeds only them. Tracked on #35.
- 1 message lacks its `SENT_TO` edge.

AC-5's identity "embedded + skipped = the graph's MESSAGE count" holds at dry-run time: 0 + (25,790 + 28 + 7,190) + 27,305 pending = 60,313.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-03T04:53:40Z @neo-opus-vega cross-referenced by #35
### @neo-opus-vega - 2026-11-03T08:07:11Z

**#1440 backfill `--apply`: final receipt** (local plane, MC container, 04:53Z to about 07:5xZ)

```json
{"mode": "apply", "messages": 60313, "embedded": 27193, "pending": 27305,
 "skipped": {"present": 25790, "retracted": 28, "incomplete": 7190}, "failed": 112}
```

- **Every pending message was attempted:** embedded plus failed equals pending (27,193 + 112 = 27,305).
- **The 112 failures were transient.** Four batches failed at 06:52Z: first `socket hang up`, then `ECONNREFUSED` on the host's LM Studio (`:1234`) while it was briefly down. It answered 200 again by 07:18Z.
- **A re-run started at 08:07Z** (pid 4974, log `backfill-1440-rerun.log`) to pick up those 112. It is idempotent, because messages already present are skipped. Its receipt follows here.
- **Chroma after the run:** see #1416.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-11-03T08:07:13Z @neo-opus-vega cross-referenced by #1416
### @neo-opus-vega - 2026-11-03T09:39:24Z

**#1440 re-runs: all 27,305 pending messages are now embedded** (follow-up to 6598549839)

| Run | Pending | Embedded | Failed |
|---|---|---|---|
| `--apply` (04:53Z) | 27,305 | 27,193 | 112 |
| re-run 1 (08:07Z) | 112 | 80 | 32 (`socket hang up`) |
| re-run 2 (08:54Z) | 32 | 32 | 0 |

- **The whole set is in:** 27,193 + 80 + 32 = 27,305, the original pending set. The scan grew from 60,313 to 60,573 messages overnight, yet pending only shrank, so new messages are embedded at write time by the live drain.
- **A failed batch is counted and left for a rerun, as #1440's contract defines.** An automatic retry is an improvement, not a fix. My defect-note `5ff18932` overstated it as a defect, and the error string alone doesn't name a cause.
- **Any retry must first cover the arm Emmy controlled on #35** (6599788553): an applied write whose response is lost while the message is retracted. That follow-up is mine. Until it lands, the rerun stays the recovery path.
- **Chroma after the backfill:** see #1416 (6598550180).

— Vega (Claude Opus 5.5, Claude Code) 🌿



