---
id: 1384
title: 'endBundlePayload can resolve when its file cannot be opened, under load'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-30T22:45:04Z'
updatedAt: '2026-10-31T14:34:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1384'
author: neo-opus-ada
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
# endBundlePayload can resolve when its file cannot be opened, under load

## Context

Brain Unit run 41122791473 (the head job of PR #1381) failed `test/playwright/unit/ai/services/shared/bundlePayload.spec.mjs`, "a writer whose directory does not exist rejects at end() instead of reporting success". All three attempts failed with "Received promise resolved instead of rejected". The base job of the same run passed it, and so did 42/42 isolated local runs under 8 workers. #1381 touches nothing in this path (reported by Mnemosyne; defect-note 7cae6e46).

## The Problem

`createBundlePayloadWriteStream()` composes `zlib` → `fs.createWriteStream()` for `.jsonl.br`/`.jsonl.gz`. `endBundlePayload()` awaits `finished()` on the composed stream. The module documents the deferred file-open race (the composed writer reports `finish` before a deferred open error), and `finished()` waiting for close was meant to settle it. It does not: an export can report success for a payload that was never written.

Measured on `dev` (2026-10-31), deterministic: a file whose open fails 50 ms late makes `endBundlePayload` resolve for `.jsonl.br` and `.jsonl.gz` (3/3); plain `.jsonl` rejects. A saturated libuv pool alone did not reproduce it.

The same boundary has a second failure. The exporters `await` their next page between writes, so an open that fails before `endBundlePayload()` is called meets no `'error'` listener: the process dies with an unhandled `'error'` (measured for `.jsonl`; on `dev` all three encodings fail the spec below). The module's contract is that this rejects at `endBundlePayload`.

## The Architectural Reality

`ai/services/shared/bundlePayload.mjs`: `createBundlePayloadWriteStream`, `endBundlePayload`. Callers are the Memory Core and KB exporters in the backup lane and the restore filter (`ai/scripts/maintenance/restore.mjs`); all of them `write()` lines and then await `endBundlePayload`. #1378 changed only the reader side, whose `decodeThrough` already left `compose` for a plain pipe with forwarded errors after a Node 24 double emission.

## The Fix

Mirror the reader: the writer holds the compressor, which pipes into the file, and the file's errors are forwarded onto it. `endBundlePayload` settles on both the writer and its file. An error raised before `end()` stays on the stream (`errored`) and rejects there; a listener keeps it from being uncaught. `compose` leaves the module. (Filed first as "keep the composition for the data path": the measurement above showed the composed duplex itself settles early.)

## Acceptance Criteria

- [ ] AC-1 A spec makes the open slow and failing deterministically (an injected delayed `createWriteStream`, no load dependence). It is red on `dev` and green with the fix, for `.jsonl.br` and `.jsonl.gz`.
- [ ] AC-2 A spec ends the writer only after its open failed. It rejects at `endBundlePayload` with the open's error, with no uncaught error, for all three encodings. Red on `dev`.
- [ ] AC-3 The existing bundle payload, backup and restore specs stay green, and the full unit suite stays green under 4 workers.

## Related

#1378, #1381, #974.

Live latest-open + closed sweep: checked the latest 20 open issues plus a search ("endBundlePayload writer reject") at 2026-10-30T22:5xZ; no equivalent.

Retrieval Hint: "endBundlePayload composed brotli fs deferred ENOENT resolves under load"

Origin Session ID: 2e07d6d3-af03-4b39-9254-f979cc291d2f


## Timeline

- 2026-10-30T22:45:04Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-30T22:45:06Z @neo-opus-ada added the `bug` label
- 2026-10-30T22:45:06Z @neo-opus-ada added the `ai` label
- 2026-10-30T22:45:06Z @neo-opus-ada added the `agent-os` label
- 2026-10-31T14:39:46Z @neo-opus-ada cross-referenced by PR #1396

