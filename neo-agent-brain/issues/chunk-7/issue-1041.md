---
id: 1041
title: The MCP bridge kills the server on recoverable remote-transport errors
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-15T09:37:03Z'
updatedAt: '2026-10-15T12:33:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1041'
author: neo-opus-vega
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
closedAt: '2026-10-15T12:33:00Z'
---
# The MCP bridge kills the server on recoverable remote-transport errors

## Context

The operator asked for an investigation on 2026-10-15 after one Vega session lost its memory-core server three times (≈08:55Z, ≈09:22Z, ≈09:36Z). Each time, an in-flight tool call returned `Connection closed`.

- **08:55Z:** the seat's launching Fleet Manager process had already been replaced, so Desktop's relaunch was refused (`issuer-unavailable`, see #980 / neomjs/neo-agent-institution#674). MC stayed dead until a seat restart.
- **09:22Z and 09:36Z:** the issuer was alive, so the relaunch succeeded, and only the in-flight call was lost.

**Observed:** the Claude Desktop MCP logs on the operator's host, which the Claude seats share, show `Neo MCP bridge transport failed.` 124× for memory-core and 39× for knowledge-base since 2026-10-07, the first day of Fleet-managed seats. The peak was 34 on 10-10. github-workflow and neural-link show 0, because they run as local stdio servers and never pass through this bridge.

**Unknown:** which underlying error tripped each drop. The bridge discards it (see below), so these counts cannot be split by cause.

## The Problem

The bridge treats every remote-transport `onerror` as terminal. The MCP SDK does not. In `@modelcontextprotocol/sdk` 1.32.1 (`dist/esm/client/streamableHttp.js`), when a resumable SSE stream drops, the SDK first raises `onerror(new Error('SSE stream disconnected: …'))` (`:247`) and then schedules its own reconnection (`:256`). Only an exhausted retry budget (`:157`) ends the stream.

By then the bridge has already closed both transports. A recoverable hiccup becomes a dead MCP server: the call in flight fails, and Desktop must relaunch the server. When the issuer is orphaned, it cannot relaunch at all. Each relaunch also opens a new Memory Core session, so one conversation's turns split across session ids. On 2026-10-15 Vega's three saves from one conversation landed in `eba5a2ca`, `b7c87928` and `2efbd8f3`, with the 09:22Z and 09:36Z drops between them.

## The Architectural Reality

- `ai/mcp/client/stdioToStreamableHttp.mjs` bridges Desktop's stdio to the plane's Streamable HTTP endpoint for memory-core and knowledge-base.
- `:185–186` assigns `localTransport.onerror = fail; remoteTransport.onerror = fail;`.
- `:131–141`: `fail` takes no argument. It reports a fresh `new Error(RUNTIME_FAILURE_MESSAGE)` (`:10`) and closes the bridge, so the cause never reaches the log.
- `send()` failures reach the same `fail` through `.catch(fail)` (`:153`, `:163`, `:183`). One failed request therefore closes the whole server.
- Tests live in `test/playwright/unit/ai/mcp/client/StdioToStreamableHttp.spec.mjs`. The last bridge change was #1034 (`e219267e`, decline discovery before initialization).

## The Fix

1. **Carry the cause.** Each report names a cause from a fixed, locally-authored vocabulary: `http-<status>`, `sse-disconnect`, `sse-reconnect-failed`, `sse-reconnect-exhausted`, `unexpected-content-type`, `network`, `timeout` or `unknown`. The bridge's stderr boundary admits no SDK or remote text, so it logs no error message, error object or header.
2. **Classify remote errors.** The plane's `TransportService` answers POSTs with SSE and keeps no event store, so a POST stream cannot resume, and only the GET stream can be open while no request is pending. After initialization, the bridge logs and survives two cases: a stream drop with no request pending, and a failed reconnect attempt that the SDK will retry. It closes on terminal errors: an exhausted reconnection, a session or auth refusal, a drop while a request is pending, or a local transport error.
3. **Contain request failures.** After initialization, a failed `send()` of a request answers that id with a JSON-RPC error, and the bridge stays up. The exceptions stay terminal: 401, 403, 404, an unexpected content type, a failed send of a non-request, and any failure before initialization.

## Acceptance Criteria

- [ ] AC-1: Every bridge report names its cause from the fixed vocabulary. Unit tests assert that no error text and no bearer value appear in the output.
- [ ] AC-2: With no request pending, `onerror(SSE stream disconnected)` and a failed reconnect attempt leave the bridge open, and later traffic still flows. With a request pending, the same drop closes the bridge and names its cause.
- [ ] AC-3: An exhausted reconnection, and a session or auth refusal, still close the bridge, with each cause logged. One test per case.
- [ ] AC-4: After initialization, a non-terminal `send()` failure on a request returns a JSON-RPC error for that id, and later requests still pass. Tested.
- ~~AC-5 (post-install): A week of Desktop MCP logs from seats on the fixed bundle shows a cause on every failure line. A comment on this ticket tallies the remaining terminal failures by cause.~~ Struck on 2026-10-15: it is an observation, not a deliverable. Out of Scope already routes any server-side cause that AC-1 surfaces to its own ticket.

## Out of Scope

- **Issuer succession after an FM restart:** that is #980 / neomjs/neo-agent-institution#674.
- **Server-side or ingress causes:** if AC-1's causes point there, they get their own ticket.
- **Local stdio servers.**

## Avoided Traps

- **A retry loop inside the bridge** on top of the SDK's own reconnection: two backoffs would fight over one stream.
- **Tolerating every error:** a dead remote would then look alive.
- **Logging the raw error object:** request context can carry the bearer header.

## Related

- #1034: the last change to this file.
- #980 and neomjs/neo-agent-institution#674: why a drop is fatal after an FM restart.
- #909: Desktop MCPs launched through Fleet admission.

**Decision Record impact:** none.

**Sweeps (2026-10-15T09:36Z):**
- Live latest-open: the latest 20 open neo-agent-brain issues contain no equivalent. #1023 concerns the Neural Link agent socket, not this transport.
- A2A: no in-flight claim on this scope in the last 30 messages.
- MC: the query "Neo MCP bridge transport failed Connection closed memory-core remote bridge drop" finds prior art (an MC attachment that died at the 10-10 FM swap) but no diagnosis.
- Own assignments: none on this surface.
- Structure map: `ai/mcp/client` owns the file, and no new file is needed.

Owner: Vega. Self-assigned at 09:47Z after Emmy, who made the last bridge change, declined because she keeps Stage #39.

Retrieval Hint: "MCP bridge transport failed recoverable SSE disconnect onerror"

Origin Session ID: b7c87928-29fb-4f04-ae25-64fdaff70416

Vega (Opus 5.5, Claude Code)





## Timeline

- 2026-10-15T09:37:05Z @neo-opus-vega added the `bug` label
- 2026-10-15T09:37:05Z @neo-opus-vega added the `ai` label
- 2026-10-15T09:37:05Z @neo-opus-vega added the `agent-os` label
- 2026-10-15T09:47:19Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-15T09:56:04Z @neo-opus-vega cross-referenced by PR #1043
- 2026-10-15T11:09:24Z @neo-opus-vega cross-referenced by #1046
- 2026-10-15T12:05:19Z @neo-opus-vega referenced in commit `6ac4a1f` - "feat(mcp): recover in the SDK's report-then-reject order and retire cancelled requests (#1041)

The SDK hands an error to onerror before it rejects the send that raised
it, and before it retries the stream that lost it; the first initial-GET
failure even reaches onerror twice as one object. Classifying inside
onerror therefore closed the bridge on a POST 502 before its rejection
could answer the request, and on a GET 502 before the SDK's retry.

onerror now closes only on what is terminal by itself (any failure before
initialization, exhausted reconnection, a stream drop with a request
pending) and reports anything else as survived one task later, unless a
send rejection claimed the error first or the bridge closed meanwhile.
A delivered notifications/cancelled retires its request id, so a
cancelled request no longer turns a later idle-stream drop terminal."
- 2026-10-15T12:06:40Z @neo-opus-vega referenced in commit `01b69c0` - "test(mcp): pin network reconnects and auth/protocol refusals in the SDK order (#1041)

The reconnect control now covers a network failure beside the 502, and 401, 403 and an unexpected content type on a request stay terminal when the SDK reports to onerror before rejecting."
- 2026-10-15T12:22:03Z @neo-opus-vega referenced in commit `a2ddfe1` - "feat(mcp): a stream GET's session or auth refusal ends the session and a closed bridge reports no survival (#1041)

A 401, 403 or 404 that reaches onerror from the stream GET has no send rejection to claim it, so onerror now treats every session-ending code as terminal. A send that the bridge's own close aborts no longer reports itself as survived."
- 2026-10-15T12:32:59Z @tobiu referenced in commit `1d611a5` - "feat(mcp): the bridge survives recoverable transport errors and names their cause (#1041) (#1043)

* feat(mcp): the bridge survives recoverable transport errors and names their cause (#1041)

The stdio-to-Streamable-HTTP bridge closed on every remote onerror, but the
MCP SDK raises onerror for a dropped SSE stream it then reconnects itself.
Each such hiccup killed the memory-core or knowledge-base server; after an
FM restart the relaunch was refused, and the log never said why.

After initialization the bridge now survives what costs no message: a
stream drop with no request pending (only the GET stream can be open then;
the plane keeps no event store, so POST streams cannot resume) and a failed
reconnect attempt the SDK retries. A request whose delivery fails is
answered locally with a JSON-RPC error. Pre-initialization failures,
401/403/404, unexpected content types, exhausted reconnection and local
errors still close both sides once. Every report names a cause from a fixed
vocabulary, so no SDK or remote text crosses the stderr boundary.

* feat(mcp): recover in the SDK's report-then-reject order and retire cancelled requests (#1041)

The SDK hands an error to onerror before it rejects the send that raised
it, and before it retries the stream that lost it; the first initial-GET
failure even reaches onerror twice as one object. Classifying inside
onerror therefore closed the bridge on a POST 502 before its rejection
could answer the request, and on a GET 502 before the SDK's retry.

onerror now closes only on what is terminal by itself (any failure before
initialization, exhausted reconnection, a stream drop with a request
pending) and reports anything else as survived one task later, unless a
send rejection claimed the error first or the bridge closed meanwhile.
A delivered notifications/cancelled retires its request id, so a
cancelled request no longer turns a later idle-stream drop terminal.

* test(mcp): pin network reconnects and auth/protocol refusals in the SDK order (#1041)

The reconnect control now covers a network failure beside the 502, and 401, 403 and an unexpected content type on a request stay terminal when the SDK reports to onerror before rejecting.

* feat(mcp): a stream GET's session or auth refusal ends the session and a closed bridge reports no survival (#1041)

A 401, 403 or 404 that reaches onerror from the stream GET has no send rejection to claim it, so onerror now treats every session-ending code as terminal. A send that the bridge's own close aborts no longer reports itself as survived."
- 2026-10-15T12:33:00Z @tobiu closed this issue

