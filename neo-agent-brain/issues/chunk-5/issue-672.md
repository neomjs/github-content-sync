---
id: 672
title: 'The Fleet creates its agents root with the umask, not owner-only'
state: CLOSED
labels:
  - bug
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T12:19:00Z'
updatedAt: '2026-10-01T12:54:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/672'
author: neo-opus-grace
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
closedAt: '2026-10-01T12:54:39Z'
---
# The Fleet creates its agents root with the umask, not owner-only

## Context

#571 now requires the agents root to be created `0700`, "by Fleet and by the move recipe alike": harness homes hold credentials (Claude and Codex logins, Codex's `cli_auth_credentials_store = "file"`). Under app data they inherited owner-only access from `~/Library` (`0700`). The operator's 2026-10-01 ruling moves every seat to `~/.neo-ai/agents`, and Institution #379 (merged) stops the packaged app from overriding that default. There, nothing inherits: `~` is `0750` (group `staff`) and `~/.neo-ai` is `0755` (measured by @neo-opus-ada, defect-note 2026-10-01T11:46Z). The move recipe's step (`mkdir -m 700`) is @neo-fable-clio's; this ticket is the Fleet half.

## The Problem

A fresh Start creates the root with the process umask: `ensureAgentRepo` → `provisionAgentRepo` runs `git clone` into `<root>/<agentId>/<owner>/<repo>`, which creates every missing ancestor `0755`, the root included. A root that already exists with group or other access is used as it is.

## The Architectural Reality

- `ai/services/fleet/ensureAgentRepo.mjs` runs on every Fleet Start before anything is written under the root: every cockpit-added seat carries a repo (`AddAgentFlow.repoOf` in the Institution defaults the slug). `managedRoot` and the lifecycle's `getInstanceRoot()` both resolve `AiConfig.fleet.agentsRoot`, so the harness homes share the root.
- Precedent for the shape: `ai/daemons/wake/receiverState.mjs:67-69` creates its state directories `0700` and `chmod`s existing ones to `0700`.
- Sibling single-purpose modules: `ensureAgentRepo.mjs`, `deriveAgentInstanceHome.mjs`, `provisionAgentRepo.mjs`.

## The Fix

`ai/services/fleet/ensureSeatRoot.mjs` exports `ensureSeatRoot(root)`: create the root `0700` (ancestors keep their default), or narrow an existing root that grants group or other access. `ensureAgentRepo` calls it on the managed root after `deriveAgentRepoPath` has validated it and before it inspects or clones.

## Acceptance Criteria

- [ ] AC-1: a missing root is created `0700`; its missing ancestors are created with the default mode.
- [ ] AC-2: an existing root with group or other access is `0700` after the call; an owner-only root is untouched.
- [ ] AC-3: `ensureAgentRepo` secures the managed root before it inspects or clones; a root that is not a directory, or cannot be narrowed, fails the call and no clone runs.

## Out of Scope

- The move recipe's `mkdir -m 700` (#571, Clio).
- File and directory modes below the root: a `0700` root closes traversal to all of them.
- Seats without `metadata.repo`: the cockpit never creates one, and the harness creates their home at spawn.
- Windows ACLs.

## Decision Record impact

`none`. Implements #571's placement requirement.

## Related

Parent #571. Institution #379 (the packaged app stops overriding the root). #662 (`writeSeatLease`).

## Sweeps

Live latest-open sweep: latest 20 open Brain issues at 2026-10-01T12:18:34Z, no equivalent; `gh search issues` for "umask" and "0700 agents root": none. A2A in-flight sweep (latest 12:16Z): Ada's defect-note says no Brain sub is filed; no claim. MC sweep: "seat folders are group-readable; harness home holds credentials; agents root permissions outside app data", 6 results, no prior decision beyond #571. Own-assignment sweep: #659 and #670 touch other surfaces. Structure map (this session, exit 0): owning folder `ai/services/fleet`; one new sibling module.

Body revised 2026-10-01 ~12:30Z: the guard moved from a `startAgentProvisioned` seam into `ensureAgentRepo`, the code whose clone creates the root.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "Fleet agents root created with umask; ensureSeatRoot 0700; harness homes group-readable outside app data"

🖖 Grace · @neo-opus-grace · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-01T12:19:01Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T12:19:02Z @neo-opus-grace added the `bug` label
- 2026-10-01T12:19:02Z @neo-opus-grace added the `ai` label
- 2026-10-01T12:19:02Z @neo-opus-grace added the `security` label
- 2026-10-01T12:19:03Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T12:19:07Z @neo-opus-grace added parent issue #571
- 2026-10-01T12:24:51Z @neo-opus-grace cross-referenced by PR #673
- 2026-10-01T12:35:50Z @neo-opus-grace cross-referenced by #674
- 2026-10-01T12:38:36Z @neo-opus-ada cross-referenced by #675
- 2026-10-01T12:42:14Z @neo-opus-grace referenced in commit `d1e7b62` - "fix(fleet): secure the agents root as the derivation resolves it (#672)

ensureSeatRoot received the raw managedRoot while deriveAgentRepoPath resolves it lexically, so
`<dir>/link/../agents` secured the directory the symlink reaches and cloned into another one left
with the umask. Both now use assertRoot's resolution. The helper's JSDoc reports the ancestor
modes as this host's measurement."
- 2026-10-01T12:54:39Z @tobiu referenced in commit `48da7a1` - "fix(fleet): the agents root is owner-only before a seat is cloned into it (#672) (#673)

* fix(fleet): the agents root is owner-only before a seat is cloned into it (#672)

ensureAgentRepo makes the managed root 0700 (ensureSeatRoot) after the path derivation validates it
and before it inspects or clones: git clone created it with the umask, and outside app data nothing
above it keeps the harness homes private. An existing root open to group or other is narrowed, as
receiverState does for its state directories.

* fix(fleet): secure the agents root as the derivation resolves it (#672)

ensureSeatRoot received the raw managedRoot while deriveAgentRepoPath resolves it lexically, so
`<dir>/link/../agents` secured the directory the symlink reaches and cloned into another one left
with the umask. Both now use assertRoot's resolution. The helper's JSDoc reports the ancestor
modes as this host's measurement."
- 2026-10-01T12:54:39Z @tobiu closed this issue
- 2026-10-01T13:06:35Z @neo-fable-clio cross-referenced by #584

