---
id: 1143
title: One seat's unprovable server refuses the whole continuity handoff
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-18T18:56:55Z'
updatedAt: '2026-10-18T20:10:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1143'
author: neo-opus-vega
commentsCount: 1
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# One seat's unprovable server refuses the whole continuity handoff

Part of neomjs/neo-agent-institution#732 (an FM update never requires closing a harness).

## Context

On 2026-10-18 at 18:05Z, the FM's quit for an update logged `continuity handoff starting: 3 running seat(s)`, then `refused at survivor-validation: proof failed: neural-link (9159 ms)`. A single server on one seat refused the whole handoff. That server could not prove because of #1140. The installer then refused (`handoff-incomplete`).

The next boot recovered from an *unclean* predecessor, so every running seat's generation became `stale`. From then on, every handoff with any seat running was refused, and stopping all seats was the only way to update.

## The Problem

Survivor validation is all-or-nothing. Any single unprovable fact about any seat turns the update into a refusal. One server, one stale generation or one seat that can't be identified is enough. The handoff exists to carry *proven* authority to the successor. Refusing everything is one way to stay safe; revoking just the unprovable part is another, equally safe, and it keeps every harness running.

## The Architectural Reality

- `FleetLifecycleService.handoffRuntimeCustody` (`ai/services/fleet/FleetLifecycleService.mjs`), `verifySeats` callback. For each running seat it throws on any of these, and the throw refuses the whole handoff:
  - `seat … is not a live original Start`
  - `runtime pin changed`
  - `admission generation missing` (any state other than `active` or `revoked`; so `stale` throws)
  - `resident input missing: <server>`
  - `credential value missing: <server>`
  - `proof failed: <server>`
- `McpLaunchAdmissionService.performHandoff`: any throw from `verifySeats` → `refused at survivor-validation`, grants cleared, journal closed. The successor reads `unclean-issuer-exit` and marks every running generation `stale`.
- Recovery already restores an active generation with some servers revoked: `recover()` maps saved revoked server rows to `REVOKED` and restores the rest.

Design authority: the operator, 2026-10-18 (quoted in the epic), and #674's AC-3: "`--quit --open` replaces the bundle while the seats keep running".

## The Fix

Per-seat unprovability revokes authority instead of refusing the update:

1. **A server that fails its proof, or misses a resident input or credential value:** revoke that server's grant (`revokeServer`), persist, continue. The seat survives with its proven servers; the revoked one reconnects at the seat's next Start.
2. **A running seat whose generation is `stale`:** carry it as stale. It holds no grants, so it hands nothing over.
3. **A running seat that is not a provable original Start** (identity, runtime pin): revoke its generation and carry the seat as unresolved, without authority.
4. **The handoff still refuses on issuer-level conditions only:** a pending Start, an unbound endpoint, a durability failure, the deadline.

Every change removes authority and none adds any, so the successor can only inherit less than before, never more.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetLifecycleService.handoffRuntimeCustody` survivor validation | #1051 continuity contract; operator 2026-10-18 | Per-seat unprovability revokes that server, generation or seat's authority; the handoff continues | Issuer-level conditions still refuse | method JSDoc | AC-1–3 |
| `McpLaunchAdmissionService.performHandoff` refusal set | same | Refuses only for pending Start, endpoint, durability, deadline | — | class doc | AC-4 |
| Sealed snapshot | journal codec | Revoked server rows and stale or unresolved seats are recorded as such | — | — | AC-1/2 |

Decision Record impact: amends #1051's survivor-validation rule. The owning runtime-generation record is ADR 0034 §2.5 per #674 (aligned-with; not re-read for this ticket).

## Acceptance Criteria

- [ ] AC-1: Two running seats, one whose neural-link proof fails. The handoff seals. In the sealed snapshot only that server's grant is revoked, the other servers and seat stay `active`, and the successor recovers both seats. Unit, red at `9cd1528b`.
- [ ] AC-2: Running seats with `stale` generations (an unclean predecessor). The handoff seals and carries them as stale. Unit, red at `9cd1528b`.
- [ ] AC-3: A running seat that fails the original-Start identity check. Its generation is revoked and the handoff seals. Unit.
- [ ] AC-4: A pending Start and a durability failure still refuse the handoff. Unit, unchanged behaviour.

## Out of Scope

- The handoff's time budget (#1081).
- The payload re-hash inside the deadline (#1142).
- Installer behaviour on refusal (sibling leaf under the epic).

## Related

neomjs/neo-agent-institution#732 (epic), #1051, #1081, #1140 / #1141, #1142, neomjs/neo-agent-institution#674.

Sweeps: as on the epic (latest 20 open Brain issues at 2026-10-18T18:52Z; nearest #1081, budget-only; org search found no equivalent). A2A and Memory Core sweeps not runnable from this seat.

Origin Session ID: c092b691-0e97-4623-b91a-86a8c9ca2e84

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-10-18T18:56:57Z @neo-opus-vega added the `bug` label
- 2026-10-18T18:56:57Z @neo-opus-vega added the `ai` label
- 2026-10-18T18:56:57Z @neo-opus-vega added the `agent-os` label
- 2026-10-18T18:57:08Z @neo-opus-vega added parent issue #732
- 2026-10-18T18:57:45Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-18T19:16:39Z @neo-opus-vega cross-referenced by #1144
### @neo-opus-vega - 2026-10-18T20:10:25Z

**Handover (Vega, session sunset 2026-10-18).** Not started. Write the red tests first (AC-1/AC-2), following the "seat survival" fixtures in `test/playwright/unit/ai/FleetLifecycleService.spec.mjs`: start, stop or crash, handoff, then recover.

Until this lands, any FM update with running seats still refuses when one server can't prove its credential, and after an unclean exit every running seat stays stale. Tonight's install therefore needed every seat stopped first. Related budget work: #1081.

Owner: Vega.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-10-18T20:11:02Z @neo-opus-vega cross-referenced by #732

