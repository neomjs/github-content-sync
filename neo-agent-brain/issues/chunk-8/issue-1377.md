---
id: 1377
title: Bundle reader splits records at U+2028; backups fail since 10-29
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-30T16:48:18Z'
updatedAt: '2026-10-30T17:59:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1377'
author: neo-opus-ada
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
blocking: []
closedAt: '2026-10-30T17:42:24Z'
---
# Bundle reader splits records at U+2028; backups fail since 10-29

## Context

The local plane's daily backup failed its bundle integrity check on 2026-10-29 and again on 2026-10-30. The lane's retry budget is exhausted until 2026-10-31T14:51Z, and the newest restorable bundle is `backup-2026-10-28T13-50-26.861Z`. The 10-30 receipt reads `mc: row-count mismatch: source=58104, bundle=58106 (delta 2)` and `graph: row-count mismatch: source=807433, bundle=807434 (delta 1)`. The same revision (`436aec63`) passed on 10-28 (mc 57211 = 57211, graph 788592 = 788592). The failure was first captured as an A2A defect-note.

## The Problem

- Exporters write `JSON.stringify(record) + '\n'`. `JSON.stringify` leaves U+2028 (LINE SEPARATOR) and U+2029 (PARAGRAPH SEPARATOR) unescaped, which is valid JSON.
- `openBundlePayload` reads every payload through `readline.createInterface`. Measured in the plane's orchestrator container (Node v24.16.0), readline breaks a line at U+2028 and at U+2029: one record holding either character yields 2 lines. U+0085 yields 1.
- Each raw U+2028/U+2029 therefore adds one phantom line. `verifyBundleIntegrity` counts with `countNonEmptyJsonlLines`, so it reports bundle > source, and the bundle is discarded.
- A read-only scan of the live store finds exactly two records holding the character, both written after the last good bundle:
  - memory `1536ce98-2313-45b4-89e5-682a48b431e2` (2026-10-28T16:34Z) holds it once in its document and once in metadata `thought`, which accounts for mc +2;
  - graph node `MESSAGE:daf2b770-bcfc-4bb2-a4cf-75997fe8e974` (2026-10-29T02:36Z) holds it once, which accounts for graph +1.
- Sessions, temporal summaries and graph edges hold none. The occurrence counts equal the deltas.
- Restore reads through the same reader (`restore.mjs`, `restoreReceipts.mjs`), so a bundle holding such a record would split it into two unparseable fragments during a restore. The integrity check is right to refuse the bundle; the reader is wrong.
- While those records exist, every run fails the same way, so retries cannot help.

## The Architectural Reality

- `ai/services/shared/bundlePayload.mjs`: `openBundlePayload` wraps `createBundlePayloadReadStream` in readline (`crlfDelay: Infinity`). `closeBundlePayload` closes `rl` and destroys `rl.input`. `decodeThrough` already forwards a file's errors onto the stream the reader holds.
- Consumers: `ai/scripts/maintenance/backup.mjs#countNonEmptyJsonlLines`, `ai/scripts/maintenance/restore.mjs` (4 call sites) and `ai/scripts/maintenance/restoreReceipts.mjs` (2).
- The bundle contract, per the function's own JSDoc, is one record per line, and lines are `\n`-terminated. JSON allows U+2028/U+2029 raw inside strings; only JavaScript's line-terminator set treats them as breaks.

## The Fix

In `openBundlePayload`, place a transform between the decoded stream and readline. The transform rewrites raw U+2028/U+2029 to the JSON escapes ` `/` `, decoding with a `StringDecoder` so that a character split across chunks is still caught. Inside a JSON string the escape parses to the same value: counts match the source, and restore yields records equal to the ones written. The transform gets the source's errors forwarded to it, and destroying it destroys the source (the `decodeThrough` pattern), so torn-file rejection and `closeBundlePayload` behave exactly as now.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `openBundlePayload` line stream | its JSDoc: each JSONL record is one line | one yielded line per `\n`-terminated record; raw U+2028/U+2029 arrive as their JSON escapes | none needed: parse-equal | the function's JSDoc | unit spec, red on `dev` |

## Decision Record impact

none

## Acceptance Criteria

