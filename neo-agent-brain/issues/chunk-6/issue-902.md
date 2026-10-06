---
id: 902
title: A Memory Core MCP call fails with an error status that nothing records
state: OPEN
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-06T13:59:35Z'
updatedAt: '2026-10-06T14:43:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/902'
author: neo-opus-grace
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
blocking:
  - '[ ] 904 A Memory Core MCP POST fails with an empty-body error status'
---
# A Memory Core MCP call fails with an error status that nothing records

## Context

Defect-note `dd86e873`, escalated by the operator on 2026-10-06. He asked whether to add a "retry x times" mechanism, noting that Memory Core writes land in a WAL-drained jsonl, so a failure only makes sense if the MC container restarts.

## The Problem

Six occurrences on 2026-10-06: four from @neo-opus-grace's session `40a3c119` (~11:24, 11:41, 13:46, 13:52Z) and two from @neo-opus-ada's that morning. `add_message` and `query_raw_memories` fail with `mcp-remote: Error POSTing to endpoint:` and nothing after the colon. The write never lands (an outbox read confirmed it each time), and an identical retry succeeds within seconds.

Measured:
- **No restart.** `neo-local-agent-os-mc-server-1` has been up 23 h and healthy, and the ingress (Caddy) 7 days.
- **An error status with an empty body.** The seats reach the MC through `npx mcp-remote http://127.0.0.1:3102/mc/mcp` (0.14.3). It throws this message only for a non-2xx response that passes its 401, 403 and 400 branches, appending the response body. So the HTTP side answered an error status with an empty body, before the write path.
- **Not the unknown-session path.** `TransportService`'s unknown-session path answers 404 with a JSON body (`{"error":"Session not found"}`), and sessions never expire there.
- **Nothing records the status.** The MC server logs no requests, and the ingress has no access log. Between 13:46 and 13:53Z its log holds only `GET /mcp … reading: context canceled` warnings, client-cancelled SSE streams, clustered at each failure.
- **Mostly parallel calls.** Three of the four failures in session `40a3c119` came while that seat had several MC calls in flight in parallel.

## The Architectural Reality

- `ai/mcp/server/shared/services/TransportService.mjs`: `app.all('/mcp')` resolves the session and delegates to the SDK's `StreamableHTTPServerTransport.handleRequest`. `AuthService` installs the bearer guard ahead of it.
- The client is a third-party stdio bridge (`mcp-remote`). Retries can't be added there without forking or wrapping it.
- `add_message` and `add_memory` take no idempotency key. If a retried POST's first attempt reached the handler and only the response was lost, the retry writes twice.

## The Fix

1. **Make the next failure name itself.** `TransportService` logs each error response its HTTP server sends: method, path, status, whether a session id was present, and duration, never headers, query or body. It listens on the server rather than as Express middleware, so it also sees what the SDK's host check and the auth guards answer ahead of the routes. The ingress needs no change: Caddy already logs upstream failures at error level, and none appeared in the failure windows, so the status originates in the MC app.
2. **Fix at the layer the status names.** Moved to #904, which owns the failure itself. A client retry is not the first fix, for the two reasons above.

## Acceptance Criteria

- [ ] A non-2xx from the MC's HTTP transport produces one log line with method, status, session presence and duration. A spec drives a 404 and a 5xx through it.
- ~~The ingress records status codes for the MC route, without headers.~~ Retired 2026-10-06: Caddy already logs upstream failures at error level, and none appeared in the failure windows (see The Fix, step 1).
- ~~The next occurrence's status is recorded here, with the fix it points to.~~ Moved 2026-10-06 to #904, which stays open past this leaf's merge (@neo-gpt's review of #903).

## Out of Scope

- Changing `mcp-remote`.
- KB and GitHub-workflow traffic. The same transport serves them, so the logging covers them, but no failure there is measured.

## Related

#16677 (closed: Memory Core stays alive while its MCP surface wedges), #561 (closed: the wake-subscription tool path), defect-note `dd86e873`.

Live latest-open sweep: no equivalent among open neo-agent-brain issues (searches for `mcp-remote`, `Error POSTing to endpoint` and `Session not found` return only unrelated or closed issues).
A2A claim sweep: no overlapping claim. @neo-opus-ada reported the same symptom (MESSAGE:286441a1).
MC sweep: superseded by the live measurements above; #16677 is the nearest prior art.

Origin Session ID: 40a3c119-6419-4e87-9248-002c686708fe



## Timeline

- 2026-10-06T14:00:09Z @neo-opus-grace added the `bug` label
- 2026-10-06T14:00:09Z @neo-opus-grace added the `ai` label
- 2026-10-06T14:16:15Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-06T14:32:26Z @neo-opus-grace cross-referenced by PR #903
- 2026-10-06T14:43:00Z @neo-opus-grace cross-referenced by #904
- 2026-10-06T14:43:18Z @neo-opus-grace marked this issue as blocking #904
- 2026-10-06T14:57:28Z @neo-opus-grace referenced in commit `f02035c` - "fix(mcp): the status log keeps a request target's path alone, never an absolute target's authority (#902)

An absolute-form target (POST http://user:pass@host/mcp) arrives whole in req.url, so stripping the query still logged its userinfo. The listener parses the target and logs its pathname; a target that does not parse reads as such, never raw. Review RA-1 on #903."

