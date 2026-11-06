---
id: 1491
title: update_issue_body overwrites rows another seat ticked since your read
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-05T19:41:09Z'
updatedAt: '2026-11-05T21:25:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1491'
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
closedAt: '2026-11-05T21:25:15Z'
---
# update_issue_body overwrites rows another seat ticked since your read

## Context

Shared tracker bodies are now edited by several seats at once. On 2026-11-05 Julia, Vega, Maryam and I ticked rows in the body of neomjs/neo#20358 within the same half hour, and the same pattern ran on neomjs/neo#15000 and neomjs/neo-agent-brain#1438.

`update_issue_body` replaces the whole body with what the caller sends. A seat that read the body, spent a minute verifying a row, and then wrote its edit silently erases every row another seat ticked in that minute. Nothing reports the loss. I avoided it only by hand: re-fetch, diff against my earlier read, and publish only if they match. Every seat would have to remember to do that.

## The Architectural Reality

`IssueService#updateIssueBody` (`ai/services/github-workflow/IssueService.mjs`) already reads the live body before writing. It needs it for `previousBytes`, `byteDelta` and the author/assignee guard (`GET_ISSUE_BODY_TARGET` / `GET_PULL_REQUEST_BODY_TARGET`). The comparison a stale-write guard needs therefore costs no extra request.

GitHub's `updateIssue` / `updatePullRequest` mutations take no precondition, so the check happens between the service's read and its write. That narrows the lost-update window from the caller's whole think time to milliseconds; it does not close it.

## The Fix

- **The precondition:** add an optional `expectedBodySha256` to `update_issue_body`, the SHA-256 hex of the body the caller read. When given and different from the live body's hash, refuse with `BODY_CHANGED`. The refusal returns `currentBodySha256` and sends no mutation.
- **The chain:** return `bodySha256` (the new body's hash) on success, so a caller can chain edits without re-reading.
- **Compatibility:** omitting the field keeps today's behaviour.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `update_issue_body` `expectedBodySha256` | This ticket | Refuse with `BODY_CHANGED` + `currentBodySha256` when the live body's hash differs, before any mutation | Omit it: unconditional write, as today. Only omission does that: a supplied blank, malformed or non-string value refuses with `INVALID_EXPECTED_BODY_SHA256` before any request (added 2026-11-05 at review RA-1, 5791574778) | `openapi.yaml` description and property | Unit spec: match writes, mismatch refuses with no mutation on issue and PR bodies, omitted writes, unusable values refuse with no request |
| `update_issue_body` result `bodySha256` | This ticket | SHA-256 hex of the body just written | — | Same | Unit spec |

## Acceptance Criteria

- [ ] With a matching `expectedBodySha256`, the body is written and the result carries `bodySha256`.
- [ ] With a stale `expectedBodySha256`, the call refuses with `BODY_CHANGED`, returns `currentBodySha256`, and sends no mutation.
- [ ] Without the field, behaviour is unchanged.
- [ ] The hash is taken over the exact `body` string the GitHub API returns, which may lack a trailing newline the caller's file has. That is documented in the property description.

## Out of Scope

- Retry or merge logic. The caller re-reads and re-applies its edit.
- Skill text telling tracker editors to pass the hash. Follow-up once this ships.

Live latest-open + closed sweep: checked the latest 10 open Brain issues, plus "update_issue_body concurrent" and "issue body lost update". No equivalent (2026-11-05 19:45Z). A2A in-flight sweep: none. Own-assignment sweep: none.

Origin Session ID: 89c130ad-8235-41ef-914c-800c028ac868

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code



## Timeline

- 2026-11-05T19:41:10Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-05T19:41:10Z @neo-opus-ada added the `enhancement` label
- 2026-11-05T19:41:11Z @neo-opus-ada added the `ai` label
- 2026-11-05T19:41:11Z @neo-opus-ada added the `agent-os` label
- 2026-11-05T19:41:27Z @neo-opus-ada cross-referenced by #1492
- 2026-11-05T19:45:18Z @neo-opus-ada cross-referenced by PR #1493
- 2026-11-05T20:41:44Z @neo-opus-ada referenced in commit `7a531da` - "fix(github-workflow): refuse an unusable expectedBodySha256 (#1491)

A supplied blank, whitespace, malformed or non-string hash was read as
omitted, so the write went through unconditionally. Now only omission
keeps the write unconditional: any supplied value must be 64 hex
characters, or the call is refused with INVALID_EXPECTED_BODY_SHA256
before any GitHub request.

The update_issue_body description drops from 1,404 to 966 characters
(1,105 on dev), under the 1,024-character tool-description cap, and
states that the check is not atomic."
- 2026-11-05T21:25:15Z @tobiu referenced in commit `c311a13` - "feat(github-workflow): update_issue_body refuses a stale write (#1491) (#1493)

* feat(github-workflow): update_issue_body refuses a stale write (#1491)

An optional expectedBodySha256 is compared against the body the service already reads before writing; a mismatch refuses with BODY_CHANGED and currentBodySha256, and a guarded write returns bodySha256 for the next one. Without the field nothing changes.

* fix(github-workflow): refuse an unusable expectedBodySha256 (#1491)

A supplied blank, whitespace, malformed or non-string hash was read as
omitted, so the write went through unconditionally. Now only omission
keeps the write unconditional: any supplied value must be 64 hex
characters, or the call is refused with INVALID_EXPECTED_BODY_SHA256
before any GitHub request.

The update_issue_body description drops from 1,404 to 966 characters
(1,105 on dev), under the 1,024-character tool-description cap, and
states that the check is not atomic."
- 2026-11-05T21:25:17Z @tobiu closed this issue

