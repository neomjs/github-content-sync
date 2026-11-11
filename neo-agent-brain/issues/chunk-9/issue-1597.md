---
id: 1597
title: Tenant ingest builds its temp JSONL as one string and hits V8's limit
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-11T04:38:32Z'
updatedAt: '2026-11-11T20:46:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1597'
author: neo-opus-vega
commentsCount: 2
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
closedAt: '2026-11-11T15:00:35Z'
---
# Tenant ingest builds its temp JSONL as one string and hits V8's limit

## Context

The corpus tenant `neo-shared/github-content-sync` has not ingested on the local plane since 2026-11-08T20:54:31Z (its last `ingest` row in `kb_ingestion_metrics`). Every sweep since then ends the same way, 28 times by 2026-11-11T03:01:02Z:

```
[2026-11-11T03:01:02.129Z] [INFO] [TenantRepoSync] neo-shared/github-content-sync materialized: envelopeFiles=59829 envelopeDeleted=0 ingested=0 deleted=0 embeddings=0 errors=1
[2026-11-11T03:01:02.130Z] [ERROR] [TenantRepoSync] neo-shared/github-content-sync failed: KB_TENANT_REPO_SYNC_SYNC_FAILED source=KB_INGEST_FAILED (Knowledge Base ingestion returned an error-bearing summary.)
```

The recorded cause, in the `detail` column of the 28 error rows (first 2026-11-08T21:25:48Z):

```
{"errors":[{"code":"KB_INGEST_FAILED","message":"Invalid string length"}]}
```

Effect: the Knowledge Base holds no issue, pull request or Discussion text newer than the checkpoint `8217ad14`, the corpus commit of 2026-11-08T17:02:54Z. Asked for the exact title of the Brain issue created on 2026-11-09 about the counts projection, `ask_knowledge_base(type: 'ticket')` returns five older references and not that issue, although its file is in the corpus (`neo-agent-brain/issues/chunk-9/`). The `ticket-create` duplicate sweep reads that same frozen state.

## The Problem

`IngestionService.writeTempJsonl` (`ai/services/knowledge-base/IngestionService.mjs:3122`) hands `fs.writeFile` a single string:

```js
await fs.writeFile(file, chunks.map(chunk => JSON.stringify(chunk)).join('\n'), 'utf8');
```

`embedChunkGroups` (`:615-627`) groups by `repoSlug`, so one tenant repository's whole accepted batch becomes that string. V8 refuses a string above `buffer.constants.MAX_STRING_LENGTH`, 536,870,888 characters on the orchestrator's Node v24.16.0, with `RangeError: Invalid string length`.

The batch is the whole corpus by design: when a profile is not delta-safe and the head moved, the envelope builder re-materializes every file (`ai/services/knowledge-base/helpers/tenantRepoIngestEnvelopeBuilder.mjs:469`), and the corpus head moves with every publish.

Measured at corpus `dev` `92e9c566`: 23,591 files, 250,928,549 characters, 256,044,602 once JSON-escaped. A normalized record carries its text twice, as `content` and as `description` (`validateAndNormalizeParsedChunk`, `:3101-3108`), so the text alone is 512,089,204 characters, 95.4 % of the limit, before the remaining fields of 59,829 records. The corpus only grows, so a retry cannot recover the lane.

The metric rows bound the throw site: `chunks_total` is 0, so the throw precedes `summary.ingested` (`:410`). A parse failure before that line is caught per file and carries its own code, and the join is the only whole-batch string between parsing and that line.

## The Architectural Reality

- `IngestionService` owns the temp file. Its reader, `VectorService.embed`, already consumes it as a line stream through `openBundlePayload` (`ai/services/knowledge-base/VectorService.mjs:2491`).
- `ai/services/shared/bundlePayload.mjs` already holds the writing half: `createBundlePayloadWriteStream` (`:310`) and `endBundlePayload` (`:333`).
- Design authority: `endBundlePayload`'s JSDoc, "Ends a payload writer and resolves once its bytes are on disk, rejecting on any error", and `openBundlePayload`'s, "Each JSONL record is one line". The temp file is a JSONL payload written without the module that defines one.
- Structure map (`npm run ai:structure-map -- --files --loc`, 2026-11-11): no new file. The change stays in `ai/services/knowledge-base/IngestionService.mjs`, whose sibling `VectorService.mjs` already imports the shared module.

