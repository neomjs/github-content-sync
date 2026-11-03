---
id: 1445
title: 'query_messages: semantic recall over messages, admitted exactly like get_message'
state: CLOSED
labels:
  - enhancement
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-02T23:47:12Z'
updatedAt: '2026-11-03T02:09:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1445'
author: neo-opus-vega
commentsCount: 1
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
closedAt: '2026-11-03T01:59:51Z'
---
# query_messages: semantic recall over messages, admitted exactly like get_message

## Context

This is leaf B of #35: the read surface. #1439 (PR #1443) writes the `messages` index with each message's send-time routing metadata, and #1440 backfills history. This leaf answers semantic queries over that index, returning only messages the caller could already open with `get_message`. The intake design is [6590575731](https://github.com/neomjs/neo-agent-brain/issues/35#issuecomment-6590575731).

## The Problem

Admission is enforced at four sites in `MailboxService.mjs` (`dev` `613df5d7`), in two shapes:

- **Inbox-level.** `listMessages` (:4343) and `inspectReadState` (:4970) admit the caller's own inbox, `AGENT:*` (list only), or a `CAN_READ_INBOX_OF` grant. `:4796` enumerates the grant set for observers.
- **Message-level.** `getMessage` (:4886–4900) admits the sender, a direct recipient, or the holder of an inbox grant on a direct message. Broadcasts follow different rules:
  - A receipt-backed broadcast is readable only by the recipients snapshotted at send time.
  - A broadcast with no delivery edges is readable by everyone. That covers legacy broadcasts and known-zero ones.
  - Inbox grants never extend to broadcasts.

A semantic surface built as yet another copy of this logic would drift. And the naive filter `to ∈ {me, AGENT:*}` leaks receipt-backed broadcasts to non-recipients.

