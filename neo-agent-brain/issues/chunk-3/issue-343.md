---
id: 343
title: Neural Link can settle a call with another client's reply
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-gpt
createdAt: '2026-09-09T01:47:45Z'
updatedAt: '2026-09-12T11:13:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/343'
author: neo-gpt
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
closedAt: '2026-09-12T11:13:51Z'
---
# Neural Link can settle a call with another client's reply

## Context

While investigating neomjs/neo#18516, whole-file browser runs intermittently returned `undefined` from an executor whose source returns an object on every terminal. That symptom led to a separate transport audit; this ticket does not claim that every observed undefined result has this cause.

A deterministic, network-free probe of the exact AST-extracted `ConnectionService.call`, `handleAppMessage` and `resolveRequest` methods at `c7be03eaebf21f42fb7f48b9e0bfed9fcad2271e` demonstrates the correlation defect independently.

## The Problem

Two fresh clients each send request ID `1`. Client A targets app A; client B targets app B. Feeding app B's response to both clients settles A's pending promise with B's result and deletes A's pending entry. A's later correct reply cannot repair it.

Measured controls:

```text
A request: target=app-A, method=owner-method, id=1
B request: target=app-B, method=peer-method,  id=1
B reply delivered to A: {id:1,result:{result:"reply-to-peer-B"}}
A observed result: {result:"reply-to-peer-B"}
A remaining pending entries: 0

Correct-owner control: returns correct-owner-A
Non-colliding-ID control: foreign id=1 leaves pending id=11 intact
```

The probe executed the actual source methods with fake outbound sockets, not a replacement correlation algorithm. Live browser observations motivated the audit but are not the sole proof.

## The Architectural Reality

- `ai/services/neural-link/ConnectionService.mjs:233,309`: every client starts `msgId = 0` and generates `++this.msgId`.
- `call():334` records resolve/reject/timeout under that ID without the expected app session.
- `handleAppMessage():624–629` receives the app session but drops it before `resolveRequest()`.
- `resolveRequest():773–785` matches only `message.id` and immediately settles/deletes the pending call.
- `ai/mcp/server/neural-link/Bridge.mjs:363–378` broadcasts app messages to all connected agents. The clients therefore need requester-safe correlation even when they explicitly targeted different apps.

This is the Neural Link transport owner's responsibility. Explicit target selection, write locking, and correct component selectors cannot compensate for accepting the wrong reply.

## The Fix

Keep request/reply correlation inside the existing ConnectionService/Bridge boundary. Ensure concurrent clients cannot produce colliding effective request identities, and bind each pending call to its expected app session before consuming a response. Preserve normal JSON-RPC result/error handling, notification broadcast, timeouts and teardown. Verify the chosen ID representation against the Engine client's actual echo path rather than assuming numeric-only or string-only behavior.

Extend the existing `test/playwright/unit/ai/services/neural-link/ConnectionService.spec.mjs` and verify the relevant Bridge tests. Add both retained specs to the existing `.github/workflows/brain-unit.yml` smoke execution list so the new regression controls execute rather than merely appear in collection. No new service or configuration surface is needed.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Outbound RPC identity | ConnectionService.call + Engine client echo | Requests from different client instances remain distinguishable | No numeric counter-only fallback across clients | Method JSDoc | Two concurrent clients, same and different target apps |
| Pending reply ownership | handleAppMessage / resolveRequest | Consume only the response belonging to the pending request and expected app | Ignore unrelated/late replies without deleting another pending call | Method JSDoc | Foreign success/error then correct-owner response |
| Regression execution | `.github/workflows/brain-unit.yml` named smoke list | Execute ConnectionService and Bridge specs on PRs | Do not substitute `--list` for execution | Workflow step | Hosted unit run lists and executes both files |
| Existing transport outcomes | Bridge app broadcast; call timeout/error paths | Preserve notifications, normal results/errors and cleanup | Existing explicit failure semantics | Existing source contracts | Positive reply, timeout, disconnect and notification controls |

## Acceptance Criteria

- [ ] Two independent clients sharing a Bridge cannot settle each other's calls, including calls to the same app and to different apps.
- [ ] A foreign-app response, including an error, does not settle or delete another pending request; the later correct response still succeeds.
- [ ] Positive success/error replies, late/duplicate replies, notifications, timeout and disconnect cleanup retain their documented behavior.
- [ ] A real Bridge/Engine-client round trip verifies the chosen ID representation and concurrent-caller isolation; source-only tests are not the complete receipt.
- [ ] Re-run the affected DockLayouts caller with the repaired transport and distinguish any remaining product failure from transport correction. No claim that this alone fixes the dock preview bug.

## Decision Record impact

None: restore intended request/reply ownership without changing authorization, write-lock semantics, or the public tool inventory.

## Out of Scope

Dock preview/embodiment repair, KB ingestion, repository migration, new credentials, or a redesign of notification broadcast.

## Avoided Traps

- Matching the app alone still permits two clients' counters to collide on the same app.
- Giving IDs more digits or a fixed offset only delays collisions.
- A malformed-value check after consuming the wrong reply does not restore ownership.
- Do not turn an intermittent runtime symptom into an unqualified claim about all prior test results.

## Related

Related: #143; neomjs/neo#18516

Creation freshness: latest 20 open Brain issues and latest 30 all-state A2A messages checked immediately before creation on 2026-09-09; no equivalent ticket or competing claim. All-state Neural Link/correlation search and #143 body read found the broad coordination authority, not this repair. MC query for unexpected undefined RPC replies returned no usable prior decision. Own-assignment sweep: six open issues, none cover request/reply correlation. Structure map ran: existing `ai/services/neural-link/ConnectionService.mjs` and neighboring services own the implementation; existing ConnectionService/Bridge specs own the tests.

Origin Session ID: 4f9b7bbe-9dbc-4d36-a167-38fe0d288113

Retrieval Hint: ConnectionService pendingRequests numeric msgId foreign app reply correlation; source c7be03eaebf21f42fb7f48b9e0bfed9fcad2271e.


## Timeline

- 2026-09-09T01:47:45Z @neo-gpt assigned to @neo-gpt
- 2026-09-09T01:47:47Z @neo-gpt added the `bug` label
- 2026-09-09T01:47:47Z @neo-gpt added the `ai` label
- 2026-09-09T01:47:47Z @neo-gpt added the `testing` label
- 2026-09-09T02:02:04Z @neo-gpt-emmy cross-referenced by PR #18519
- 2026-09-09T02:05:01Z @neo-gpt cross-referenced by PR #344
- 2026-09-12T11:13:51Z @tobiu referenced in commit `76313d6` - "Merge pull request #344 from neomjs/codex/343-neural-reply-ownership

fix(neural-link): bind replies to their requester and app (#343)"
- 2026-09-12T11:13:52Z @tobiu closed this issue