- [x] AC-1 A unit spec writes a payload in each encoding (`jsonl`, `jsonl+br`, `jsonl+gz`) holding a record whose document contains U+2028 and a record whose metadata contains U+2029. `countNonEmptyJsonlLines` returns the record count. The spec is red on `dev` and green on the branch.
- [x] AC-2 The same payloads, read through `openBundlePayload` and `JSON.parse`, yield records deep-equal to the written ones.
- [x] AC-3 A U+2028 whose three UTF-8 bytes straddle a chunk boundary is still escaped; the spec forces that boundary.
- [x] AC-4 The existing bundle payload, backup and restore specs stay green, including torn-file rejection and the early-close path.
- [x] AC-5 (post-merge, after a plane cut to a `dev` head containing this fix) The plane's next backup publishes a bundle whose mc and graph integrity checks pass. Its receipt goes on this ticket.

## Out of Scope

- Editing the two records: a data mutation that would hide the defect.
- Writer-side escaping in each exporter: defense in depth, not needed once the shared reader is right.
- Other JSONL readers that use readline outside the bundle path (`ingestTenant.mjs`, `knowledgeBaseArtifact.mjs`, a restore benchmark), which share the latent behavior.
- The backup lane's retry budget and schedule.

## Avoided Traps

- Skipping or loosening the integrity check. It is the guard that kept an unrestorable bundle from being published.
- Replacing readline with a hand-written splitter, which would re-derive the reader's torn-file and close semantics.
- Running the backup again. The failure is deterministic while those records exist.

## Related

- #974 (bundle payload compression); its Out of Scope and Avoided Traps exclude nothing here.
- neomjs/neo-agent-institution#516: today's plane cut (H2) goes to `dev` head, so landing this first lets a backup right after the cut succeed.

Live latest-open + closed sweep: checked the latest 20 open issues plus closed searches ("backup integrity line", "readline bundle payload"; Out of Scope and Avoided Traps read on #974) at 2026-10-30T16:5xZ, and found no equivalent and no prior exclusion. A2A sweep (all-state, last 30, through 16:47Z): no claim on this scope. Memory Core sweep on the problem nouns and own-assignment sweep: no equivalent. Structure map: `ai/services/shared/bundlePayload.mjs` is an existing file in its owning folder; no new file.

Retrieval Hint: "backup integrity row-count mismatch U+2028 readline line separator bundle payload"

Origin Session ID: 7c778f69-b8e7-427d-bcb5-cebe79b24b3d

## Timeline

- 2026-10-30T16:48:18Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-30T16:48:20Z @neo-opus-ada added the `bug` label
- 2026-10-30T16:48:20Z @neo-opus-ada added the `ai` label
- 2026-10-30T16:48:21Z @neo-opus-ada added the `agent-os` label
- 2026-10-30T16:52:59Z @neo-opus-ada cross-referenced by PR #1378
- 2026-10-30T17:00:13Z @neo-opus-ada referenced in commit `cbdd454` - "fix(backup): the reader's comments state its reason, not its ticket (#1377)"
- 2026-10-30T17:42:24Z @tobiu referenced in commit `4ca0f50` - "fix(backup): a record holding U+2028 stays one line in the bundle reader (#1377) (#1378)

* fix(backup): a record holding U+2028 stays one line in the bundle reader (#1377)

JSON.stringify leaves U+2028/U+2029 raw and readline breaks a line at
both, so one record read as two: every daily backup since 10-29 failed
its row-count parity (mc +2, graph +1 = the raw separators in two live
records), and restore would have split those rows. openBundlePayload now
hands readline the decoded payload with both characters as their JSON
escapes, which parse to the same value.

* fix(backup): the reader's comments state its reason, not its ticket (#1377)"
- 2026-10-30T17:42:25Z @tobiu closed this issue
- 2026-10-30T17:44:45Z @neo-opus-ada cross-referenced by #516
### @neo-opus-ada - 2026-10-30T17:59:31Z

## AC-5 receipt: the first backup after the fix publishes a passing bundle (2026-10-30)

With the operator's authorization, the local plane was cut to Brain `dev` `4ca0f506` (this ticket's merge) at 17:50:42Z. The backup run that followed published `backup-2026-10-30T17-51-10.172Z`:

| subsystem | source | bundle | status |
|---|---|---|---|
| kb | 86,333 | 86,333 | pass |
| mc | 58,191 | 58,191 | pass |
| graph | 808,917 | 808,917 | pass |

- `bundle-meta.json` names `gitSha 4ca0f506`. The receipt reads `status: success, restorable: true`, finished 17:58:09Z.
- The three raw U+2028 that failed the 10-29 and 10-30 runs are still in the store, so the counts match with them present.
- A read-back of the published graph payload through `openBundlePayload` finds `MESSAGE:daf2b770…` on one line. The line carries the escaped form once, and `JSON.parse` returns a value holding its one U+2028, the same as the live record.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code



