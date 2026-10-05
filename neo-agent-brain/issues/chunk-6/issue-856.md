---
id: 856
title: The plane's fleet-server admits defineAgent with its owner principal
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-04T17:18:20Z'
updatedAt: '2026-10-05T11:03:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/856'
author: neo-opus-ada
commentsCount: 0
parentIssue: 83
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 52 Build ownerPrincipal + the operator-to-agent derived relation (normalization contract owned)'
blocking:
  - '[x] 857 In plane mode the relay defines seats on the plane, then applies them'
closedAt: '2026-10-05T11:03:58Z'
milestone: FM v1
---
# The plane's fleet-server admits defineAgent with its owner principal

## Context

This is the second of four leaves on #52's product path. The order is Grace's, for row 4 ([neomjs/neo-agent-institution#414 5982017439](https://github.com/neomjs/neo-agent-institution/issues/414#issuecomment-5982017439)):
1. S4b (#52);
2. this leaf;
3. the relay defines seats on the plane;
4. #700 reads `operatesSeat` from the plane.

[The authority map](https://github.com/neomjs/neo-agent-brain/issues/52#issuecomment-5981497771) and its three answers settled that only the plane can mint an owner principal: Clio 5981979269, Sophie 5981965903 and Grace. Today the plane's composed Fleet service parks the verb that would use it. Filed under row 1 as `added`, with a non-author read (Clio's condition).

## The Problem

- `FLEET_S1_METHOD_POLICY` in `ai/services/fleet/fleetServerPolicy.mjs` holds `defineAgent: 'awaiting-s4'`, so the composed `fleet-server` answers `degraded` and no seat can be defined on the plane.
- S4b adds the stamp, but on the plane it stays dormant while the verb is parked. In S4b, `dispatchFleetRequest` forwards an admission only to `SEAT_CREATING_METHODS` (`defineAgent`), and `FleetRegistryService.defineAgent` stamps the relation from `admission.ownerPrincipal`.

## The Architectural Reality

- `createFleetRequestContext` (`fleetServer.mjs:120–145`) stamps `ownerPrincipal` from `ForgeConnectionRegistryService.resolveOwner`. Without one it stamps `ownerResolution {state, reason}`.
- The policy already refuses a lifecycle-write verb whose context has no principal, naming the resolution: "requires a forge-resolved admission subject — the owner is <state>: <reason>" (`fleetServerPolicy.mjs`, around `:170–178`).
- The composed service is the profile-gated `fleet-server` (`deploy/cloud/docker-compose.yml:664–711`; `NEO_AUTH_MODE` defaults to `github-pat`; `NEO_FLEET_DATA_DIR` is its own root).
- Structure map: `ai/services/fleet` (`fleetServerPolicy.mjs`, `fleetServer.mjs`, `dispatchFleetRequest.mjs`, `FleetRegistryService.mjs`).

## The Fix

- `defineAgent` leaves `awaiting-s4` and is admitted as a lifecycle write that requires `ownerPrincipal`. Without one it refuses through the existing path, with the request's own `ownerResolution` (`uninitialized`, `unregistered`, `refused`, `unavailable`), and writes nothing.
- The admitted call reaches `FleetRegistryService.defineAgent` with `{ownerPrincipal}`. That writes the definition in the service's `dataDir` and stamps the relation (S4b).
- Every other `awaiting-s4` verb stays parked.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `defineAgent` on the composed `fleet-server` | #52's product path; ADR 0038 §2.1 | Admitted only with an `ownerPrincipal`. Writes the definition in the service's `dataDir` and stamps the relation | No principal: refused with the owner resolution's state and reason; nothing written | `FLEET_S1_METHOD_POLICY` JSDoc | fleet-server policy and dispatch specs |
| The other seat verbs | the S1 ledger | Unchanged: `awaiting-s4` | — | — | ledger assertion |

## Acceptance Criteria

- [ ] AC-1: on the composed service, a forge-authenticated `defineAgent` whose owner resolves writes the definition and stamps the relation, and `operatesSeat(principal, id)` answers `operates` (spec at the `fleetServer` boundary, temp data root, real registry).
- [ ] AC-2: without a principal the call refuses with the owner resolution's state and reason (`uninitialized`, `unregistered`, `refused`, `unavailable`), and writes no definition and no relation (unit, one arm each).
- [ ] AC-3: a possession-only admission, with no provider facts, refuses and writes nothing (unit).
- [ ] AC-4: every other `awaiting-s4` verb is still parked (ledger assertion).

## Out of Scope

- The relay sending its `defineAgent` to the plane (the next leaf).
- #700's writer.
- The recipe's register row on the plane host (Clio's row-1 leaf).
- Owner isolation for start, stop and remove (S4b retains today's admission).

## Related

Blocked by #52. Blocks the relay leaf and #700. Rows: neomjs/neo-agent-institution#414 (row 4), neomjs/neo-agent-institution#351 (row 1). Row 4's cut-line: if this has not merged by 2026-10-20, Grace brings the operator a fallback.

Sweeps: live latest-open 20 Brain issues re-read immediately before filing, no equivalent; A2A, the last 30 in all read states, no claim on this scope; Memory Core, the decision lives on #52. Own assignments: #52 is this leaf's prerequisite and does not overlap.

Decision Record impact: `aligned-with` ADR 0038 §2.1 (definitions are plane-owned).

Origin Session ID: 6b13f348-5848-47a1-8740-c4a9d1dfaea7
Retrieval Hint: "fleet-server admits defineAgent ownerPrincipal awaiting-s4 plane-side seat definition relation stamp"


## Timeline

- 2026-10-04T17:18:20Z @neo-opus-ada added this to the **FM v1** milestone
- 2026-10-04T17:18:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-04T17:18:21Z @neo-opus-ada added the `enhancement` label
- 2026-10-04T17:18:21Z @neo-opus-ada added the `ai` label
- 2026-10-04T17:18:42Z @neo-opus-ada cross-referenced by #857
- 2026-10-04T17:18:59Z @neo-opus-ada marked this issue as being blocked by #52
- 2026-10-04T17:19:01Z @neo-opus-ada marked this issue as blocking #857
- 2026-10-04T17:28:35Z @neo-fable-clio cross-referenced by #858
- 2026-10-04T17:28:47Z @neo-fable-clio cross-referenced by #52
- 2026-10-04T17:32:52Z @neo-gpt-sophie cross-referenced by #859
- 2026-10-04T17:38:38Z @neo-gpt-sophie cross-referenced by #700
- 2026-10-04T18:35:26Z @neo-opus-grace cross-referenced by #414
- 2026-10-04T18:44:38Z @neo-opus-ada cross-referenced by PR #861
- 2026-10-04T19:50:47Z @neo-opus-ada cross-referenced by #863
- 2026-10-04T20:48:58Z @neo-opus-ada added parent issue #83
- 2026-10-05T09:31:15Z @neo-fable cross-referenced by #28
- 2026-10-05T10:12:15Z @neo-opus-ada referenced in commit `82330cf` - "docs(fleet): the served-verbs JSDoc names no ticket (#856)"
- 2026-10-05T10:12:17Z @neo-opus-ada cross-referenced by PR #872
- 2026-10-05T10:29:40Z @neo-opus-ada referenced in commit `f8e2664` - "chore(fleet): merge dev, the ledger spec keeps #869's catalog verb parked and defineAgent served (#856)"
- 2026-10-05T11:03:58Z @tobiu referenced in commit `819c0fc` - "feat(fleet): the plane's fleet-server admits defineAgent with its owner principal (#856) (#872)

* feat(fleet): the plane's fleet-server admits defineAgent with its owner principal (#856)

defineAgent leaves awaiting-s4. The S1 dispatcher already refuses a
lifecycle-write verb with no ownerPrincipal, naming the owner resolution,
and already hands the admission to the seat-creating verb, so the composed
service now writes the definition in its own dataDir and stamps the
operator relation from S4b (#861). Every other awaiting-s4 verb stays
parked.

* docs(fleet): the served-verbs JSDoc names no ticket (#856)"
- 2026-10-05T11:03:58Z @tobiu closed this issue
- 2026-10-05T11:07:03Z @neo-opus-ada cross-referenced by PR #877
- 2026-10-05T13:30:11Z @neo-gpt-sophie cross-referenced by PR #881
- 2026-10-05T14:02:23Z @neo-opus-ada cross-referenced by #571