The explicit observer read (`readMailboxObservation`, #921) is a separate admission family. It admits broadcasts delivered to a granted inbox, matches former logins, and opens everything under `team`. `query_messages` follows `getMessage`, and the observer family stays out of scope.

## The Architectural Reality

- **Chroma 1.5.9 applies `where` before ranking.** On #35, foreign rows placed nearest the query never came back, on both the brute-force and HNSW paths. The filter therefore needs no post-filter, and its result is never a shortened top-k.
- **Array metadata with `$contains` works** (#1439 Fix 3, measured).
- **#1439 stores per row:** `from`, `to`, `kind` (`direct` | `broadcast`), `cohort` (`known` | `legacy-unknown`), `recipients` (receipt-backed broadcasts only, never empty), `threadId`, `sentAt`.
- **Known-zero broadcasts are open to all.** `getMessage` admits any broadcast without delivery edges, and the observer's legacy predicate includes `intendedRecipientCount = 0`. #1439's metadata cannot say so in a `where` clause: a known-zero row has `cohort: 'known'` and no `recipients` key, and Chroma has no `$exists`.
- **The plane's index needs no refresh.** On 11-03 the message WAL held 25.6k records, every identity spelled canonically. Its 6,140 broadcasts each had at least one recipient. Known-zero and legacy rows arrive only through #1440, whose shared mapper inherits this leaf's fields.
- **Identity spellings.** `getMessage` compares through `sameMailboxIdentity`, which normalizes spellings and ignores former logins.

## The Fix

1. **One admission module**, a new helper beside `mailboxReadStateClassifier.mjs` (structural pre-flight at intake).
   - `canReadMessage(me, facts)` is extracted from `getMessage` (:4886–4900). `getMessage` consumes it, with behavior unchanged.
   - `admissionWhere(me, grants)` is its set form, compiled to Chroma `where`:
     `{$or: [{from: me}, {to: me}, {recipients: {$contains: me}}, {cohort: 'legacy-unknown'}, {$and: [{kind: 'broadcast'}, {recipientCount: 0}]}, {$and: [{kind: 'direct'}, {to: {$in: grants}}]}]}`. The grant clause appears only when grants exist.
   - No tenant clause. `getMessage` admits by identity alone, and a sender-`userId` filter would hide a cross-tenant DM that its recipient can open.
2. **The mapper states what admission reads.** `toMessageEmbedRecord` writes `recipientCount` on known-cohort broadcasts. It normalizes `from`, `to` and `recipients` through `normalizeMailboxIdentityForComparison`, so the `where` compares one canonical `me`.
3. **`query_messages`**, a new memory-core MCP tool. It takes `{query, nResults, fromIdentity?, threadId?}` and returns ranked messages with `distance`.
   - Content comes from the graph node, not the stored document. A retracted message therefore shows its placeholder, never its old text, even if a stale vector lingered.
   - Each hit is re-checked with `canReadMessage` against its graph facts. A row whose metadata drifted from its graph is dropped and logged, so every returned message is one `getMessage` would open.
   - A message whose node is absent is dropped from the result.
   - Archive is a recipient-side view and stays the caller's filter, not admission.
4. **A property spec** proves the set form. The generated corpus covers:
   - direct messages, including granted inboxes;
   - receipt-backed, known-zero and legacy broadcasts;
   - sender rows and non-canonical stored spellings.

   Over that corpus, `admissionWhere` admits exactly what `canReadMessage` admits. The spec evaluates the `where` clause against each row's metadata.
5. **An adversarial falsifier**, run against a throwaway Chroma:
   - A foreign DM with near-identical wording is never returned, for any of ten phrasings.
   - Neither is a receipt-backed broadcast that was not delivered to the caller.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Error | Docs / Schema | Verification Evidence |
|---|---|---|---|---|---|
| `query_messages` (memory-core MCP, new) | #35 Fix §2–3 | Semantic similarity over admitted messages only: `{query, nResults, fromIdentity?, threadId?}` returns `{messages: [{messageId, subject, body, from, to, sentAt, distance}]}` | Unbound identity → the same error `list_messages` raises; vector store unavailable → an explicit error, never an empty list | `openapi.yaml`: "semantic similarity", as against `list_messages`' "filter + sort" | property spec + adversarial falsifier + tool-description budget audit |
| `getMessage` admission | `MailboxService.mjs:4886–4900` | Unchanged; now delegated to `canReadMessage` | Unchanged | — | existing MailboxService specs |
| `neo-agent-messages` row metadata | #1439 `toMessageEmbedRecord` | Adds `recipientCount` on known-cohort broadcasts; identities stored canonical | Rows written before this leaf lack the field; the plane holds no known-zero row, so nothing is under-admitted | mapper JSDoc | `messageWalStore.spec.mjs` |

Decision Record impact: `none`. One admission authority is shared by the get path and the recall path.

## Acceptance Criteria

- [ ] AC-1: `canReadMessage` is extracted, and `getMessage` consumes it with behavior unchanged; the existing MailboxService specs are green.
- [ ] AC-2: A property spec shows `admissionWhere` admits exactly the messages `canReadMessage` admits, over the generated corpus. The corpus includes known-zero broadcasts and non-canonical stored spellings.
- [ ] AC-3: Against a throwaway Chroma 1.5.9, a foreign DM and an undelivered receipt-backed broadcast are never returned, under any of ten phrasings.
- [ ] AC-4: `query_messages` returns graph-current content (retracted → placeholder) with `distance`. Its tool description distinguishes semantic recall from `list_messages`, within the description budget.
- [ ] AC-5: The mapper writes `recipientCount` on known-cohort broadcasts and canonical identities. A hit whose graph facts fail `canReadMessage` is dropped and logged. Both are covered by specs.

## Out of Scope

- The index writer (#1439) and the backfill (#1440).
- The observer admission family (`readMailboxObservation`).
- The memory-mining skill line: #35's leaf C, in neo-agent-skills, once this ships.

## Avoided Traps

- **Filtering after ranking:** it leaks by silence and returns a shortened top-k.
- **`to ∈ {me, AGENT:*}`:** it admits receipt-backed broadcasts to non-recipients.
- **A further admission copy:** admission and recall would drift apart.
- **Returning the stored document:** a retracted message's old text could resurface.
- **A tenant clause:** it would make recall stricter than `getMessage` for cross-tenant DMs.

## Related

- Parent: #35. Siblings: #1439 / PR #1443 (writer), #1440 (backfill).

*Prescription check (11-03, before code):* Three corrections against the first draft:
- known-zero broadcasts are admitted to all, which needs `recipientCount`;
- identities are normalized at write;
- the tenant clause is dropped.

It also names the observer family.

*Sweep attestations:*
- *Live latest-open + closed sweep:* the latest 6 open since 23:26Z, plus a search across all states ("query_messages OR semantic message recall OR admission predicate"), at 2026-11-02T23:46Z. Hits: #35, #1439, #1440 (this chain), #124 / #474 / #1298 (unrelated). No equivalent.
- *A2A in-flight sweep:* the last 8 messages, all statuses, at 23:46Z. No claim.
- *Memory Core sweep:* covered by the 23:00Z #1439 sweep on the same nouns.
- *Own-assignment sweep:* the overlap is this chain only.
- *Structure map (§1c):* run this session (exit 0). The owner is `ai/services/memory-core/helpers/` (beside `mailboxReadStateClassifier.mjs`).

Ownership: self-assigned (Vega).

Origin Session ID: afcce2e4-5eb9-4fa6-8895-4730263a042d

Retrieval Hint: "query_messages admission where canReadMessage set form receipt-backed broadcast known-zero recipientCount property spec adversarial falsifier"


## Timeline

- 2026-11-02T23:47:12Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-02T23:47:14Z @neo-opus-vega added the `enhancement` label
- 2026-11-02T23:47:14Z @neo-opus-vega added the `ai` label
- 2026-11-02T23:47:14Z @neo-opus-vega added the `security` label
- 2026-11-02T23:47:14Z @neo-opus-vega added the `agent-os` label
- 2026-11-02T23:51:38Z @neo-gemini-maryam cross-referenced by PR #1443
- 2026-11-03T01:13:45Z @neo-opus-vega cross-referenced by PR #1450
- 2026-11-03T01:23:59Z @neo-opus-vega referenced in commit `f9934b7` - "test(memory-core): pin query_messages to the read tool tier (#1445)"
- 2026-11-03T01:59:51Z @tobiu referenced in commit `6397a70` - "feat(memory-core): query_messages recalls only what get_message would open (#1445) (#1450)

* feat(memory-core): query_messages recalls only what get_message would open (#1445)

canReadMessage is extracted from getMessage into helpers/messageAdmission.mjs, and admissionWhere is its Chroma set form. query_messages pre-filters the message index with the set form and re-checks every hit against its graph with the predicate. The mapper stores canonical identities and counts a known broadcast's audience, so a known-zero broadcast is open to everyone, as getMessage opens it.

* test(memory-core): pin query_messages to the read tool tier (#1445)"
- 2026-11-03T01:59:52Z @tobiu closed this issue
- 2026-11-03T02:04:44Z @neo-opus-vega cross-referenced by PR #1451
### @neo-opus-vega - 2026-11-03T02:09:59Z

**Post-merge validation, observed on the plane** at `6397a703`. The cut finished at 02:06Z on 11-03 with all four services healthy at that revision. I called `query_messages` as @neo-opus-vega through the plane's MCP endpoint.

- **A foreign DM is never returned.** The query was the exact subject of a DM between two other agents (`MESSAGE:573224f7…`).
  - An unfiltered Chroma query ranks that DM first.
  - `query_messages` returned a full 10 results, none of them that DM.
  - Every result was sent by me, sent to me, or a broadcast.
- **A DM to me is found by its wording.** The query was the opening of Maryam's DM to me (`MESSAGE:7577aafa…`), and it ranks among the top results.
- **Ranking note.** The same DM's subject line alone does not reach the top 30 among the 491 DMs to me. The indexed document is subject plus body, and its 1.3 KB body dominates the embedding.
  - This is ranking, not an approximate-search miss. Its true L2² distance is 0.6375; the 30th filtered result's is 0.6324.
  - Queries work best with content words. #35's skill line (leaf C) should say so.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-03T02:19:58Z @neo-opus-vega cross-referenced by #1416

