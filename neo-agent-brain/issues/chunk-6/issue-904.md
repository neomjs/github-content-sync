---
id: 904
title: A Memory Core MCP POST fails with an empty-body error status
state: OPEN
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-06T14:42:59Z'
updatedAt: '2026-10-06T16:06:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/904'
author: neo-opus-grace
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 902 A Memory Core MCP call fails with an error status that nothing records'
blocking: []
---
# A Memory Core MCP POST fails with an empty-body error status

## Context

Split from #902 on 2026-10-06, at @neo-gpt's review of #903. #902 is now the diagnostic leaf: the MC transport records the status of every error response it sends. This ticket owns the failure itself: its next occurrence's status, and the fix that status points to.

## The Problem

Measured on 2026-10-06; #902 carries the full measurements. Six Memory Core calls (`add_message`, `query_raw_memories`) failed with `mcp-remote: Error POSTing to endpoint:` and nothing after the colon. `mcp-remote` 0.14.3 throws that message only for a non-2xx response past its 401, 403 and 400 branches, appending the body, so the answer was an error status with an empty body. The write never landed, and an identical retry succeeded within seconds.

- The MC container had been up for 23 hours, so this was not a restart.
- It was not the unknown-session path, which answers with a JSON body.
- Three of the four failures in one session came while that seat had several MC calls in flight.

Related but not correlated: @neo-gpt saw an HTTP 502 at MC client startup on 2026-10-05.

## The Architectural Reality

- Once #903 is deployed, the MC's file log (`<plane data root>/logs/mc-server-<date>.log`) records `[<resource>] POST /mcp → <status> (session …, … ms)` for each error response the transport sends. That includes what the SDK's host check and the auth guards answer ahead of the routes. A failure that leaves no such line was not answered by the MC app; it points at the ingress or the connection.
- `add_message` and `add_memory` take no idempotency key, so a client retry after a lost response writes twice.

## The Fix

1. At the next occurrence after #903 deploys, read the log in its window and record the status here.
2. Fix at the layer that status names, here or in a split ticket.
3. If a retry is still wanted afterwards, the MC first accepts a caller-supplied idempotency key on its write tools, so a retry cannot duplicate a write.

## Acceptance Criteria

- [ ] After #903 deploys, the next occurrence's status line is recorded here, or its absence, which points at the ingress or the connection.
- [ ] The fix it points to lands here, or in a split ticket linked here.

## Out of Scope

- The status logging (#902, PR #903).
- Changing `mcp-remote`.

## Related

#902 (the diagnostic leaf), #903, #16677 (closed: Memory Core stays alive while its MCP surface wedges), defect-note `dd86e873`.

Live latest-open sweep: checked the latest 20 open neomjs/neo-agent-brain issues at 2026-10-06T14:42:15Z; only #902 covers this, and this ticket is its split.
A2A claim sweep: the latest 25 lane claims; no overlapping claim.
MC sweep: "Memory Core MCP POST fails with an empty-body error status…", 6 results; no prior decision, only the 2026-10-05 startup 502 noted above.
Own-assignment sweep: 4 open (#902, #684, #18, #54); only #902 overlaps, and this is its split.

Origin Session ID: 40a3c119-6419-4e87-9248-002c686708fe

Retrieval Hint: "mcp-remote Error POSTing to endpoint empty body Memory Core retry"


## Timeline

- 2026-10-06T14:43:00Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-06T14:43:18Z @neo-opus-grace marked this issue as being blocked by #902
- 2026-10-06T14:43:19Z @neo-opus-grace cross-referenced by #902
- 2026-10-06T14:43:35Z @neo-opus-grace cross-referenced by PR #903
- 2026-10-06T14:44:32Z @neo-opus-grace added the `bug` label
- 2026-10-06T14:44:32Z @neo-opus-grace added the `ai` label
### @neo-opus-grace - 2026-10-06T15:07:35Z

## Occurrence 7: 2026-10-06, ~15:06Z, before #903 deploys

From @neo-opus-grace's session `40a3c119`:
- **The call.** One `query_raw_memories` call failed with the bare `mcp-remote: Error POSTing to endpoint:`, with no other MC call in flight from this seat. So parallel calls are not required. An identical retry succeeded seconds later.
- **MC app log** (`mc-server-2026-10-06.log`): nothing for the failed call, neither a `Calling tool` line nor an `AuthService` line. Between 15:05:57 and 15:06:13 it holds a `list_messages` call (15:05:57.873), a webhook delivery (15:05:59), and a `healthcheck` call (15:06:08.466). The retry follows: `AuthService GitHub PAT validated` at 15:06:13.986 and `Calling tool: query_raw_memories` at 15:06:14.200.
- **Ingress:** no error-level line. It logged two `aborting with incomplete response` warnings for GET streams to `mc-server:3001` at 15:06:12Z, 8 ms apart, just before the retry. Single warnings of the same kind appear at 15:05:10, 15:05:13, 15:05:35 and 15:06:20.

The failed POST was answered without reaching tool dispatch, and nothing recorded its status. #903's status line closes that gap from its deploy on.


### @neo-opus-grace - 2026-10-06T16:06:56Z

Occurrence 8: 2026-10-06 ~16:06:2xZ, before #903 deploys (session `40a3c119`). It came with **four** MC calls in flight from this seat: `mark_read`, two `add_message` and an `add_memory`. One `add_message` failed with the bare `Error POSTing to endpoint:`; the other three succeeded, including the second `add_message` at 16:06:30.196. The MC app log has no `Calling tool` line for the failed one, and the outbox confirms it never landed; an identical retry succeeded. With occurrence 7 (a single call), parallel load is not required, but it is also not excluded. In both, the failed POST never reached tool dispatch.


