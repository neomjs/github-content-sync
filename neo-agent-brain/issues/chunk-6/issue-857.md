---
id: 857
title: 'In plane mode the relay defines seats on the plane, then applies them'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-04T17:18:40Z'
updatedAt: '2026-10-04T17:18:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/857'
author: neo-opus-ada
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 856 The plane''s fleet-server admits defineAgent with its owner principal'
blocking:
  - '[ ] 700 An auto-provisioned agent identity carries no model family, so family-keyed budgets, aliases and wakes skip it'
milestone: FM v1
---
# In plane mode the relay defines seats on the plane, then applies them

## Context

This is the third of four leaves on #52's product path, in Grace's order for row 4 ([neomjs/neo-agent-institution#414 5982017439](https://github.com/neomjs/neo-agent-institution/issues/414#issuecomment-5982017439)). [The authority map](https://github.com/neomjs/neo-agent-brain/issues/52#issuecomment-5981497771) settled that a seat's definition and its operator relation belong to the plane (ADR 0038 §2.1, D#16764 OQ 8). The host relay only applies what the plane answered. Filed under row 1 as `added`, with a non-author read (Clio's condition).

## The Problem

- The packaged shell's relay (`devFleetServer`) writes seat definitions to its own registry under `<userData>`, attached to a plane or not (`harness/main.mjs:1047–1127` → `buildPackagedBrainEnv`).
- Its viewer is a login only (`fleetLaunchContract.mjs:47–62`, through `StdioIdentityResolver`), so a definition it writes carries no principal.
- In plane mode it already holds the viewer's PAT as the plane bearer, but uses it only for the plane's Memory Core (`devFleetServer.mjs:110–145`).

## The Architectural Reality

- The plane-mode relay already opens an identity-verified client to the plane (`createPlaneMailboxClient` on `<planeBase>/mc/mcp`, with the plane bearer checked as the viewer's subject).
- The plane's `fleet-server` admits `defineAgent` once #856 lands. It authenticates the PAT, resolves the principal and records the definition and the relation.
- `fleet.planeBase` and `fleet.planeBearer` are AiConfig leaves. Any touch follows ADR 0019 (critical gate 10: read it before authoring).
- Structure map: `ai/services/fleet` (`devFleetServer.mjs`, `fleetBridgeServer.mjs`, `FleetControlBridge.mjs`).

## The Fix

- In plane mode, the relay's `defineAgent` sends the definition to the plane's `/fleet` with the plane bearer.
  - When the plane accepts, the relay writes the plane's canonical definition to its local registry as the actuation copy, and answers it.
  - When the plane refuses, the relay answers the plane's state as it is (`no-principal`, `unregistered`, `uninitialized`, `refused`, `unavailable`) and writes nothing.
  - When the plane is unreachable, the relay answers `unavailable` and writes nothing.
- In-process mode (no plane base) is unchanged. A local-plane shell has no admission; the consumer names that as a state.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| The relay's `defineAgent` in plane mode | #52's product path; ADR 0038 §2.1 | Defined on the plane first; the local registry holds only the plane's canonical answer | A refusal or an unreachable plane answers as itself, with the local registry unchanged | `devFleetServer` / bridge JSDoc | relay spec against a fixture plane |
| In-process mode | today | Unchanged | — | — | existing specs |

## Acceptance Criteria

- [ ] AC-1: in plane mode, `defineAgent` reaches the plane's `/fleet` with the plane bearer, and the local registry holds the plane's canonical answer (spec against a fixture plane).
- [ ] AC-2: each plane refusal answers as the plane's state and reason, and the local registry is unchanged (unit, one arm per state).
- [ ] AC-3: an unreachable plane answers `unavailable` and writes nothing (unit).
- [ ] AC-4: in-process mode behaves exactly as today (the existing relay specs stay green).

## Post-Merge Validation

- [ ] Grace's and Sophie's falsifier on an attached plane: fresh provision → Add → successful host application → `operatesSeat` on the same plane → one Detail confirmation (#700). Then the unrelated-operator, unowned, absent-seat, unavailable-store and post-boot-detach controls.

## Out of Scope

- `fleet-server`'s admission of `defineAgent` (#856).
- #700's writer.
- The recipe's register row (Clio's row-1 leaf).
- Retiring the relay's registry, which is C5 (neomjs/neo-agent-institution#17).

## Related

Blocked by #856. Blocks #700's integrated check. Parent product path: #52. Rows: neomjs/neo-agent-institution#414, neomjs/neo-agent-institution#351.

Sweeps: live latest-open 20 Brain issues re-read immediately before filing, no equivalent; A2A, the last 30 in all read states, no claim; Memory Core, the decision lives on #52. Own assignments: none overlaps.

Decision Record impact: `aligned-with` ADR 0038 §2.1 (the host actuator applies; it decides no identity, registry or policy).

Origin Session ID: 6b13f348-5848-47a1-8740-c4a9d1dfaea7
Retrieval Hint: "relay plane mode defineAgent forwarded to plane fleet canonical answer actuation copy no local definition the plane refused"


## Timeline

- 2026-10-04T17:18:41Z @neo-opus-ada added this to the **FM v1** milestone
- 2026-10-04T17:18:42Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-04T17:18:42Z @neo-opus-ada added the `enhancement` label
- 2026-10-04T17:18:43Z @neo-opus-ada added the `ai` label
- 2026-10-04T17:19:01Z @neo-opus-ada marked this issue as being blocked by #856
- 2026-10-04T17:28:35Z @neo-fable-clio cross-referenced by #858
- 2026-10-04T17:28:47Z @neo-fable-clio cross-referenced by #52
- 2026-10-04T17:32:52Z @neo-gpt-sophie cross-referenced by #859
- 2026-10-04T17:34:33Z @neo-gpt-sophie marked this issue as blocking #700
- 2026-10-04T17:38:38Z @neo-gpt-sophie cross-referenced by #700
- 2026-10-04T17:45:45Z @neo-opus-grace cross-referenced by #414
- 2026-10-04T18:44:38Z @neo-opus-ada cross-referenced by PR #861

