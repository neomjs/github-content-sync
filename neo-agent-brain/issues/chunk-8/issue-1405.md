---
id: 1405
title: KB embed splits records at U+2028; content-sync stalled since 10-30
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-31T20:33:55Z'
updatedAt: '2026-10-31T21:10:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1405'
author: neo-opus-vega
commentsCount: 1
parentIssue: null
subIssues:
  - '[ ] 1407 JSONL readers outside the shared reader still split records at U+2028'
subIssuesCompleted: 0
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-31T21:10:23Z'
---
# KB embed splits records at U+2028; content-sync stalled since 10-30

## Context

The canonical plane's `neo-shared/github-content-sync` tenant has not embedded since 2026-10-30 18:51Z. Every sweep logs `embeddings=0 errors=1 codes=KB_VECTOR_EMBED_FAILED` about 200 ms after `Starting knowledge base embedding...`. #1382 measured the re-materialization loop this causes, and #1403 the missing trace.

Observed on the plane (Brain `4ca0f50`, Node v24.16.0), counting `kb-server-<day>.log` lines:

| Day | `Starting knowledge base embedding...` | `Loaded N knowledge chunks from file.` |
|---|---|---|
| 10-20 … 10-29 | 43–78 a day | equal to the starts |
| 10-30 | 161 | 35 |
| 10-31 (to 20:26Z) | 555 | 0 |

The last good load was at 2026-10-30T17:18:20Z (56,573 chunks, ~2 s); the first failure at 18:51:17Z. So the failure sits in `VectorService.embed` between `Starting…` (`VectorService.mjs:2468`) and `Loaded…` (`:2482`): the JSONL read.

## The Problem

`IngestionService.writeTempJsonl` serializes each chunk with `JSON.stringify` and joins the lines with `\n`. `JSON.stringify` leaves U+2028 and U+2029 raw, which is valid JSON. `VectorService.embed` reads the file with `readline`, which also breaks a line at each of them, so a record holding one arrives as two halves and `JSON.parse` throws `SyntaxError`. The `IngestionService` catch records that as the unclassified `KB_VECTOR_EMBED_FAILED`.

Reproduced in memory on the plane's Node (v24.16.0) and on v25.9.0:

```
readline over  JSON.stringify({t: 'a b'}) + '\n' + JSON.stringify({t: 'ok'})
→ first line is the half before U+2028 → JSON.parse: SyntaxError   (same for U+2029)
```

NEL, lone surrogates, CR and LF round-trip fine.

At `github-content-sync` `origin/dev`, exactly one file holds either character: `neo-agent-brain/issues/chunk-8/issue-1377.md`. Corpus commit `59744658` published it at 2026-10-30 18:36:39Z, between the last good load and the first failure. It is #1377 itself ("Bundle reader splits records at U+2028; backups fail since 10-29"), whose body quotes the raw characters. #1378 fixed the backup bundle readers, and #1377's Out of Scope listed other readline readers as latent. The KB embed reader was not on that list, and #1377's own text tripped it two hours later.

## The Architectural Reality

