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
updatedAt: '2026-10-30T22:45:04Z'
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

`createBundlePayloadWriteStream()` composes `zlib` → `fs.createWriteStream()` for `.jsonl.br`/`.jsonl.gz`. `endBundlePayload()` awaits `finished()` on the composed stream. The module documents the deferred file-open race (the composed writer reports `finish` before a deferred open error), and `finished()` waiting for close was meant to settle it. Under full-suite threadpool load, the open's ENOENT apparently arrives after the composed stream has settled. An export can then report success for a payload that was never written.

## The Architectural Reality

`ai/services/shared/bundlePayload.mjs`: `createBundlePayloadWriteStream`, `endBundlePayload`. Callers are the Memory Core and KB exporters in the backup lane. #1378 changed only the reader side.

## The Fix

Settle the writer on the inner file stream's own outcome: await its `'open'` before ending, or settle on the file stream's `'close'`/`'error'` and not only the composed duplex's. Keep the composition for the data path.

## Acceptance Criteria

- [ ] AC-1 A spec makes the open slow and failing deterministically (an injected delayed `createWriteStream`, no load dependence). It is red on `dev` and green with the fix, for `.jsonl.br` and `.jsonl.gz`.
- [ ] AC-2 The existing bundle payload, backup and restore specs stay green, and the full unit suite stays green under 4 workers.

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