## The Fix

`writeTempJsonl` writes one record per line through `createBundlePayloadWriteStream`, waits for `drain` when the stream asks for it, and settles with `endBundlePayload`. No string larger than one record exists. The bytes on disk stay what they are today for a batch that fits: records separated by `\n`, no trailing newline. A failure while writing removes the partial file and rejects.

## Acceptance Criteria

- [ ] AC-1: a unit spec shows `writeTempJsonl` never hands the file system more than one record and its separator per write, and that the file reads back through `openBundlePayload` as the same records, U+2028 and U+2029 included. Red on `dev`, where one write carries the batch.
- [ ] AC-2: a spec with a record that cannot be serialized shows `writeTempJsonl` rejects and leaves no temp file behind.
- [ ] AC-3 (before merge, real data): a scratch run builds the batch from the corpus at its current head through the service's own record path. `dev`'s writer throws `Invalid string length`; the fixed writer produces a file with one line per record. The receipt goes in the PR.

## Post-Merge Validation

- [ ] On the first cut that carries the fix, `github-content-sync` logs `ingested>0` at a head newer than `8217ad14`, its `consecutiveFailures` reads 0, and `ask_knowledge_base(type: 'ticket')` cites an issue created after 2026-11-08. The assignee runs the cut and posts the receipt here.

## Out of Scope

- The 28 error rows are filed under `repo_slug: neo`: the catch in `ingestSourceFiles` (`:431-441`) records the metric with the default repository because it has no tenant context. Sent as a defect-note.
- A tenant lane failed 28 times while the lane reported `status: completed` with `errors: []`. That is #64's subject.
- Whether the corpus profile can be delta-safe, so an advance re-reads only the files that changed. A question for the extractor catalogue, not for this outage.
- Carrying the text twice per record. Dropping `description` would have postponed this failure, not removed it, and consumers read the field.
- The next ceiling is heap: the batch still lives in memory as objects, as it does today.

## Avoided Traps

- Several smaller temp files per repository: `VectorService.embed` treats one file as one settlement population (`accepted`, `settled`, `remaining`), so a split changes its contract for a limit only the writer has.
- The same `join` in slices through `appendFile`: a second writer without backpressure and without the error contract `endBundlePayload` already carries.

## Related

#64 (tenant ingestion; at 27 sub-issues, so this ticket is not linked as another) · #1405 and #1407 (this lane's previous stall, on the reader side) · #1384 (`endBundlePayload` under load) · #65

Decision Record impact: none

Ownership: self-assigned. Not a `good first issue`: a live outage, already owned.

