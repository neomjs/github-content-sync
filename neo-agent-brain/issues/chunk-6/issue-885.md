---
id: 885
title: 'Start refuses a benched seat, at admission and again just before the spawn'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-05T12:47:54Z'
updatedAt: '2026-10-05T15:17:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/885'
author: neo-opus-vega
commentsCount: 1
parentIssue: 28
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 874 Start fleet reads a seat''s participation from its identity node, not the seed'
blocking: []
closedAt: '2026-10-05T15:01:37Z'
---
# Start refuses a benched seat, at admission and again just before the spawn

## Context

#28's design read ([5991791737](https://github.com/neomjs/neo-agent-brain/issues/28#issuecomment-5991791737)) names two refusals instead of a new display state. This is the second one: `startAgent` refuses a benched seat in words, at admission and again at `spawnPermitted`, the last check before the spawn. A bench that commits while a start is being prepared then wins (Sophie, [5991945216](https://github.com/neomjs/neo-agent-brain/issues/28#issuecomment-5991945216)). The bench itself is recorded by #883, and the Fleet reads it from the identity node through the presence snapshot (#874).

## The Problem

`startAgent` never reads participation. Start fleet's rule 2 skips a benched seat in the cockpit, but a per-card Start, a restart, or any other caller of the verb launches it. Start also admits a seat once at entry, so a bench recorded during the asynchronous preparation would still be followed by a spawn.

## The Architectural Reality

- `src/fleet/contract/launchAuthority.mjs` `launchRefusalOf(definition)` is the one start refusal, a pure predicate over the registry row. `FleetManager.assertStartPermitted` (admission, and restart before its stop), `startAgentProvisioned.mjs` `spawnPermitted` (the re-read before spawn) and `FleetControlBridge.fleetRoster` (`launchRefusal` on each DTO row) all ask it.
- `FleetManager.fleetPresenceStatus()` answers each seat's `participation` and `participationRead` (#874).

## The Fix

1. `launchRefusalOf(definition, participation)` takes the seat's participation as an optional second input. The release refusal comes first. Then a known non-`active` status refuses in words that carry the operator's date and reason.
2. `assertStartPermitted` reads the seat's participation from the presence snapshot before any preparation, so a benched seat's restart is refused before its stop.
3. `spawnPermitted` re-reads it, through a reader the Fleet passes in, and refuses the spawn when a bench landed during preparation.
4. `fleetRoster` stamps the same words as the row's `launchRefusal`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `launchRefusalOf(definition, participation)` (existing; gains an optional second input) | #28 design read | Release refusal first. Then `operator_benched` gives "benched by the operator on <date>: <reason>", and another non-`active` status gives its own name with the reason. | `participation` absent, `null`, or `active`: no refusal. The one-argument call is unchanged | JSDoc | `launchAuthority` spec |
| `FleetManager.assertStartPermitted` (existing; stays synchronous, gains an optional `participation` input) | this ticket | `startAgent` awaits the seat's participation (new `seatParticipation`) inside the seat's home and passes it in; `restartAgent` does the same before its stop. Both refuse a benched seat before preparation and before a stop | An unread participation read is no refusal; the cockpit's Start fleet excludes `unobserved` seats (Institution #568) | JSDoc | `FleetManager` spec |
| `spawnPermitted` (existing) | Sophie 5991945216 | Re-reads participation after preparation; a bench recorded meanwhile refuses the spawn | A bench recorded after this re-read lands on a seat that starts. That bound is stated, not claimed away | JSDoc | `startAgentProvisioned` spec: bench during delayed preparation |
| Fleet DTO row `launchRefusal` (existing) | this ticket | Carries the participation refusal | Unchanged for seats without one | JSDoc | `FleetControlBridge` spec |

## Acceptance Criteria

- [ ] `launchRefusalOf` gives the release refusal first, then the bench words with date and reason. Absent, `null`, or `active` participation gives no refusal (spec).
- [ ] `startAgent` and `restartAgent` refuse a benched seat before provisioning or stopping anything, and admit an `active` or unread seat (specs).
- [ ] A bench recorded while a start's preparation is pending refuses the spawn, and the harness is not started (spec with a delayed preparation).
- [ ] The roster row's `launchRefusal` carries the bench words (spec).
- [ ] Post-merge, on the plane: a per-card Start of a benched seat is refused with the operator's reason. Receipt on this ticket.
  Residual-Owner: #28
  *State 2026-10-05:* a per-card Start runs in the installed FM's relay, which runs the Institution's pinned Brain (`f24815d`, before this change). So the receipt waits for that pin to move, under neomjs/neo-agent-institution#568, which is blocked by neomjs/neo-agent-institution#571.

## Out of Scope

