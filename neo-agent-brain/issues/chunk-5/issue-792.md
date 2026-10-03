---
id: 792
title: The Fleet serves a seat's recent turn summaries
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-03T07:08:40Z'
updatedAt: '2026-10-03T16:59:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/792'
author: neo-fable
commentsCount: 0
parentIssue: 391
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 476 The Agent Detail''s Thought stream pane reads the seat''s recent turns'
closedAt: '2026-10-03T16:18:23Z'
---
# The Fleet serves a seat's recent turn summaries

## Context

The producer leaf for the Agent Detail's Thought stream pane (neomjs/neo-agent-institution#391, the tracker of the detail's live panes). The operator's observation on the installed Fleet Manager (2026-10-01) was four panes reading `not observed`; the Repository and Current lane panes are live since, and the thought stream names its missing producer on the pill. #741 gave the Memory Core's recency read a sharing policy, so a peer's public turn summaries are readable. Nothing in the Fleet serves them to the cockpit.

## The Problem

Measured on the canonical plane at `dev@804356b` (2026-10-03):

| Read | Answer |
| :--- | :--- |
| `query_recent_turns({agentIdentity: '@neo-opus-ada', memorySharing: 'team', detail: 'summary', limit: 3})` | three public turn summaries, newest first; `memorySharing: {policy: 'team', clamped: false}` |
| the same without `memorySharing` | zero turns (the default tenant contract) |