- `ai/services/knowledge-base/VectorService.mjs:2475-2481`: a hand-rolled `fs.createReadStream` + `readline.createInterface` + `JSON.parse` per line.
- `ai/services/shared/bundlePayload.mjs` (#1378): `openBundlePayload(filePath)` / `closeBundlePayload(rl)` read a `.jsonl`, `.jsonl.br` or `.jsonl.gz` payload line by line and hand readline U+2028/U+2029 as their JSON escapes, which parse to the same value.
- Every `embed()` caller passes a `.jsonl` path: `IngestionService.writeTempJsonl` (`ingest-….jsonl`) and `coreCorpusProfileRunner` (`<prefix>.<sha256>.jsonl`).
- Design authority: `bundlePayload.mjs`, `escapeLineSeparators`: "`JSON.stringify` leaves both characters raw, which is valid JSON, and readline breaks a line at each of them: unescaped, a record holding one reads as two".

## The Fix

`VectorService.embed` reads its JSONL through `openBundlePayload` and releases it with `closeBundlePayload` in a `finally`; the `readline` import goes. The writer stays as it is: the reader decides where a line ends, and any JSONL handed to `embed()`, not only the ingestion temp file, may hold the characters raw.

## Acceptance Criteria

- [ ] AC-1: a unit spec embeds a JSONL record whose content holds U+2028 and U+2029 as one chunk, and the text reaching the embedder carries both characters unchanged. Red on `dev` (`SyntaxError`).
- [ ] AC-2 (post-merge, plane): on a plane running the fix, `github-content-sync` logs `Loaded N knowledge chunks from file.` with `embeddings>0`, and its deferral clears. The receipt goes on #64.

## Out of Scope

- The other JSONL readers that hand-roll `readline` over a raw file: the KB artifact, upload and tenant-ingest scripts, two Memory Core restore helpers and the restore benchmark meters. They share the exposure with no live symptom; a separate ticket routes them through the shared reader and adds a guard. This one stays one file so the outage fix lands first.
- #1403's trace (stage and error class) stays its own ticket. It would have read this cause as the load stage with a `SyntaxError`.
- #1404's recovery gate is independent and still needed.

## Avoided Traps

- Escaping in `writeTempJsonl` only: it fixes one writer and leaves every other JSONL fed to `embed()` exposed. #1377 rejected writer-side escaping for the same reason.
- Editing or dropping #1377's corpus record: the text is legitimate content, and the escape keeps it exact.

## Related

#1377 / #1378 (same class, backup readers) · #1382 / #1404 · #1403 · #64

Live latest-open + closed sweep: checked the latest 20 open issues plus closed and all-state searches ("U+2028", "readline JSONL", "github-content-sync embeddings=0"; Out of Scope, Fix and Avoided Traps read on #1377) at 2026-10-31T20:33Z, and found no equivalent and no exclusion of the KB embed reader. A2A sweep (all-state, last 30, through 20:30Z): no claim on this scope. Memory Core sweep on the problem nouns ("github-content-sync embeddings=0 KB_VECTOR_EMBED_FAILED … U+2028 readline"): 8 results, no prior decision. Own-assignment sweep: #1382 and #1403 are on the same surface, and neither covers the cause.
Structure map: `npm run ai:structure-map -- --files --loc` run; no new file, so the owning folder `ai/services/knowledge-base/` is unchanged and the reused sibling is `ai/services/shared/bundlePayload.mjs`.
Decision Record impact: none.

Origin Session ID: 1a234070-11da-426b-bffe-417a090fc2d0
Retrieval Hint: "KB embed SyntaxError U+2028 readline JSONL github-content-sync embeddings=0"



## Timeline

- 2026-10-31T20:33:55Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-31T20:33:56Z @neo-opus-vega added the `bug` label
- 2026-10-31T20:33:56Z @neo-opus-vega added the `ai` label
- 2026-10-31T20:33:56Z @neo-opus-vega added the `agent-os` label
- 2026-10-31T20:39:44Z @neo-opus-vega referenced in commit `0298f92` - "chore(knowledge-base): the block fixer's alignment of the reader line (#1405)"
- 2026-10-31T20:40:43Z @neo-opus-vega cross-referenced by PR #1406
- 2026-10-31T20:42:03Z @neo-opus-vega cross-referenced by #1407
- 2026-10-31T20:45:34Z @neo-opus-vega cross-referenced by #1382
- 2026-10-31T20:45:56Z @neo-opus-vega cross-referenced by PR #1404
- 2026-10-31T20:47:09Z @neo-opus-vega cross-referenced by #1403
- 2026-10-31T20:51:39Z @neo-opus-vega added sub-issue #1407
### @neo-opus-vega - 2026-10-31T20:56:46Z

**Handover (Vega, session sunset 2026-10-31 ~21:00Z): PR #1406 is APPROVED and at the operator's merge gate.** Julia approved at exact `0298f92d` (review 5724856738, zero RAs). CI is 24/24, the PR is CLEAN, and no review request is open (I withdrew Euclid's).

Pickup after the merge:
1. Once a plane cut carries it, read `neo-shared/github-content-sync` on the plane. Expect `Loaded N knowledge chunks from file.` in `/app/.neo-ai-data/logs/kb-server-<day>.log`, `embeddings>0` in the orchestrator's materialized line, and no `deferred: embedding incomplete`. Post that as AC-2's receipt on #64.
2. Julia's non-blocking follow-up (state the reader's extension contract in its JSDoc) lives in #1407, linked here as a sub-issue.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-10-31T21:10:23Z @tobiu referenced in commit `63e99c6` - "fix(knowledge-base): a record holding U+2028 stays one line in the KB embed reader (#1405) (#1406)

* fix(knowledge-base): a record holding U+2028 stays one line in the KB embed reader (#1405)

VectorService.embed read its JSONL through a bare readline, which breaks a line at
U+2028 and U+2029 while JSON.stringify leaves both raw. A record quoting either split
into two halves and JSON.parse threw, so every github-content-sync embed since
2026-10-30 18:51Z failed as the unclassified KB_VECTOR_EMBED_FAILED. The trigger was
#1377's own body, published to the corpus at 18:36Z.

The read now goes through openBundlePayload, which hands readline both characters
as their JSON escapes (#1378), and closeBundlePayload releases the file on every exit.

* chore(knowledge-base): the block fixer's alignment of the reader line (#1405)"
- 2026-10-31T21:10:24Z @tobiu closed this issue
- 2026-10-31T21:27:23Z @neo-opus-vega cross-referenced by #1411
- 2026-10-31T21:30:57Z @neo-opus-vega cross-referenced by PR #1414
- 2026-10-31T21:33:08Z @neo-opus-vega cross-referenced by #64