- The card's disabled Start and the Participation row (#28's Institution leaf).
- Refusing to bench a running seat (the cockpit verb, #28).
- Wake eligibility and the heartbeat (#879, #880).

## Decision Record impact

Aligned with ADR 0038: the start gate reads the plane's identity fact and decides nothing about identity.

## Related

#28 · #874 (the read) · #883 (the write) · #875

Live latest-open sweep: latest 20 open issues at 2026-10-05T12:47Z, no equivalent; "refuses benched" and "spawnPermitted" searches return #28 only · A2A in-flight sweep: no claim on the start gate in the last 60 min · Memory Core sweep: the design reads on #28 (Mnemo, Sophie) are the prior art, folded above.

Origin Session ID: 79265a5a-6888-4d34-94ee-0d933cbacff1

Retrieval Hint: `query_raw_memories("startAgent refuses a benched seat at admission and at spawnPermitted; launchRefusalOf takes participation")`



## Timeline

- 2026-10-05T12:47:54Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-05T12:47:55Z @neo-opus-vega added the `enhancement` label
- 2026-10-05T12:47:55Z @neo-opus-vega added the `ai` label
- 2026-10-05T12:47:55Z @neo-opus-vega added the `agent-os` label
- 2026-10-05T12:48:01Z @neo-opus-vega added parent issue #28
- 2026-10-05T12:48:03Z @neo-opus-vega marked this issue as being blocked by #874
- 2026-10-05T12:50:38Z @neo-opus-vega cross-referenced by #883
- 2026-10-05T12:59:21Z @neo-opus-vega cross-referenced by PR #886
- 2026-10-05T13:18:40Z @neo-opus-ada cross-referenced by #888
- 2026-10-05T13:39:38Z @neo-opus-vega referenced in commit `b3e33b6` - "feat(fleet): Start refuses a benched seat, at admission and again just before the spawn (#885)

launchRefusalOf takes the seat's participation: the release refusal first, then the operator's bench with its date and reason. startAgent reads it inside the seat's home so the start keeps its queue place, restartAgent before its stop, spawnPermitted re-reads it after preparation, and the roster row's launchRefusal carries the same words. An unread participation is no refusal."
- 2026-10-05T14:12:54Z @neo-opus-vega referenced in commit `b7ff8ab` - "feat(fleet): startAgent's JSDoc names the read its bench refusal depends on (#885)"
- 2026-10-05T15:01:11Z @tobiu referenced in commit `1879b58` - "feat(fleet): Start refuses a benched seat, at admission and again just before the spawn (#885) (#886)

* feat(fleet): Start refuses a benched seat, at admission and again just before the spawn (#885)

launchRefusalOf takes the seat's participation: the release refusal first, then the operator's bench with its date and reason. startAgent reads it inside the seat's home so the start keeps its queue place, restartAgent before its stop, spawnPermitted re-reads it after preparation, and the roster row's launchRefusal carries the same words. An unread participation is no refusal.

* feat(fleet): startAgent's JSDoc names the read its bench refusal depends on (#885)"
- 2026-10-05T15:01:38Z @tobiu closed this issue
### @neo-opus-ada - 2026-10-05T15:08:39Z

## Plane cut receipt: local plane at Brain `1879b588` (2026-10-05)

Operator-approved in-session at 13:44Z ("cut after #886"). #886 merged at 15:01:10Z as `1879b588af51cfd19932a60b1550e56b3c6e0bdf`. The local plane moved from `ed894a2a` to `1879b588`, which brings #887, #881, #882, #884, #889 and #886.

| | |
|---|---|
| build | 15:06:06Z → 15:06:39Z, exit 0 |
| recreate | 15:07:39Z (mc-server, kb-server, fleet-server), orchestrator 15:08:01Z |
| all healthy | 15:08:08Z |
| `mc-server` | image `d7759ac4ac63`, label and `/app/.neo-revision` = `1879b588af51cfd19932a60b1550e56b3c6e0bdf` |
| `kb-server` | image `c00a190d9561`, same revision |
| `orchestrator` | image `ac7a7bc051ab`, same revision |
| `fleet-server` | image `1b70412c9e19`, same revision |
| untouched | Chroma `b89d731f60ea`, ingress `bf26d90ce88a` |
| host daemons | wake and host-edge running again (pids 92083, 92085) |
| readback | Memory Core answers through a seat's MCP client after the cut (`list_messages`, 15:08Z) |

**No reseed.** `ai/graph/identityRoots.mjs` is unchanged since `ed894a2a`. #884 changed only `seedAgentIdentities.mjs`, so that a reseed keeps a runtime bench. #883's AC-5 receipt exercises that reseed itself.

The plane is ready for the post-merge receipts of #874 AC-5, #883 AC-5 and #885 AC-5.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code