The cockpit reaches the Memory Core only through Fleet read verbs (`fleetMemories`, `fleetSessionMemories`, `fleetActivity`, …). No verb wraps the recency read: `grep -rn "queryRecentTurns\|query_recent_turns" ai/services/fleet` finds one comment in a seat-memory template and no source. With the full team entering the roster (#571), every seat's detail opens on a thought stream the plane can answer and the Fleet cannot pass on.

## The Architectural Reality

- `ai/services/fleet/fleetSessionMemoriesSource.mjs` — the sibling to lift: a viewer-bound source over one injected Memory Core operation, a typed fail-honest envelope (`capability {state, reason, capturedAt, detail}`), no Fleet synthesis, cache, ranking or permission simulation; a failure is `unavailable`, never a fabricated empty page.
- `ai/services/fleet/wireFleetSessionMemoriesSource.mjs` installs it on the bridge; `ai/services/fleet/devFleetServer.mjs:444` injects the operation through `callHistoryOperation` (the plane client in plane mode, the in-process tool service otherwise) and the viewer through `RequestContextService.getAgentIdentityNodeId()`.
- `ai/services/fleet/FleetControlBridge.mjs:723` — the verb passes the source's envelope through, or names an unwired source `unavailable`.
- `ai/services/fleet/fleetServerPolicy.mjs` — two tables carry every verb: the phase table (`fleetSessionMemories: 'awaiting-s5'`, :41) and the class table (`'read-observe'`, :93).
- `src/fleet/contract/wire.mjs:14` — `FLEET_WIRE_METHODS`, the vocabulary a client may send.
- `query_recent_turns` (#741): an explicit `memorySharing` widens to public summaries only — `team` or `legacy` with `projection: 'private'` or `detail: 'full'` is refused before any read, and a requested `team` over a `private` default is clamped and answers empty. The answer carries `memorySharing {policy, clamped}` and a `nextCursor {timestamp, id}`.

## The Fix

One read verb, `fleetRecentTurns`, in the sibling's shape:

1. `ai/services/fleet/fleetRecentTurnsSource.mjs` — `createFleetRecentTurnsSource({queryRecentTurns, resolveViewerIdentity, now})` → `readRecentTurns({agentIdentity, limit, before})`. It asks the operation for `{agentIdentity, memorySharing: 'team', detail: 'summary', limit, before}` and passes the rows through. The envelope: `{capability, viewer, target, page: {limit, before}, turns, count, nextCursor, memorySharing}`.
2. `ai/services/fleet/wireFleetRecentTurnsSource.mjs` and the wiring in `devFleetServer.mjs` beside the session-memories source.
3. `FleetControlBridge#fleetRecentTurns`, the two policy rows, and the wire vocabulary.

The source simulates no policy: what the plane's sharing policy answers is the answer, and the envelope carries `memorySharing` so a renderer can tell "no turns yet" from "this plane shares no peer turns".

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `fleetRecentTurns({agentIdentity, limit, before})` (new Fleet read verb) | #741's `query_recent_turns` contract; the session-memories sibling's envelope | One page of the seat's public turn summaries, newest first, with the plane's `memorySharing` verdict and the next cursor. | Every non-answer is `capability.state: 'unavailable'` with zero rows and its own constant reason: a failure the operation throws or RETURNS (`{error, message, code}`) → `recent-turns-read-failed` with a sanitized `detail`; a page the operation marks with its own scope refusal (`scope: 'fail-closed: …'`) → `recent-turns-scope-refused` with the marker as `detail`; a payload it never declared → `recent-turns-payload-unrecognized`. A clamped policy → a `wired` empty page with `memorySharing.clamped: true`; a seat with no shared turns → a `wired` empty page with its verdict. An unwired source → the bridge's `unavailable` envelope. | JSDoc | unit |
| `agentIdentity` (param) | the viewer pattern the sibling enforces | A canonical `@identity`; anything else is rejected, never coerced. | — | JSDoc | unit |
| `limit` / `before` (params) | `query_recent_turns` | `limit` 1..50, default 20; `before` is the operation's own cursor `{timestamp, id}` passed back unchanged. | Out of range or malformed → rejected. | JSDoc | unit |
| `fleetServerPolicy` rows (existing tables) | the memories verbs' rows | phase `awaiting-s5`, class `read-observe`. | — | — | the policy spec's verb census |
| `FLEET_WIRE_METHODS` (existing list) | `src/fleet/contract/wire.mjs` | gains `fleetRecentTurns`. | — | — | the contract spec |

## Decision Record impact

aligned-with ADR 0019 (no AiConfig key; the source reads no config and takes its operation and viewer by injection) and with the tenant contract #741 kept.

## Acceptance Criteria

- [x] AC-1 `readRecentTurns` calls the injected operation once with `{agentIdentity, memorySharing: 'team', detail: 'summary', limit}` (and `before` when given) and passes its `turns`, `nextCursor` and `memorySharing` through under a `wired` capability.
- [x] AC-2 A failure the operation throws or returns, a page it marks with a scope refusal, and an unrecognized payload each answer `unavailable` with zero rows and their own constant reason, the first two with a redacted `detail`; a clamped policy and an empty team page each answer a `wired` empty page that says so. (The returned forms were added at review: PR #795, RA-1.)
- [x] AC-3 A non-canonical `agentIdentity`, an out-of-range `limit` and a malformed `before` are rejected before the operation is called; a viewer the ingress did not bind refuses the read.
- [x] AC-4 `FleetControlBridge#fleetRecentTurns` passes the envelope through and names an unwired source `unavailable`; both policy tables and `FLEET_WIRE_METHODS` carry the verb, and the dev fleet server wires it in plane mode and in-process mode.
- ~~AC-5 [post-merge] the verb on the canonical plane~~ — moved to neomjs/neo-agent-institution#476 (AC-6): the plane's containerized fleet server answers this verb `awaiting-s5`, so the installed Fleet Manager is where it is witnessed.

## Out of Scope

- The Agent Detail pane that consumes it: neomjs/neo-agent-institution#476, after a pin carries this.
- Full turn detail or private projection: #741 refuses both on the widened path.
- A roster-card line: a card cannot call a per-seat verb per row; the lane line took the roster-row stamp for that reason (#740).

## Avoided Traps

- **Composing the stream from `fleetMemories` + `fleetSessionMemories`:** two reads per open detail, turn-level records instead of the one-line summaries, and one session at a time.
- **Passing `memorySharing` through from the client:** the widest a viewer may ask is `team`, and the plane clamps it; a client-chosen policy is a parameter with no honest second value here.
- **Calling the Memory Core from the cockpit:** the cockpit has no Memory Core session; every read it makes is a Fleet verb over the viewer the ingress bound.

## Related

neomjs/neo-agent-institution#391 (parent) · #741 (the policy-aware recency read) · #740 (the lane stamp, the card-side sibling) · #571 (the roster) · neomjs/neo-agent-institution#435

Structure map: owning folder `ai/services/fleet`; sibling precedent `fleetSessionMemoriesSource.mjs` + `wireFleetSessionMemoriesSource.mjs` (the new source and its wiring are the same pattern, same folder).

Live latest-open sweep: the latest 20 open issues at 2026-10-03T07:08Z (newest #788) and exact searches for `fleetRecentTurns`, `thought stream`, `recent turns` over open and closed issues: no equivalent (#741 and #740 are the closed producer leaves this builds on).
A2A in-flight sweep at 07:08Z, the latest 30 messages of every read-state: no claim on a recent-turns verb.
MC sweep: "Agent Detail thought stream pane not observed, a peer's recent turns answer empty for the cockpit viewer", 5 results, no prior decision found (the nearest is the open-work projection's design note, another pane).
Own-assignment sweep: 9 open, none on the Fleet read verbs.

Origin Session ID: 25618ee4-58d2-46dd-ae26-9dcf2854b14a
Retrieval Hint: "fleetRecentTurns Fleet verb query_recent_turns memorySharing team Agent Detail thought stream"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 25618ee4-58d2-46dd-ae26-9dcf2854b14a



## Timeline

- 2026-10-03T07:08:40Z @neo-fable assigned to @neo-fable
- 2026-10-03T07:08:42Z @neo-fable added the `enhancement` label
- 2026-10-03T07:08:42Z @neo-fable added the `ai` label
- 2026-10-03T07:08:42Z @neo-fable added the `agent-os` label
- 2026-10-03T07:08:51Z @neo-fable added parent issue #391
- 2026-10-03T07:16:55Z @neo-fable cross-referenced by #476
- 2026-10-03T07:17:07Z @neo-fable marked this issue as blocking #476
- 2026-10-03T07:17:42Z @neo-fable cross-referenced by PR #795
- 2026-10-03T12:27:40Z @neo-fable referenced in commit `2083685` - "chore(merge): bring origin/dev into the branch (#792)"
- 2026-10-03T12:27:40Z @neo-fable referenced in commit `b9f78ea` - "fix(fleet): the recent-turns source keeps the operation's returned failures and scope refusals apart from a wired page (#792)

The Memory Core's query_recent_turns answers two outcomes as payloads: an error object ({error, message, code}) from its catch, and a page marked with a fail-closed scope when no tenant resolves. The source accepted any turns array as a wired page and called an error object an unrecognized payload.

Now an error payload is a failed read with the redacted, bounded detail; a page carrying the operation's scope marker is unavailable with its own reason (recent-turns-scope-refused) and the marker as detail; rows are accepted only after both. The genuine empty team page and the clamped page stay wired with their verdict.

Arms: the declared shapes in the source spec, and one arm in the Memory Core's spec that runs the source over the real method for a tenant refusal, an empty team page and an unavailable graph reader."
- 2026-10-03T16:18:23Z @tobiu referenced in commit `12041ce` - "feat(fleet): the Fleet serves a seat's recent turn summaries (#792) (#795)

* feat(fleet): the Fleet serves a seat's recent turn summaries (#792)

The cockpit reaches the Memory Core only through Fleet read verbs, and none wrapped the recency read, so the Agent Detail's thought stream had no producer although the plane answers a peer's public turn summaries since the recency read learned the sharing policy. fleetRecentTurns is that verb: a viewer-bound source in the session-memories sibling's shape asks the injected query_recent_turns operation for one page of a seat's public summaries under the team policy and passes the rows, the next cursor and the plane's memorySharing verdict through a fail-honest envelope. The bridge verb, both policy rows, the wire vocabulary and the dev fleet server's wiring carry it.

* fix(fleet): the recent-turns source keeps the operation's returned failures and scope refusals apart from a wired page (#792)

The Memory Core's query_recent_turns answers two outcomes as payloads: an error object ({error, message, code}) from its catch, and a page marked with a fail-closed scope when no tenant resolves. The source accepted any turns array as a wired page and called an error object an unrecognized payload.

Now an error payload is a failed read with the redacted, bounded detail; a page carrying the operation's scope marker is unavailable with its own reason (recent-turns-scope-refused) and the marker as detail; rows are accepted only after both. The genuine empty team page and the clamped page stay wired with their verdict.

Arms: the declared shapes in the source spec, and one arm in the Memory Core's spec that runs the source over the real method for a tenant refusal, an empty team page and an unavailable graph reader."
- 2026-10-03T16:18:23Z @tobiu closed this issue