Live latest-open + closed sweep: checked latest 20 open + closed search (tenant ingest string length, jsonl, corpus stalled, content-sync stalled; Out of Scope / Avoided Traps read on #1405) on 2026-11-11; no equivalent or prior exclusion found. Backlog band: 155 open, red; no owning ticket can take a one-PR fix.
A2A sweep: latest 30 messages, all read-states, no claim on this scope.
MC sweep: `github-content-sync tenant ingest failing KB_INGEST_FAILED Invalid string length corpus frozen`, `knowledge base answers stale, ticket and PR conversations missing since days, content-sync tenant stalled`; 12 results, no prior decision found.
Own-assignment sweep: 17 open in this repository, bodies of #64 and #65 read, none overlapping.

Origin Session ID: c5c56705-fbd2-411a-9136-6447cb2eb379

Retrieval Hint: `query_raw_memories("corpus tenant ingest Invalid string length writeTempJsonl")` · `IngestionService.mjs:3122`

## Timeline

- 2026-11-11T04:38:32Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-11T04:38:34Z @neo-opus-vega added the `bug` label
- 2026-11-11T04:38:34Z @neo-opus-vega added the `ai` label
- 2026-11-11T04:38:34Z @neo-opus-vega added the `agent-os` label
- 2026-11-11T04:54:25Z @neo-opus-vega cross-referenced by PR #1598
- 2026-11-11T05:16:06Z @neo-opus-vega cross-referenced by #132
### @neo-opus-vega - 2026-11-11T05:41:45Z

**Handover at session sunset, 2026-11-11 (owner Vega, session `c5c56705-fbd2-411a-9136-6447cb2eb379`).**

- **State:** PR #1598 at `a5e499b7` is approved by Julia (review 5854195957, no required actions) and waits for the merge. My note on the review's two non-blocking observations is on the PR.
- **Next, mine, in my next session:** after the merge, a notice to the swarm, then the local plane is cut on the merged SHA. Three readings go here as the receipt, and the box in the PR's Post-Merge Validation gets ticked:
  1. the orchestrator log shows `github-content-sync` with `ingested>0` at a head newer than `8217ad14`;
  2. the deployment snapshot shows its `consecutiveFailures` at 0;
  3. `ask_knowledge_base(type: 'ticket')` cites an issue created that day.
- **What to expect on the first sweep:** the whole corpus is materialized again (about 59,700 records at corpus `92e9c566`). How many of them it embeds is the first number to read; it is not measured yet.
- **If the first sweep fails again:** the cause is the `detail` column of the newest `kb_ingestion_metrics` error row. It is filed under `repo_slug: neo`, not under the tenant repository (defect-note of 2026-11-11).

— Vega (Fable 5.2, Claude Code) 🌿

- 2026-11-11T15:00:35Z @tobiu referenced in commit `443cc90` - "fix(knowledge-base): write the ingest temp file one record at a time (#1597) (#1598)

writeTempJsonl joined one repository's whole batch into a single string.
The corpus tenant's batch (59,671 records, 541.7 M characters at corpus
92e9c566) passed V8's limit of 536,870,888 characters on 2026-11-08, and
every sweep since then failed with "Invalid string length".

The batch now goes through the shared bundlePayload writer one record per
write and waits for the stream's drain. The bytes on disk are unchanged:
records separated by ""
- 2026-11-11T15:00:37Z @tobiu closed this issue
### @neo-opus-vega - 2026-11-11T17:59:01Z

**Final receipt, 2026-11-11 (local plane, Brain `7219f4e1`): all three readings hold.**

- **Log.** The cut ran at 17:04Z and killed one attempt, so the tenant waited out its backoff and retried by itself at 19:03:59Z. The first slice, 19:09:07Z: `neo-shared/github-content-sync materialized: envelopeFiles=60060 envelopeDeleted=0 ingested=60123 deleted=0 embeddings=160 errors=0`, with `streak 35 -> 0`. Nineteen slices in all, 4,062 embeddings, no error. The last, 20:44:12Z: `neo-shared/github-content-sync completed: head=c5e0c0bf ingested=60123 deleted=186 (270454ms)`. `c5e0c0bf` (2026-11-11T14:21Z) is 12 commits past `8217ad14` (2026-11-08T17:02Z).
- **Manifest**, 20:44:20Z: `consecutiveFailures: 0`, `lastIngestedRev: c5e0c0bf`, nothing outstanding.
- **Retrieval.** `ask_knowledge_base(type: 'ticket')`, asked which ticket reports the `Invalid string length` failure of this tenant, names this ticket and cites `neo-agent-brain/issues/chunk-9/issue-1597.md`. The ticket was created on 2026-11-11.

Two things the readings showed beside the criterion:

- For this ticket's literal title the same tool does not cite it: `query_documents` returns it at rank 12 of 12 (score 100; the top hit scores 398). A ranking matter, sent as a defect-note.
- Every slice re-materialized all 60,060 envelope files to embed between 100 and 430 chunks, about five minutes each, 100 minutes from the retry to the checkpoint.

The box in PR #1598 is ticked.

— Vega (Claude Fable 5.2, Claude Code) 🌿

- 2026-11-11T18:57:06Z @neo-opus-vega cross-referenced by #1608
- 2026-11-11T19:38:23Z @neo-gemini-maryam cross-referenced by PR #1611

