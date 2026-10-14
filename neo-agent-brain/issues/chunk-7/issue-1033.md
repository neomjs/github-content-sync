---
id: 1033
title: MCP bridge exits on discovery before initialization
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-14T20:46:48Z'
updatedAt: '2026-10-14T21:23:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1033'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-14T21:23:59Z'
---
# MCP bridge exits on discovery before initialization

## Context
A stdio MCP client can probe the optional `server/discover` method before sending `initialize`. The stdio-to-Streamable-HTTP bridge currently closes the client connection on this sequence instead of allowing initialization to follow an unsupported-method response.

## The Problem
Portable reproducer, using an inert bearer and a fixture HTTP MCP server:
1. Send a JSON-RPC request for `server/discover` before initialization.
2. A direct stdio server returns MethodNotFound (`-32601`) and remains available for `initialize`.
3. The bridge forwards the probe to an HTTP endpoint that requires initialization first; the transport error closes both sides, so initialization cannot follow.

Initialize-first probes do not exercise this failure.

## The Architectural Reality
`ai/mcp/client/stdioToStreamableHttp.mjs#bridgeTransports` forwards local requests through the MCP SDK HTTP client and treats a transport error as terminal. The bridge does not implement server discovery. Its existing tests are in `test/playwright/unit/ai/mcp/client/StdioToStreamableHttp.spec.mjs`.

Design authority: the bridge's existing transport-forwarding contract and the standard negative JSON-RPC method response. Declining an unsupported discovery probe must not advertise discovery capability or weaken authentication.

## The Fix
Before initialization, answer an optional `server/discover` request locally with JSON-RPC MethodNotFound, preserving its id and the open connection. Continue forwarding initialize and subsequent traffic through the existing authenticated HTTP transport. Keep other error, cleanup and secret-safe diagnostic behavior unchanged. Use the existing bridge and spec; no vendor adapter or new module.

## Contract Ledger
| Surface | Authority | Behavior | Failure | Docs | Evidence |
|---|---|---|---|---|---|
| Pre-initialization discovery | `bridgeTransports` | Unsupported-method response; no upstream request or session claim | Local send errors use existing cleanup | Bridge JSDoc | Request-id and no-upstream controls |
| Initialized forwarding | Existing SDK transports | Preserve version/id forwarding and authentication | Existing bounded sanitized errors and one close | Existing JSDoc | Executable HTTP fixture and existing suite |

Decision Record impact: none.

## Acceptance Criteria
- [ ] Pre-initialization `server/discover` receives `-32601` with the same id, including id 0, without contacting the upstream or closing either transport.
- [ ] A following initialize/initialized/tools-list sequence succeeds through the executable and an HTTP fixture that rejects non-initialization traffic without a session.
- [ ] Post-initialization forwarding, failure cleanup and secret-safe diagnostics retain their current behavior.

## Post-Merge Validation
- [ ] Native client acceptance of real MC/KB calls remains with the existing seat-onboarding outcome.

Residual-Owner: #571

## Out of Scope
Implementing discovery, broader method emulation, SSE reconnection policy, credential changes, global configuration policy, or endpoint-wait timing.

## Avoided Traps
Counting initialize-first tests as discovery-handshake coverage; promoting a negative capability response into connection failure; printing remote response bodies or bearers.

Related: #571.
Ownership: Emmy.
Live latest-open, A2A, own-assignment, Memory Core, KB and source/history sweeps found no equivalent discovery-handshake leaf. The existing HTTP-error and endpoint-timing tickets cover different causes. Structure map confirms the existing client bridge/spec own the change.
Retrieval Hint: MCP server/discover before initialize bridge EOF.

Authored by Emmy (GPT-6, Codex Desktop).


## Timeline

- 2026-10-14T20:46:49Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-14T20:46:49Z @neo-gpt-emmy added the `bug` label
- 2026-10-14T20:46:50Z @neo-gpt-emmy added the `ai` label
- 2026-10-14T20:47:38Z @neo-gpt-emmy added parent issue #571
- 2026-10-14T20:54:07Z @neo-gpt-emmy cross-referenced by PR #1034
- 2026-10-14T21:01:20Z @neo-gpt-emmy cross-referenced by #571
- 2026-10-14T21:23:59Z @tobiu referenced in commit `e219267` - "fix(mcp): decline discovery before initialization (#1033) (#1034)"
- 2026-10-14T21:23:59Z @tobiu closed this issue

