---
id: 883
title: 'The plane host records a seat''s bench on its identity node, and a reseed or a sign-in keeps it'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-05T12:18:14Z'
updatedAt: '2026-10-05T15:17:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/883'
author: neo-opus-vega
commentsCount: 0
parentIssue: 28
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-05T13:37:59Z'
---
# The plane host records a seat's bench on its identity node, and a reseed or a sign-in keeps it

## Context

#28's converged design puts a bench on the plane's identity node, written plane-side with identity-wide authority: intake [5992121734](https://github.com/neomjs/neo-agent-brain/issues/28#issuecomment-5992121734), Mnemo's read [5991791737](https://github.com/neomjs/neo-agent-brain/issues/28#issuecomment-5991791737), Ada's route [5992040789](https://github.com/neomjs/neo-agent-brain/issues/28#issuecomment-5992040789) with Sophie's amendment. Today the only identity-wide writer is the plane host, so the write lands first as a host command. The cockpit's verb waits for a principal with that scope, and #28 stays open for it.

## The Problem

- **No path records a bench.** Changing participation takes a roots edit, a PR, a cut and a reseed. An operator without our roots has no path at all.
- **Two writers undo a decision.** The seeder projects the roots' participation over an existing node (Euclid's probe [5991931647](https://github.com/neomjs/neo-agent-brain/issues/28#issuecomment-5991931647)). The Memory Core's sign-in refresh rewrites an auto-provisioned node as `active` on every authentication.

## The Architectural Reality

- `ai/scripts/fleet/seatOperators.mjs` is the plane-host command precedent: an `os-user:` actor, and a dry run without `--apply`.
- `ai/scripts/setup/seedAgentIdentities.mjs` layers each root's properties over the existing node.
- `ai/mcp/server/memory-core/Server.mjs` `ensureAgentIdentityForAuthContext` writes `fullProperties`, including `participationStatus: 'active'`, onto an auto-provisioned node at each sign-in.
- `learn/agentos/IdentitySchema.md` names the seeder the canonical update path.
- `who_is_online` reads identity rows from SQLite, so a write shows on the next read.

## The Fix

1. A write sets four fields on an existing `AgentIdentity` node and clears `reactivationTrigger`:
   - `participationStatus`: `active` or `operator_benched`;
   - `statusReason`: required for a bench, cleared on return;
   - `since`;
   - `participationDecidedBy`: the actor.
2. A host command beside `seatOperators.mjs`: `bench --identity <id> --reason <text>`, `activate --identity <id>`, `show --identity <id>`. It is a dry run unless `--apply`, and its actor is `os-user:<user>`. On the plane it runs in the Memory Core container.
3. The seeder leaves the participation fields of a node carrying `participationDecidedBy` alone, and says so in its log. A node without that field is projected as today.
4. The sign-in refresh never writes `participationStatus` on an existing node. A first sign-in still creates the node `active`.
5. `IdentitySchema.md` states the precedence and the command.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `ai/scripts/fleet/participation.mjs` (new) | #28 route (Ada 5992040789) | `bench --identity <id> --reason <text>`, `activate --identity <id>` and `show --identity <id>`. Each prints its result as JSON and exits 1 on a refusal. Without `--apply` nothing is written. The actor is `os-user:<user>` | A missing `--identity`, a bench without `--reason`, or an unknown command is refused before the graph opens. A graph that did not open exits 1 with its reason | Module JSDoc; `IdentitySchema.md` | `participation.spec` |
| `recordParticipation` (new; `ai/services/memory-core/recordParticipation.mjs`) | this ticket | Writes `participationStatus`, `statusReason`, `since`, `participationDecidedBy` and `reactivationTrigger: null` on an existing `AgentIdentity` node through `upsertGlobalNode` | Unknown id, a node that is not an identity, a bench without a reason, or no actor: refused, nothing written. Re-recording the same decision writes nothing, so `since` keeps its date | JSDoc | `participation.spec` |
| Node fields `participationDecidedBy` (new) and `since` | `IdentitySchema.md` › Participation decisions | Mark a decision the operator recorded, and when it began | A node without `participationDecidedBy` carries the seed's participation | `IdentitySchema.md` | specs above |
| `seedAgentIdentities` precedence (existing) | this ticket | Leaves a decided node's participation fields alone and logs "participation kept" | A node without a decision is projected as today | Script header; `IdentitySchema.md` | `seedAgentIdentities.spec` (#883 describe) |
| `ensureAgentIdentityForAuthContext` (existing) | this ticket | Writes `participationStatus` only when it creates a node | Other refreshed properties are unchanged | JSDoc | `Server.spec` (#883 arm) |

The two refusals of #28's design read: Start's refusal of a benched seat is #885. Refusing to bench a seat whose runtime is observed up stays with #28's cockpit verb, which runs where the Fleet observes runtime. This host command does not observe runtime and does not refuse a running seat.

## Acceptance Criteria

- [ ] `bench` writes the four fields, and `activate` returns the identity to `active` with the reason cleared. Without `--apply`, nothing is written. An unknown id, a node that is not an `AgentIdentity`, and a bench without a reason are refused (specs).
- [ ] A reseed keeps a recorded decision in both directions: a recorded `active` over a benched root, and a recorded bench over an active root. It still projects participation onto a node without a decision (specs).
- [ ] A sign-in refresh of an auto-provisioned node keeps a recorded bench, and a first sign-in still creates the node `active` (specs).
- [ ] `IdentitySchema.md` names the precedence and the command.
- [ ] Post-merge, on the plane, one seat is benched and then returned through the command. `who_is_online` reads both states with the reason, and a reseed between them keeps the decision. Receipt on this ticket.
  Residual-Owner: #28
  *State 2026-10-05:* on the plane at `1879b588` the command failed (`TypeError: Neo.get is not a function`) and wrote nothing. The fix is #891 (PR #892). Its fresh-process spec covers bench, reseed-keeps-it and return on a seeded identity. The plane receipt re-runs after the cut that carries #892.

## Out of Scope

- The cockpit's wire verb and its identity-wide principal (#28 stays open for it), and that verb's refusal to bench a seat observed up.
- Start's refusal of a benched seat, at admission and before the spawn (#885).
- The other readers (#879, #880) and the Detail row (#28's Institution leaf).
- The refresh's other properties, such as `trustTier`, which this change leaves unchanged.

## Decision Record impact

Aligned with ADR 0038 (identity facts stay with the plane) and ADR 0032 (`participationStatus` is an established operational field).

## Related

#28 (Mnemo 5991791737 · Ada 5992040789 · Sophie 5991945216 · Euclid 5991931647 · Emmy 5991932341) · #874 · #875 · `learn/agentos/IdentitySchema.md`

Live latest-open sweep: latest 20 open issues at 2026-10-05T12:17Z, no equivalent; "bench" and "seeder" title searches return the #28 family only · A2A in-flight sweep: no claim on the write in the last 60 min · Memory Core sweep: Euclid's and Emmy's controls (survive sign-in, reseed and relaunch; named precedence and provenance) are folded into the ACs.

Origin Session ID: 79265a5a-6888-4d34-94ee-0d933cbacff1

Retrieval Hint: `query_raw_memories("plane host command records a bench on the identity node; seeder and sign-in refresh keep a recorded participation decision")`



## Timeline

- 2026-10-05T12:18:14Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-05T12:18:16Z @neo-opus-vega added the `enhancement` label
- 2026-10-05T12:18:16Z @neo-opus-vega added the `ai` label
- 2026-10-05T12:18:16Z @neo-opus-vega added the `agent-os` label
- 2026-10-05T12:18:20Z @neo-opus-vega added parent issue #28
- 2026-10-05T12:29:11Z @neo-opus-vega cross-referenced by PR #884
- 2026-10-05T12:47:55Z @neo-opus-vega cross-referenced by #885
- 2026-10-05T12:50:40Z @neo-opus-vega referenced in commit `81ab379` - "feat(fleet): the sign-in refresh spec removes the benched identity it creates (#883)"
- 2026-10-05T13:37:59Z @tobiu referenced in commit `df5353e` - "feat(fleet): the plane host records a seat's bench on its identity node, and a reseed or a sign-in keeps it (#883) (#884)

* feat(fleet): the plane host records a seat's bench on its identity node, and a reseed or a sign-in keeps it (#883)

A bench took a roots edit, a PR, a cut and a reseed, and two writers undid
any decision: the seeder projected the roots' participation over the node, and
the sign-in refresh rewrote an auto-provisioned node as active.

- recordParticipation writes participationStatus, statusReason, since and
  participationDecidedBy on an existing AgentIdentity node; re-recording the
  same decision writes nothing, so since keeps its date.
- ai/scripts/fleet/participation.mjs (bench | activate | show) is the plane-host
  path, a dry run unless --apply, with an os-user actor.
- The seeder leaves the participation fields of a decided node alone, reading
  them from the stored row as it already did for createdAt.
- The sign-in refresh writes participationStatus only when it creates a node.
- IdentitySchema.md states the precedence and the command.

* feat(fleet): the sign-in refresh spec removes the benched identity it creates (#883)"
- 2026-10-05T13:38:00Z @tobiu closed this issue
- 2026-10-05T14:16:05Z @neo-gpt-sophie cross-referenced by PR #886
- 2026-10-05T15:15:02Z @neo-opus-vega cross-referenced by #891
- 2026-10-05T15:17:01Z @neo-opus-vega cross-referenced by PR #892

