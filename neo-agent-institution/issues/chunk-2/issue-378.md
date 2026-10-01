---
id: 378
title: The installed Fleet Manager keeps agent seats inside its app data
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - architecture
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T11:04:35Z'
updatedAt: '2026-10-01T12:10:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/378'
author: neo-opus-grace
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
closedAt: '2026-10-01T12:10:25Z'
---
# The installed Fleet Manager keeps agent seats inside its app data

## Context

Operator ruling, 2026-10-01 (neomjs/neo-agent-brain#571, issuecomment-5929565535): agent seats use one default everywhere, `~/.neo-ai/agents`, this machine included. Measured the same day:
- The installed Fleet Manager's app data is 2.2 GB, and all of it is the two seats: `neo-gpt-sophie` 1.6 GB and `neo-opus-ada` 593 MB.
- Four pre-install backups of that data hold another 3.6 GB.
- One seat's clone holds a local commit that exists nowhere else.

## The Problem

`harness/brain.mjs:586` (`buildPackagedBrainEnv`) sets `NEO_FLEET_AGENTS_ROOT` to `<dataRoot>/fleet/agents`. That overrides two things:
- The Brain leaf's default, `~/.neo-ai/agents`, at `ai/configBase.mjs:307`. The leaf is `planeMember: false`: "its working trees and path-keyed memory must outlive any plane".
- The Brain's deployment cookbook (`learn/agentos/DeploymentCookbook.md:301`).

The consequences:
- The seats share the app's lifetime: an uninstall cleaner or a fresh-profile reset takes them, unpushed work included.
- Every backup of the app data copies them, login profiles included.

The value was never a placement decision. #302 carried the retired `instanceRoot` placement over to the renamed leaf.

## The Architectural Reality

- `harness/brain.mjs:829` merges the packaged fragment over `process.env`, and no first-run key names the root. The installed app therefore takes a new root only with a rebuild. Clio measured this on 2026-10-01.
- The smoke profile (`buildBrainProfile`, `:429`) binds the root under its throwaway isolation root, and `test/playwright/unit/harness/brain.spec.mjs:270-275` asserts it. That must stay.
- ADR 0019 §10.9: an artifact that must outlive the plane is not a plane member. The leaf's default already resolves outside the plane and outside any checkout, and AiConfig remains the SSOT for its value.

## The Fix

- `buildPackagedBrainEnv` stops setting `NEO_FLEET_AGENTS_ROOT`, so the Brain's leaf default applies. An operator's own `NEO_FLEET_AGENTS_ROOT` still wins, because the fragment no longer shadows it.
- The comment above it states where seats live.
- `brain.spec` gains an arm: the packaged fragment does not place the root. The smoke arm keeps its isolated root.

## Acceptance Criteria

- [ ] AC-1: `buildPackagedBrainEnv` returns no `NEO_FLEET_AGENTS_ROOT` (unit; red on dev).
- [ ] AC-2: The smoke profile still binds `NEO_FLEET_AGENTS_ROOT` under its isolation root (the existing arm stays green).
- [ ] AC-3 `[L4-deferred — operator handoff needed]` (post-merge, installed): After the next repackage, Sophie's and Ada's seats move to `~/.neo-ai/agents/<id>/…` by #571's recipe: push first, copy Claude memories for the new path, and sign in after the move. The roster shows both, and the app data holds no seat folder. Owner after merge: #7; recipe and moves: neomjs/neo-agent-brain#571.

## Out of Scope

- The setup wizard's placement question (#351).
- Moving the other agents' seats (neomjs/neo-agent-brain#571).
- The extraction neomjs/neo-agent-brain#669 needs before the installed app is replaced. That ordering binds the repackage, not this change.

## Related

#302 · #345 · #351 · #7 · neomjs/neo-agent-brain#571 · neomjs/neo-agent-brain#669

Live latest-open sweep: the latest 20 open Institution issues at 2026-10-01T11:03Z, plus a search for agents root, `AGENTS_ROOT`, app data and userData seats: no equivalent (#302 and #345 are closed predecessors). A2A: Clio handed the Institution change to me (2026-10-01 10:34Z / 10:39Z). Own assignments: none on this surface.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364

Retrieval Hint: "installed Fleet Manager seats inside app data" · "NEO_FLEET_AGENTS_ROOT packaged override"

## Timeline

- 2026-10-01T11:04:36Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T11:04:37Z @neo-opus-grace added the `bug` label
- 2026-10-01T11:04:37Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T11:04:38Z @neo-opus-grace added the `ai` label
- 2026-10-01T11:04:38Z @neo-opus-grace added the `architecture` label
- 2026-10-01T11:08:05Z @neo-opus-grace cross-referenced by PR #379
- 2026-10-01T11:39:55Z @neo-opus-grace cross-referenced by #380
- 2026-10-01T11:40:19Z @neo-fable-clio cross-referenced by #351
- 2026-10-01T11:44:47Z @neo-opus-ada cross-referenced by #571
- 2026-10-01T12:10:25Z @tobiu referenced in commit `275b148` - "fix(harness): the installed Fleet Manager leaves agent seats on the Brain's per-user default (#378) (#379)

The packaged profile placed NEO_FLEET_AGENTS_ROOT under its own data root, so every
agent's clones and harness homes lived and died with the app's data and filled every
backup of it (2.2 GB of app data, all of it two seats; 3.6 GB of pre-install backups).
The value came across in #302 from the retired instanceRoot, not from a decision.

buildPackagedBrainEnv now places the seats only when its caller must contain them,
which the packaged smoke does inside its throwaway root. The installed app leaves the
Brain's fleet.agentsRoot default, ~/.neo-ai/agents, in force: the operator's ruling of
2026-10-01, one default for everyone."
- 2026-10-01T12:10:25Z @tobiu closed this issue

