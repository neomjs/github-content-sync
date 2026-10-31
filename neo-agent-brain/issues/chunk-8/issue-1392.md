---
id: 1392
title: A received bridge response disappears during generation retirement
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-31T01:16:03Z'
updatedAt: '2026-10-31T16:48:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1392'
author: neo-gpt-euclid
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
closedAt: '2026-10-31T16:48:59Z'
---
# A received bridge response disappears during generation retirement

## Context

A focused cross-family review trace found a response-settlement race on both current base `b9044a64fe4802ebe0b2dc6fd733d743e0247eba` and PR #1391 head `1ab190e4211840d509eca9693aa076082c18644e`. It is inherited; the session-resume repair does not introduce it.

## The Problem

After a valid response arrives, `wire().onmessage` removes its entry from `pending`, then queues local delivery in a promise microtask. If a404 retires that generation first, retirement finds no pending entry to answer and the queued delivery fails its current-generation check. The local connection stays open with neither a result nor an error for the request.

Observed with inert transport-interface callbacks, not a production outage or real-network timing measurement: request2 forwarded once, zero local replies for2, local closed count0, on both exact revisions.

## The Architectural Reality

Owner: [stdioToStreamableHttp.mjs](https://github.com/neomjs/neo-agent-brain/blob/1ab190e4211840d509eca9693aa076082c18644e/ai/mcp/client/stdioToStreamableHttp.mjs#L285-L312), specifically response receipt, `pending` ownership, deferred local delivery and `retire`. The existing [spec helpers](https://github.com/neomjs/neo-agent-brain/blob/1ab190e4211840d509eca9693aa076082c18644e/test/playwright/unit/ai/mcp/client/StdioToStreamableHttp.spec.mjs) provide initialized inert transports and `flush`.

Design authority: the owning module promises that lost requests receive an error with an unknown outcome and are never replayed. Its current retirement path already owns settling generation-bound pending requests; the received-but-not-delivered gap escapes both settlement paths.

## The Fix

Keep one settlement owner until a received response is committed to local delivery or retirement answers it. Preserve stale-generation and local-close fences, exactly-once local replies and the no-replay rule. Repair this inside the existing bridge and its spec; no new recovery loop, API or session-identity scheme.

Reproducer after `startInitializedBridge()`:
```js
local.onmessage({jsonrpc:'2.0', id:2, method:'tools/list', params:{}});
await flush();
remote.onmessage({jsonrpc:'2.0', id:2, result:{tools:[]}});
remote.onerror(Object.assign(new Error('inert404'), {code:404}));
await flush();
// Currently zero replies for2 while local.closed is0.
```

## Contract Ledger

| Surface | Authority | Behaviour | Fallback | Evidence |
|---|---|---|---|---|
| Local reply settlement | Existing bridge pending ownership and no-replay contract | A current response followed by retirement yields exactly one local result or explicit failure | A late response from an already-retired generation stays ignored; local shutdown remains terminal | Response→retire→delivery and retire→late-response controls, including queued microtask ordering |

Decision Record impact: aligned with the existing recovery contract; no identity or configuration change.

## Acceptance Criteria

- [ ] Response→404 retirement before deferred delivery yields exactly one local result or explicit failure, with no silent pending request.
- [ ] Retirement→late response and local-close controls preserve their fences and never duplicate replies.
- [ ] No failed or uncertain write is replayed; later requests retain the existing recovery behaviour.
- [ ] The owning bridge spec passes, with the new case failing against the unchanged base.

## Out of Scope

Session-ID continuity across server restarts, ingress502 causes, server orphan cleanup and a new retry/transport abstraction.

## Avoided Traps

Deleting pending ownership before either settlement commits creates the gap. Unconditionally forwarding every stale callback would break the opposite fence. Do not solve response settlement by replaying a request.

Related: #1041, #1062, #1390, #1391.

unowned-rationale: a verified inherited defect discovered during review, ready for one bounded bridge repair; the reviewer retains Stage#346 and has not claimed implementation or assigned a peer.

Creation checks: exact-module/all-state bridge searches found no open equivalent; own six assigned Brain issues have different surfaces. Memory sweep on dropped reply/pending retirement returned historical transport-continuity work, no matching decision. Structure map confirms `ai/mcp/client`; no new module. Live latest20 open and recent all-state A2A rechecked immediately before filing.

Origin Session ID: d56648bd-aa2e-40d4-a6c2-4cf39b36af5a
Retrieval Hint: bridge received response pending deleted retirement deferred local delivery


## Timeline

- 2026-10-31T01:16:05Z @neo-gpt-euclid added the `bug` label
- 2026-10-31T01:16:05Z @neo-gpt-euclid added the `ai` label
- 2026-10-31T01:16:05Z @neo-gpt-euclid added the `agent-os` label
- 2026-10-31T01:20:05Z @neo-gpt-euclid cross-referenced by PR #1391
- 2026-10-31T01:25:26Z @neo-fable-mnemosyne cross-referenced by #1390
- 2026-10-31T14:39:31Z @neo-opus-vega cross-referenced by #1395
- 2026-10-31T15:56:33Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-31T16:01:19Z @neo-gemini-maryam cross-referenced by PR #1398
- 2026-10-31T16:48:58Z @tobiu referenced in commit `10c96bd` - "fix(mcp): preserve pending settlement across deferred local response delivery (#1392) (#1398)"
- 2026-10-31T16:49:00Z @tobiu closed this issue

