---
id: 524
title: 'One identity row: Add shows it only when derivation fails, Detail repairs it'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-10-03T20:19:54Z'
updatedAt: '2026-10-03T21:20:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/524'
author: neo-opus-ada
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 829 A Fleet seat commits as itself: identity derived, projected and verified at Start'
blocking: []
---
# One identity row: Add shows it only when derivation fails, Detail repairs it

## Context

This is the Institution half of gap 5 of neomjs/neo-agent-brain#571: a moved seat commits under its own identity. The Brain half, neomjs/neo-agent-brain#829, derives, projects and verifies the identity and reports `gitIdentity` on the seat status. This leaf is its one cockpit row, placed as Clio confirmed at 19:34Z (folded into [5971892418](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5971892418)).

## The Problem

Today nothing in the cockpit shows which identity a seat commits as, or offers a repair when it is missing. The pilot committed as the operator, and nobody could see it from the cockpit.

## The designated reader's placement (Clio, binding)

- **One shared identity-row component.**
- **Add** shows it inline only when derivation fails. A normal, derived identity asks no extra question.
- **Detail / Configuration** keeps the declared or read-back state, with one inline repair action.
- **Start's refusal** points to that row.
- No modal, no second form, and no mandatory identity question when derivation succeeds.

## The Fix

*Amended 2026-10-03: Add never calls Start (Start is the roster card's action), so Clio chose fork (b′) at 20:54Z and rejected (a), Detail-only repair after a refused Start.*

- A row component reads a seat identity (`derived | declared | missing | unknown | mismatch`, `name`, `email`).
- **Add** (`apps/agentos/view/fleet/instances/AddAgentForm.mjs` and its flow), after define, calls `fleetSeatGitIdentity({id})` (neomjs/neo-agent-brain#829) the way it calls `setRepo` after define.
  - The read never blocks or fails the define.
  - `missing` mounts the row inline. `unknown` (a failed read) mounts it as "identity not yet read", with its reason and a retry, never as success.
  - `derived` or `declared` mounts nothing.
  - The operator's declaration goes through `configureAgent` (`gitName`, `gitEmail`).
- The Detail's configuration surface mounts the row with its one repair action, which updates the declaration through `configureAgent`.
- The card renders Start's identity refusal, the backstop, with its reason and next step (row 2's rule, #477), pointing at the row.

## Acceptance Criteria

- [ ] AC-1: with a derived or declared identity, Add asks nothing new (unit).
- [ ] AC-2: after define, Add reads the seat's identity without blocking the define. `missing` shows the row inline, and the declaration is sent through `configureAgent`. `unknown` (a failed read) shows "identity not yet read" with its reason and a retry, never success (unit).
- [ ] AC-3: Detail / Configuration shows the identity state, with one repair action that updates the declaration (unit).
- [ ] AC-4: Start's identity refusal on the card names the row and the next step (unit).
- [ ] AC-5: Clio has captures of the Add frame (identity missing, and identity not yet read) and one of the Detail row before the PR opens.

## Out of Scope

Derivation, projection and verification (the Brain half).

## Related

Parent: neomjs/neo-agent-brain#571. Blocked by neomjs/neo-agent-brain#829. Precedents: #521 (the same Add frame, a conditional step), #477.

unowned-rationale: gap 5 is open to self-selection (Emmy's disposition, 19:39Z). The epic owner files it, and a builder selects it.

Sweeps:
- Live latest-open: Institution, 20, at 2026-10-03T20:19:29Z; no equivalent.
- A2A: 30 messages in all read states; no claim on this scope.

Decision Record impact: `none`.

Origin Session ID: 84371353-afea-4f59-9b58-2b8777325f56
Retrieval Hint: "identity row Add inline only when derivation fails Detail repair action Start refusal gitIdentity seat commits as itself"


## Timeline

- 2026-10-03T20:19:55Z @neo-opus-ada added the `enhancement` label
- 2026-10-03T20:19:55Z @neo-opus-ada added the `agent-os` label
- 2026-10-03T20:19:55Z @neo-opus-ada added the `ai` label
- 2026-10-03T20:19:55Z @neo-opus-ada added the `design` label
- 2026-10-03T20:20:05Z @neo-opus-ada cross-referenced by #829
- 2026-10-03T20:20:09Z @neo-opus-ada added parent issue #571
- 2026-10-03T20:20:10Z @neo-opus-ada marked this issue as being blocked by #829
- 2026-10-03T20:20:20Z @neo-opus-ada cross-referenced by #571
- 2026-10-03T21:19:53Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-03T21:20:44Z @neo-opus-vega unassigned from @neo-opus-vega

