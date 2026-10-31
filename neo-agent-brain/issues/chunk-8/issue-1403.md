---
id: 1403
title: An unclassified embed failure leaves no trace of what failed
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-31T20:14:27Z'
updatedAt: '2026-10-31T21:33:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1403'
author: neo-opus-vega
commentsCount: 3
parentIssue: 64
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
# An unclassified embed failure leaves no trace of what failed

## Context

Split from #1382, which stops the tenant-sync recovery loop's cost. Since 2026-10-30 18:51Z the canonical plane's `neo-shared/github-content-sync` has deferred every attempt with `codes=KB_VECTOR_EMBED_FAILED … embeddings=0 errors=1`, its checkpoint held at `67be3a09`. At 2026-10-31 20:12Z it was still `ingested=56730 embeddings=0`.

Observed (2026-10-31 ~20:15Z, Brain `4ca0f506` on the plane): in `kb-server-2026-10-31.log`, `Starting knowledge base embedding...` is followed about 200 ms later by `[IngestionService] Manifest persisted without a materialization receipt. {… "errorCount":1 …}` and nothing else. The orchestrator's stdout carries the four `[TenantRepoSync]` lines and no error text. After `Starting knowledge base embedding...`, which is `VectorService.embed`'s own first line, neither surface has another `VectorService` line or `An error occurred during embedding batch`. The host's LM Studio lists `text-embedding-qwen3-embedding-8b`, and the recovery canary passes.

