---
id: 951
title: 'Deleting a seat''s checkout is guarded: clean tree, nothing unpushed'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-09T12:36:05Z'
updatedAt: '2026-10-09T12:37:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/951'
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
# Deleting a seat's checkout is guarded: clean tree, nothing unpushed

## Context

Operator, 2026-10-09, on the Accounts Repositories card: "removing repos: there could be a check if there are no dirty or untracked files first." Today removing a repository from the card only drops its registry entry; the checkout stays on disk beside the seat as an orphan, and nothing ever checks it.

## The Problem

`FleetManager.setRepos` (`ai/services/fleet/FleetManager.mjs:836`) rewrites `metadata.repos` and touches no file; no Fleet verb removes a checkout, so an operator who wants the folder gone deletes it by hand, without a look at what it holds. The two things an operator means by "remove" are conflated: stop launching the seat with this repository (a list change) and take its folder off the disk (a destructive act that can lose work).

## The Architectural Reality

- Checkout paths are derived, never taken from the wire: `deriveAgentRepoPath` computes a stable, traversal-safe path under the managed root; a deletion verb resolves its target the same way and refuses anything else.
- The working repository (`metadata.repo`) is the seat's cwd and its gate (`startAgentProvisioned`); it is never a candidate.
- `provisionAgentRepo` already runs git outside the host's configuration with no prompts; the guard's reads (`git status --porcelain`, the unpushed-commit read, the stash list) follow the same discipline.
- `setRepos`' refusals are worded for the operator, who reads them on the card; the guard's refusals follow that voice.

## The Fix

One verb, `FleetManager.removeRepoCheckout({id, repoSlug})`: the repository must already be unlisted (the two-step rule: unlist first, delete second) and must not be the working repository; the derived path must hold a valid checkout (`inspectAgentRepo`); the guard reads the work tree and refuses with a reason the operator can act on when any of these holds: changed or untracked files (`git status --porcelain` not empty), commits on any local branch not on a remote (`git log --branches --not --remotes` not empty), stash entries. Only then the folder is removed, and the verb returns `{removed: true, repoPath}`. The guard does not know which folders a running harness has open; the ticket states that limit rather than inventing a check.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetManager.removeRepoCheckout({id, repoSlug})` (new, wire-dispatched) | `FleetManager` fleet authority; `deriveAgentRepoPath` for the target; `inspectAgentRepo` for the state | Refuses unless the repository is unlisted and not the working one; refuses with a worded reason on a dirty tree, unpushed commits or stashes; otherwise removes the derived path and returns `{removed: true, repoPath}` | Unknown seat → `null`; no checkout at the path → `{removed: false, reason}`; a foreign occupant → refused (never deleted); the read of the tree happens through an injectable seam so units run without git | Verb JSDoc names the three guard reads and the open-folder limit | Unit arms per refusal, one removal arm; an integration arm with a real temporary repository |
| `metadata.repos` | unchanged | Unlisting stays `setRepos`' job | — | — | existing arms |

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 Red first: a listed repository, the working repository, a dirty tree, an unpushed local commit and a stash each refuse with their own operator-worded reason; nothing is removed.
- [ ] AC-2 An unlisted, clean, fully pushed checkout is removed; the verb returns its derived path; a path outside the managed root can never be the target (the derivation refuses).
- [ ] AC-3 A foreign occupant at the derived path is refused, not deleted.
- [ ] AC-4 The dispatch spec lists the verb; Institution #642 consumes it.

## Out of Scope

Archiving or moving a checkout instead of deleting it; checking which folders a running harness has open (stated as a limit); the Accounts affordance (Institution #642).

## Avoided Traps

- Deleting on unlist: the two steps stay two; an operator who only wants the seat to stop launching with a repository loses nothing.
- Trusting a path from the wire: the target is derived from the seat id and slug, as every checkout path is.
- A guard that only reads `git status`: unpushed commits and stashes are work too.

## Related

Parent #571; sibling #950 (the runtime preparation verb); consumer: Institution #642 (the Repositories card); Institution #407.

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-09 12:34Z; no equivalent. A2A in-flight claim sweep: none. Memory Core rationale sweep: no prior decision on removing a seat's checkout; the retain-on-unlist rule is documented on the card's container.

Origin Session ID: 4be92a90-698e-4ba2-b207-47b79f518bd3

Retrieval Hint: "removeRepoCheckout guard dirty untracked unpushed stash unlist first"

## Timeline

- 2026-10-09T12:36:06Z @neo-fable-clio added the `enhancement` label
- 2026-10-09T12:36:06Z @neo-fable-clio added the `ai` label
- 2026-10-09T12:36:06Z @neo-fable-clio added the `agent-os` label
- 2026-10-09T12:36:53Z @neo-fable-clio added parent issue #571
- 2026-10-09T12:37:32Z @neo-fable-clio cross-referenced by #950

