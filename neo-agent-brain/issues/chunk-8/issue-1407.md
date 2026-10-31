---
id: 1407
title: JSONL readers outside the shared reader still split records at U+2028
state: OPEN
labels:
  - bug
  - good first issue
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-31T20:42:02Z'
updatedAt: '2026-10-31T20:51:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1407'
author: neo-opus-vega
commentsCount: 0
parentIssue: 1405
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
# JSONL readers outside the shared reader still split records at U+2028

## Context

One defect class hit twice in two days. #1377: the backup bundle readers, fixed by #1378 with a shared escaping reader. #1405: the KB embed reader, which stalled `github-content-sync` from 2026-10-30 18:51Z after #1377's own body reached the corpus. Each fix covered only the reader that failed. #1377's Out of Scope had already named "other JSONL readers that use readline outside the bundle path … which share the latent behavior". No ticket carried them, and the next one failed two hours later.

## The Problem

`JSON.stringify` leaves U+2028 and U+2029 raw, which is valid JSON. Node's `readline` breaks a line at each of them; this is verified on v24.16.0, the plane's runtime, and on v25.9.0. Any reader that pipes a raw JSONL file into `readline.createInterface` therefore turns a record holding either character into two unparseable halves. After #1405, these readers still do it:

| Reader | Input |
|---|---|
| `ai/services/memory-core/helpers/vectorJsonlImport.mjs:155` | `fs.createReadStream(filePath, {encoding: 'utf8'})` |
| `ai/services/memory-core/helpers/restoreTargetSetAdmission.mjs:184` | the same |
| `ai/scripts/maintenance/knowledgeBaseArtifact.mjs:279` | `fs.createReadStream(jsonlPath)` |
| `ai/scripts/maintenance/uploadKnowledgeBase.mjs:107` | `fs.createReadStream(filePath)` |
| `ai/scripts/maintenance/ingestTenant.mjs:92` | a file stream or stdin |
| `ai/scripts/benchmark/restore-empty-target-meter.mjs:350`, `:389`, `:435` | file streams |

`ai/scripts/setup/firstRun.mjs:367` is an interactive prompt, not JSONL. None of these readers shows a symptom yet. Each will fail the first time its input carries the character, and both memory rows and KB content can carry it.

## The Architectural Reality

- `ai/services/shared/bundlePayload.mjs` holds the one escaping reader. `openBundlePayload(filePath)` and `closeBundlePayload(rl)` pick the encoding by extension (`.jsonl`, `.jsonl.br`, `.jsonl.gz`) on top of the module-private `escapeLineSeparators(input)` transform. The module doc scopes it to backup bundles, although #1405 now uses it for KB ingestion.
- `ingestTenant.mjs` reads stdin as well as files, so it needs a stream-input entry point beside the path-based one.
- Design authority: `bundlePayload.mjs`, `escapeLineSeparators`: "`JSON.stringify` leaves both characters raw, which is valid JSON, and readline breaks a line at each of them: unescaped, a record holding one reads as two".

## The Fix

1. Route every reader above through the shared reader: the path-based ones through `openBundlePayload` / `closeBundlePayload`, and stdin through an exported stream entry point over `escapeLineSeparators`.
2. Make the class unrepeatable with a lint or census spec that fails on `readline.createInterface` outside the shared module, with an explicit allowlist for interactive prompts.
3. Settle the module's name: either keep `bundlePayload` and widen its doc to every JSONL payload, or move the reader into a JSONL-generic module beside it. The PR records which, and why. Whichever name wins, the reader's JSDoc states its extension contract (`.jsonl`, `.jsonl.br`, `.jsonl.gz`; any other suffix throws `BUNDLE_PAYLOAD_EXTENSION`), so a temp path with another suffix fails loudly by design rather than by surprise. That follow-up came from Julia's review on #1406.

## Acceptance Criteria

- [ ] AC-1: each listed production reader keeps a record holding U+2028 and U+2029 whole, through a spec per reader or one parametrized spec. Red on `dev`.
- [ ] AC-2: the guard runs in CI. It fails on a newly added `readline.createInterface` over a file outside the shared module and passes on the allowlisted prompt.

## Out of Scope

- The KB embed reader, which #1405 covers.
- Writer-side escaping, which #1377 rejected as defense in depth once the shared reader is right.

## Avoided Traps

- Fixing only the reader that fails next. That is how this class reached a second outage; the guard is the deliverable that ends it.
- A hand-written `\n` splitter: #1377 rejected re-deriving readline's torn-file and close semantics.

## Related

#1377 / #1378 · #1405 / #1406

unowned-rationale: Ready for a contributor; maintainers provide planning and review. The design is settled and public, the reproduction is one line, and nothing is live-broken.

Live latest-open + closed sweep: checked the latest 20 open issues plus all-state searches ("readline U+2028", "JSONL reader split record", org-wide "U+2028"; Out of Scope and Avoided Traps read on #1377) at 2026-10-31T20:41Z, and found no equivalent; #1377's Out of Scope names the gap without an owner. A2A sweep (all-state, last 10, through 20:41Z): no claim on this scope. Memory Core sweep on the problem nouns ("JSONL reader readline splits record line separator restore import fails parse"): 6 results, no prior decision. Own-assignment sweep: #1405 is the only same-surface hit, and it covers one reader.
Structure map: `npm run ai:structure-map -- --files --loc` run. No new file is needed unless the reader moves; it would then sit in `ai/services/shared/` beside `bundlePayload.mjs`.
Decision Record impact: none.

Origin Session ID: 1a234070-11da-426b-bffe-417a090fc2d0
Retrieval Hint: "readline U+2028 JSONL reader split record guard lint shared reader"



## Timeline

- 2026-10-31T20:42:03Z @neo-opus-vega added the `bug` label
- 2026-10-31T20:42:04Z @neo-opus-vega added the `good first issue` label
- 2026-10-31T20:42:04Z @neo-opus-vega added the `ai` label
- 2026-10-31T20:42:04Z @neo-opus-vega added the `agent-os` label
- 2026-10-31T20:49:56Z @neo-gemini-julia cross-referenced by PR #1406
- 2026-10-31T20:51:39Z @neo-opus-vega added parent issue #1405
- 2026-10-31T20:56:48Z @neo-opus-vega cross-referenced by #1405
- 2026-10-31T21:29:05Z @imshah-id cross-referenced by PR #1412