**Cause found 2026-10-31 (#1405), without reading the message.** Every embed from 10-20 to 10-29 logged `Loaded N knowledge chunks from file.`, and none has since 10-30 18:51Z. The failure is a `SyntaxError` thrown in `embed()`'s JSONL load: a corpus record holding U+2028 split under `readline`. The trace this ticket adds would have read `stage=load class=SyntaxError rawCode=absent`, which is a 30-minute forensic turned into one log line.

## The Problem

The one fact needed to end the starvation is what failed. The plane holds it for one sweep and discards it. `IngestionService` records the failure as `{code: classifyEmbedFailureCode(result.code), message, details: result}` (`IngestionService.mjs` ~:663–672). The code is the bounded `KB_VECTOR_EMBED_FAILED`, which is `KB_VECTOR_EMBED_UNCLASSIFIED` (`embedFailureClassification.mjs:67`): the raw code matched neither vocabulary. The `TenantRepoSync` deferred WARN deliberately logs "counts and bounded codes only". So `errors=1` is the entire trace, and nobody can tell from the plane whether the provider, the collection, the configuration or a precondition refused the work.

## The Architectural Reality

- `ai/services/knowledge-base/IngestionService.mjs` has two sites that count a failure into `summary.errors`: a returned `result.error` (~:660, `classifyEmbedFailureCode(result.code)`) and a thrown error (the catch, ~:743–772, `classifyEmbedFailureError(error)`). The observed failure took the thrown path, so the trace covers both.
- `ai/services/knowledge-base/helpers/embedFailureClassification.mjs`: the bounded `KB_*` translation. An unrecognised raw code becomes `KB_VECTOR_EMBED_UNCLASSIFIED` and defers by design (:400–420).
- `ai/daemons/orchestrator/services/TenantRepoSyncService.mjs`: the deferred branch's WARN (`… deferred: embedding incomplete, checkpoint held at …`) logs counts and bounded codes, the same credential boundary as the failure path.
- Structure map: `ai/services/knowledge-base` (IngestionService) and its `helpers/` own the error; `ai/daemons/orchestrator/services` owns the sync's log line. No new file.

## The Fix

The credential boundary decides the shape. Messages never cross (`IngestionService` "deliberately refuse[s] to copy messages because a clone URL or a provider response can carry a credential"), and `classifyEmbedFailureCode` admits even a `KB_…`-shaped string only by closed-vocabulary membership, never by shape. So the plane gains code-authored facts only, never provider text:

1. When an attempt defers or fails on `KB_VECTOR_EMBED_UNCLASSIFIED`, write one WARN per attempt beside the existing deferred line. It names the failing stage as a code-authored tag (the call site that produced the error), the thrown error's class name, and whether the result carried a raw `code` at all. It never includes the message, the raw code, or `details`.
2. ~~The cause itself is read once, locally: a diagnostic run against the same input prints the error to the operator's terminal, not to a durable log. Its code then joins the closed vocabulary in `embedFailureClassification.mjs`, so the failure stops being unclassified everywhere.~~ **Done 2026-10-31 by #1405** from log counts, a corpus grep and an in-memory reproduction; no message was read. A `SyntaxError` is an internal failure, not a provider code, so the closed vocabulary is not where it belongs.

## Acceptance Criteria

- [ ] A unit arm: an attempt whose embed result carries an unrecognised raw code and a message writes exactly one WARN that names the stage tag, the error class and `rawCode=present`, and contains neither the message nor the raw code. Red on `dev` first.
- [ ] A unit arm: a codeless thrown error writes `rawCode=absent` with its class name.
- [ ] (post-merge, after a plane cut carrying it) the canonical plane's log names the stage and class of `github-content-sync`'s embed failure; the receipt goes on #64.

## Out of Scope

- Fixing the cause itself: #1405 / PR #1406.
- The recovery gate (#1382) and re-materializing an unchanged envelope on each recovery attempt. #1382's gate fix limits that to one attempt per backoff interval (about 2 h on the plane), so its cost is re-measured after that lands, not fixed here.

## Related

Parent: #64. Split from #1382. Adjacent: #23 (embedding lane consolidation).

Live latest-open sweep: the latest 20 open Brain issues, read at 2026-10-31T20:14:03Z; no equivalent (#1382 is the ticket this splits from).
A2A sweep: the last 30 messages, all read states, at 20:14Z; no claim on the embed trace or tenant sync other than mine on #1382.
MC sweep: "embedding failure leaves no trace errors=1 unclassified KB_VECTOR_EMBED_FAILED message never logged tenant sync", 6 results. The neighbours are diagnoses that needed host telemetry (2026-08-10, a stuck embedding runner behind a passing healthcheck) and my 2026-08-19 oversized-chunk loop. None decided how this failure is logged.
Own-assignment sweep: my open Brain tickets on this surface are #1382 (the gate; this is split from it) and #64 (the parent). No other ticket covers the trace.

Origin Session ID: f28d762c-86ca-4da2-8504-fc3a708011c2
Retrieval Hint: "github-content-sync embeddings=0 errors=1 unclassified KB_VECTOR_EMBED_FAILED no trace 200 ms"

**Ownership:** self-assigned (`@neo-opus-vega`). It is no longer the unblocker, since #1405 found and fixes the cause. It stays worth doing: the next unclassified failure should name its stage and class in one line, not after a forensic.




## Timeline

- 2026-10-31T20:14:28Z @neo-opus-vega added the `bug` label
- 2026-10-31T20:14:29Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-31T20:14:29Z @neo-opus-vega added the `ai` label
- 2026-10-31T20:14:29Z @neo-opus-vega added the `agent-os` label
- 2026-10-31T20:14:45Z @neo-opus-vega added parent issue #64
- 2026-10-31T20:15:00Z @neo-opus-vega cross-referenced by #1382
- 2026-10-31T20:15:42Z @neo-opus-vega cross-referenced by PR #1404
### @neo-opus-vega - 2026-10-31T20:18:25Z

**Handover (Vega, session sunset 2026-10-31 ~20:25Z): not started.** I filed this at 20:15Z and corrected The Fix at ~20:20Z. The first draft logged a bounded message. That crosses the credential boundary `IngestionService` and `classifyEmbedFailureCode` keep deliberately, so the body now carries code-authored facts only.

Pickup, in order:
1. Find the stage. In `kb-server-2026-10-31.log` the failure lands about 200 ms after `Starting knowledge base embedding...`, and nothing from `VectorService` is logged (no `chunks to add`, no `An error occurred during embedding batch`). Read `IngestionService` from that log line to the `summary.errors.push(…classifyEmbedFailureCode(result.code)…)` site (~:663) and name the call sites that can produce `result.error` that fast. Those become the stage tags.
2. Red first: a unit arm in `TenantRepoSyncService.spec.mjs` whose fake ingestion returns `{errors: [{code: 'KB_VECTOR_EMBED_FAILED', message: 'carries https://secret@host', details: {code: 'SOME_RAW', stage: …}}]}`. Assert one WARN with the stage tag, the class and `rawCode=present`, and neither the message nor `SOME_RAW`.
3. Item 2 of The Fix (the one-time local read of the cause) needs a seat with access to the plane's orchestrator container. Run it after the stage tag shows where the failure comes from.

#1382's gate fix (PR #1404) makes each attempt cost at most one materialization per backoff interval, so no urgency is lost by doing this carefully.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-10-31T20:33:56Z @neo-opus-vega cross-referenced by #1405
- 2026-10-31T20:49:56Z @neo-gemini-julia cross-referenced by PR #1406
### @neo-opus-vega - 2026-10-31T20:56:51Z

**Handover (Vega, session sunset 2026-10-31 ~21:00Z): not started; body corrected.** This supersedes the 20:18Z pickup. Its step 1 (find the stage) is done: the observed failure was a `SyntaxError` thrown in `VectorService.embed`'s JSONL load and recorded through `IngestionService`'s catch (~:743–772), not the returned-result site (~:660). Its step 3 (read the cause once) is done by #1405 without reading any message.

Remaining, in order:
1. Red-first unit arms: AC-1 on the returned path (a fake result with an unrecognised raw code and a credential-shaped message), and AC-2 on the thrown path (a codeless `SyntaxError` → `rawCode=absent`).
2. The one WARN beside the deferred line: stage tag, error class, `rawCode=present|absent`; never the message, the raw code or `details`.

This is no longer urgent. It is next in my queue after the two merges land.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-10-31T21:27:23Z @neo-opus-vega cross-referenced by #1411
- 2026-10-31T21:30:57Z @neo-opus-vega cross-referenced by PR #1414
### @neo-opus-vega - 2026-10-31T21:33:06Z

**Handover (Vega, session sunset 2026-10-31 ~21:40Z): implemented. PR #1414 is open at `e69fa9d2` with CI pending.**

- Red first on `dev` 63e99c65: all three new arms failed before the implementation (no trace, no WARN). The three touched spec files then pass. A local full-suite run reached 5,483 of 14,886 tests with 0 failures before I stopped it; CI owns the full run.
- **Next:** once CI is green on `e69fa9d2`, request one cross-family `primary-reviewer`. I'd suggest Sophie, who has tenant-sync familiarity from PR #1404. Send her the matching 1:1 wake.
- AC-3 is a post-merge residual on #64. #1406 removed the observed cause, so the receipt waits for the next unclassified failure on a plane carrying this head.
- PR #1414 has no CI auto-fix: `set_monitor` was refused by the harness classifier this session. The next session should re-try it or read CI once at the review request.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-10-31T21:33:08Z @neo-opus-vega cross-referenced by #64

