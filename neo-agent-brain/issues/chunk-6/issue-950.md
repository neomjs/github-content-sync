---
id: 950
title: 'A running seat''s new repositories are cloned on the fly, not at restart'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-09T12:35:41Z'
updatedAt: '2026-10-09T12:50:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/950'
author: neo-fable-clio
commentsCount: 0
parentIssue: 571
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
# A running seat's new repositories are cloned on the fly, not at restart

## Context

Operator, 2026-10-09: most active seats list `neo`, `neo-agent-brain` and `neo-agent-institution` in Accounts and not `neo-agent-skills`; a peer whose harness cannot clone into its own seat folder (Emmy's Codex sandbox) needs the Fleet to do it, and "when a peer session is running, getting a new clone folder on the fly without starting a new session is key" — Claude and Codex harnesses already work across several folders, and a peer can drive the Fleet Manager itself (Neural Link or native control): enter Accounts, add the repository to its own seat, get the clone, use it. The distinction he drew: before a seat's first boot the Start may clone; afterwards the clone must come without a restart.

## The Problem

Saving the Repositories card records the list and nothing else. `FleetManager.setRepos` (`ai/services/fleet/FleetManager.mjs:836`) validates the entries (a working repository first, no duplicate of it, no checkout collision) and writes `metadata.repos` to the registry; the clone happens only in `startAgentProvisioned` (`ai/services/fleet/startAgentProvisioned.mjs:572`), which loops `metadata.repos` through `ensureRepo` and records per-repository outcomes. A seat that is already running gets its new folder at its next Start, which costs the session; Accounts shows that folder's state as the outcome of the *last start* (#408), because that is the only time anything happens.

## The Architectural Reality

- `ensureAgentRepo` (`ai/services/fleet/ensureAgentRepo.mjs`) is already the one entry point: derive the stable path, inspect the disk, clone when absent, reuse a valid checkout, refuse a foreign occupant; the seat's credential reaches only its origin through the child's environment (`provisionAgentRepo.mjs`). Nothing in it depends on the harness process: a clone beside a running seat is an ordinary host operation.
- The Start converges the commit identity on every checkout (`convergeGitIdentity`) before anything runs there; a checkout prepared at runtime must get the same.
- `FleetLifecycleService.setRepoOutcomes` (`FleetLifecycleService.mjs:1308`) binds outcome rows to one launch (`pid`, `startedAt`) and refuses a stale one; a runtime preparation of a live seat records against that same live launch.
- The seat's projected workspace (`prepareManagedAgentWorkspace.mjs:552` builds the assignment map from `metadata.repo` plus `metadata.repos`) is written at Start; a clone prepared at runtime exists on disk now and is named in the seat's projected instructions from its next Start. The ticket says so rather than pretending otherwise.
- Fleet verbs travel through `dispatchFleetRequest` as single `{id, …}` payloads; `setRepo`, `setRepos` and `setAvatar` are the fleet-authority precedent (not control-plane: a seat's own definition).

## The Fix

One verb, `FleetManager.prepareRepos({id})`, beside `setRepos`: for a seat with a live launch, run `ensureRepo` for every listed repository, converge the identity on each newly created checkout, record the rows through `setRepoOutcomes` against the live launch, and return them. The Start's loop and this verb share one helper so the two paths cannot drift. Outcome rows gain `via: 'start' | 'prepare'` and `at` (ISO), so a consumer can say "prepared now" apart from "at the last start".

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetManager.prepareRepos({id})` (new, wire-dispatched) | `FleetManager` fleet authority, as `setRepos`; `ensureAgentRepo` for the disk; `convergeGitIdentity` for the identity | For a seat with a live launch: every listed repository ensured (clone if absent, reuse if valid), identity converged on new checkouts, rows `[{repoSlug, state: prepared \| failed, reason?, via: 'prepare', at}]` recorded on the live launch and returned | Unknown seat → `null`; no live launch → `{state: 'at-start', repos: []}` and nothing touched (the Start clones); a foreign occupant → that row `failed` with a redacted reason, the others continue; a credential error never echoes the PAT (`redactReadFailure`) | Verb JSDoc; `status()` `@returns` names `via`/`at` | Unit arms with the `cloneRepo` seam; the dispatch spec lists the verb |
| `status.repos[]` rows | `FleetLifecycleService.setRepoOutcomes` | Rows carry `via` and `at`; a runtime preparation replaces the rows for the repositories it touched and keeps the others | Rows from a launch the seat has since replaced are not written (existing rule) | `setRepoOutcomes` JSDoc | Unit arm: start rows then a prepare merge |
| `startAgentProvisioned` repo loop | existing | Unchanged behavior, now through the shared helper with `via: 'start'` | — | — | Existing Start arms stay green |

Decision Record impact: none (a verb beside its siblings; no ADR touched).

## Acceptance Criteria

- [ ] AC-1 For a live seat listing two repositories of which one is missing on disk, `prepareRepos` clones only the missing one (the `cloneRepo` seam observes exactly one call), converges its identity, and returns two rows: `prepared` for both, `via: 'prepare'`, with `at`.
- [ ] AC-2 Without a live launch the verb touches nothing and answers `{state: 'at-start'}`; an unknown id answers `null`.
- [ ] AC-3 A foreign occupant at one path yields `failed` with a redacted reason for that repository while the other is prepared; the launch record holds both rows.
- [ ] AC-4 The Start's rows read `via: 'start'`; existing Start arms unchanged; the dispatch spec lists `prepareRepos`.
- [ ] AC-5 (post-merge, with the Institution leaf) On the installed Fleet Manager a running seat gains its new folder while its session runs and commits and pushes from it in that same session (Ada's witness shape; Codex reach unverified, Emmy's seat falsifies it); the receipt is the self-service walk recorded on Institution #642.

## Out of Scope

Telling the seat by A2A that its folder exists (the seat that drove the card knows; an operator-driven add is read on the card); refreshing existing clones (Grace's D#19411); deleting a checkout (#951); the Accounts card itself (Institution #642).

## Avoided Traps

- A second clone path: the Start loop and the verb share one helper, or they drift.
- Cloning for a seat that has never started: the Start owns first-boot provisioning (the working checkout, the workspace projection); the verb answers `at-start`.
- Reporting a runtime preparation as a start outcome: the `via` word keeps the card honest (#408's "last start" suffix).

## Related

Parent #571 (the seat folder layout the Fleet provisions and launches into); sibling #951 (the guarded checkout deletion); consumer: Institution #642 (the Repositories card); #682 (a seat holds more than one repository, cloned before launch); Institution #407, #408; D#19411 (clone refresh).

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-09 12:34Z; no equivalent. A2A in-flight claim sweep: none. Memory Core rationale sweep: Euclid's 2026-10-03 receipt on #571 states the current rule ("additions clone on next start, removed declarations retain checkouts"); no prior design for a runtime path.

Origin Session ID: 4be92a90-698e-4ba2-b207-47b79f518bd3

Retrieval Hint: "prepareRepos running seat clone on the fly setRepos ensureAgentRepo via prepare"

## Timeline

- 2026-10-09T12:35:42Z @neo-fable-clio added the `enhancement` label
- 2026-10-09T12:35:42Z @neo-fable-clio added the `ai` label
- 2026-10-09T12:35:43Z @neo-fable-clio added the `agent-os` label
- 2026-10-09T12:36:52Z @neo-fable-clio added parent issue #571
- 2026-10-09T12:37:33Z @neo-fable-clio cross-referenced by #951

