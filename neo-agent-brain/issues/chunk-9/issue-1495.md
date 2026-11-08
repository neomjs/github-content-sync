---
id: 1495
title: SDK clients reject structured tool errors as invalid success results
state: CLOSED
labels:
  - bug
  - good first issue
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-05T20:24:48Z'
updatedAt: '2026-11-08T20:55:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1495'
author: neo-gpt-euclid
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-08T20:55:37Z'
---
# SDK clients reject structured tool errors as invalid success results

## Context

Reviewing #1493 at `3dedc2efb025aac8762843522bd0db4db4ec34e0` exposed an inherited MCP consumer failure. The new `BODY_CHANGED` refusal and the incumbent `FOREIGN_BODY_FORBIDDEN` refusal both retain their complete objects in the server's `structuredContent`, but the locked SDK 1.32.2 client rejects them:

```
-32602 Structured content does not match the tool's output schema:
data must have required property 'id' ... 'url' ... 'updatedAt' ...
'previousBytes' ... 'updatedBytes' ... 'byteDelta'
```

Matching and unguarded successful body-write controls pass. No GitHub body was changed in the probe.

## The Problem

The listed output schema describes success only. A client that validates the actual structured error against that listing throws a schema error instead of returning the typed refusal and its recovery metadata. This also affects an existing refusal, so it is separate from #1493's hash guard.

## The Architectural Reality

- `ai/mcp/server/BaseServer.mjs#formatToolResult` preserves object errors with `isError: true` and complete `structuredContent`.
- `ai/mcp/validation/openApiValidator.mjs#buildOutputZodSchema` selects the 200/201/202 success response only.
- `github-workflow/openapi.yaml#UpdateIssueBodyResponse` requires the success fields named above.
- The locked SDK's `Client.callTool` validates present structured content even when `isError` is true.

Design authority: #204 requires retaining structured error receipts; #310 establishes listing/formatter agreement and preservation of success composition/wrapper semantics. Neither closed ticket owns this error-vs-success mismatch.

## The Fix

Make the published output schema admit the structured error shape the existing formatter actually emits alongside its declared success shape. Discriminate the error branch by the error key; preserve strict required success fields, primitive wrappers and declared compositions. Keep the full error metadata and the formatter's error classification.

The shared schema-publication boundary owns this, rather than weakening each service or disabling client validation.

| Surface | Authority | Behavior | Failure | Docs | Evidence |
|---|---|---|---|---|---|
| Published MCP output schema | Existing formatter; #204/#310 | Actual success and structured object-error results validate | Malformed non-error successes still reject | Converter JSDoc | Actual generated schema + formatter + SDK client |
| Error envelope | Existing BaseServer contract | Retain isError and typed receipt/recovery fields | No metadata-to-message workaround | Existing formatter docs | Refusal and receipt controls |

Decision Record impact: none; aligns schema publication with the existing wire envelope.

## Reproduction

From the pinned Brain tree with its locked dependencies:
1. Generate `update_issue_body`'s actual output schema with `buildOutputZodSchema` and `toOpenApiJsonSchema`.
2. Instantiate the real SDK `Client`; stub only its transport-facing `request`. Answer `tools/list` with that schema and call `client.listTools()` to cache its validator.
3. Format a `BODY_CHANGED` object containing `currentBodySha256` through the actual `BaseServer#formatToolResult`; answer `tools/call` with that envelope. `client.callTool()` throws -32602.
4. Repeat with incumbent `FOREIGN_BODY_FORBIDDEN`: same failure. Repeat with a complete success object: passes.

This was verified in memory using exact-source method extraction; no server, live mutation or formatter/schema substitute was used.

## Acceptance Criteria

- [ ] The generated listing, actual formatter and real SDK listTools→callTool path return both refusal controls with `isError: true` and their structured metadata intact.
- [ ] A complete body-update success still passes; a malformed non-error success still fails schema validation.
- [ ] Existing untyped/composed/primitive-wrapper output controls remain valid, including a structured receipt error from the drag-tool contract.
- [ ] Tests exercise the converter→formatter→client boundary with stubbed transport; no GitHub mutation or shared infrastructure is required.

## Out of Scope

Service policy changes, retry/merge logic, removing structured error receipts, bypassing SDK validation, and #1493's supplied-token validation/description repair.

## Avoided Traps

Do not delete structuredContent on errors, reopen the closed wrapper/nullability tickets, or turn the success schema into an unconstrained object. Formatter-only assertions cannot prove client delivery.

Related: #1493, #204, #310, #487.

unowned-rationale: Ready for a contributor; the public source coordinates, failure, controls and owning boundary are identified. Maintainers provide review.

Origin Session ID: f7ab949d-f37d-4a58-b896-146606ec4e3c
Retrieval Hint: "MCP structured errors SDK outputSchema success-only -32602 BODY_CHANGED FOREIGN_BODY_FORBIDDEN"

Live latest-open + closed sweep: latest20created-descending open Brainissues and closedstructuredContent/outputSchema searches read at20:25Z; #204/#310/#487 scope and traps read, no matching owner. Open#42 is deployment-acceptance CLI input parsing, not this result validation boundary. Recent30all-state A2A messages show no competing claim. Three MCproblem-noun searches returned no applicable prior decision; sixownBrainassignments are unrelated. Structuremap confirms existingserver/validation owners; no newmodule placement.

## Timeline

- 2026-11-05T20:24:50Z @neo-gpt-euclid added the `bug` label
- 2026-11-05T20:24:50Z @neo-gpt-euclid added the `good first issue` label
- 2026-11-05T20:24:50Z @neo-gpt-euclid added the `ai` label
- 2026-11-05T20:24:50Z @neo-gpt-euclid added the `agent-os` label
- 2026-11-05T20:25:24Z @neo-gpt-euclid cross-referenced by PR #1493
- 2026-11-06T10:16:11Z @github-actions cross-referenced by #42
- 2026-11-07T19:01:36Z @neo-gemini-julia cross-referenced by PR #20386
### @anasdahani220 - 2026-11-07T19:32:31Z

I'll take this. @neo-gemini-julia suggested it to me on neomjs/neo#20386. Plan: extend buildOutputZodSchema in ai/mcp/validation/openApiValidator.mjs so a structured error receipt (isError: true) is accepted alongside the declared success schema, and cover it with a stubbed-transport unit spec (success result, error receipt, and an invalid result that is still rejected).


### @github-actions - 2026-11-07T19:32:42Z

Thanks for claiming this, @anasdahani220, and welcome! You're first in line, so you can start now.

A maintainer confirms the assignment when one is online, and that can take a while. You don't need to wait for it.

<sub>An automatic reply. A maintainer follows up. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@anasdahani220 -->

- 2026-11-08T16:38:43Z @anasdahani220 referenced in commit `e2af138` - "fix(mcp): admit structured error receipts in object output schemas (#1495)"
- 2026-11-08T16:42:58Z @anasdahani220 referenced in commit `767127d` - "fix(mcp): admit structured error receipts in object output schemas (#1495)"
- 2026-11-08T16:44:34Z @anasdahani220 cross-referenced by PR #1517
- 2026-11-08T20:55:37Z @tobiu referenced in commit `50843e1` - "fix(mcp): admit structured error receipts in object output schemas (#1495) (#1517)"
- 2026-11-08T20:55:37Z @tobiu closed this issue

