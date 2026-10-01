---
id: 712
title: A seat's PAT is presented only to the forge host it was stored for
state: OPEN
labels:
  - enhancement
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T18:10:27Z'
updatedAt: '2026-10-01T18:10:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/712'
author: neo-opus-grace
commentsCount: 0
parentIssue: 684
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 710 A seat''s repository records its forge, and a GitLab slug may name nested groups'
blocking: []
---
# A seat's PAT is presented only to the forge host it was stored for

## Context

This is leaf 2 of #684, the Fleet's GitLab parity. Leaf 1 (#710, PR #711) records a repository's forge and allows nested GitLab slugs. This leaf makes a GitLab seat's clone authenticate. Leaf 3 (`NEO_GITLAB_*` injection, the plan, rendering) builds on both.

## The Problem

- **The clone authenticates GitHub only.** `gitCloneCommand` (`provisionAgentRepo.mjs:33-53`) presents the seat's PAT through a helper keyed on `https://github.com`. A private GitLab repository therefore clones without auth and fails.
- **A stored PAT does not record its host, so the clone cannot simply follow the repository.** The PAT is written only by `defineAgent`, which is create-only. The working repository changes later through `setRepo`, a wire verb (`FLEET_WIRE_METHODS`). If the clone presented the PAT to whatever host the working repository names, one wire call re-pointing a seat would send its PAT to a host of the caller's choosing on the next start. Today a GitHub PAT cannot leave `github.com`, and this leaf must keep that guarantee.

## The Architectural Reality

- **The registry row.** `defineAgent` writes the PAT to `credentials.enc` (`{agentId: pat}`, AES-256-GCM) and the definition to the registry row in one transaction (`FleetRegistryService.mjs:374-391`). The scoped verbs cannot reach a top-level row field:
  - `updateAgent`, the path `setRepo` and `setAvatar` use, spreads the existing row and patches only `metadata` and `modelProvider` (`:407-433`);
  - `configureAgent` allowlists `id`, `harnessType`, `mcpServers` and `mcpTarget` (`:461`).
- **`toPublic`** strips only credential and launch vocabulary (`PUBLIC_SENSITIVE_KEY_RE`), so a forge and its host stay visible to the cockpit.
- **The clone's username.** GitLab accepts any non-empty username beside a PAT and does not validate it (GitLab docs, "Personal access tokens"). The Brain's existing `x-access-token` therefore serves both forges, as it already does in KB ingestion (`gitMirror.mjs:212`). Only the helper's host changes.
- **The threading path:** `startAgentProvisioned` → `ensureAgentRepo` → `provisionAgentRepo` → the `cloneRepo` seam (`gitClone` → `gitCloneCommand`). The working repository and the extras (#683) all clone through it.

## The Fix

1. **`defineAgent` takes `forge` and `forgeHost`:**
   - GitHub (the default) records neither, so the PAT stays bound to `https://github.com` and existing rows are unchanged.
   - `gitlab` requires `forgeHost`: an `https` origin with no path, credentials, query or fragment. Both fields are written on the row beside the PAT. Nothing else writes them.
   - A host with `github` refuses, as does an unknown forge, using leaf 1's `REPO_FORGES`.
2. **`gitCloneCommand` takes the credential's origin** (default `https://github.com`). It presents the PAT only to an `https` clone URL with no userinfo whose origin equals it, with the helper keyed on that origin. The child's token variable becomes forge-neutral.
3. **`startAgentProvisioned` passes the row's `forgeHost`** through `ensureAgentRepo` and `provisionAgentRepo` for the working repository and every extra. An extra on another host clones unauthenticated, as a GitLab extra on a GitHub seat does today.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `defineAgent` `forge` / `forgeHost` (wire `defineAgent`) | `FleetRegistryService` | `gitlab` + an `https` origin, recorded on the row with the PAT | omitted: GitHub, nothing recorded | `defineAgent` JSDoc | registry unit arms |
| `gitCloneCommand(…, credentialOrigin)` | `provisionAgentRepo.mjs` | PAT presented only to that origin, over `https`, with no userinfo | `https://github.com` | JSDoc | `provisionAgentRepo.spec` matrix |
| `credentials.enc` | registry | unchanged (`{agentId: pat}`) | none | none | the existing registry arms stay green |

## Acceptance Criteria

- [ ] AC-1: a GitLab define records `forge` and `forgeHost` and never returns the PAT. A GitHub define writes the same row as before. A GitLab define without a host refuses and writes nothing, as do a host that is not a bare `https` origin, a host with `github`, and an unknown forge (unit).
- [ ] AC-2: `updateAgent` (the path `setRepo` and `setAvatar` use) and `configureAgent` leave `forge` and `forgeHost` as defined (unit).
- [ ] AC-3: a GitLab seat's PAT reaches only its own origin. It is not sent to `github.com` or to another GitLab host, and a GitHub seat's PAT still reaches `github.com` only. Covered by a `gitCloneCommand` matrix, plus a `startAgentProvisioned` arm proving the origin reaches every clone, extras included (unit).
- [ ] AC-4 `[L4-deferred — operator handoff needed]` (post-merge, installed): a GitLab seat clones a private repository on a self-hosted instance. This is the same instance as #684's installed arm, so the residual owner is #684.

## Out of Scope

- `NEO_GITLAB_*` injection, the plan admitting `gitlab-workflow`, harness rendering, and a working repository on a host other than the PAT's. These are leaf 3.
- Rotating a stored PAT, for which no surface exists today.
- `githubUsername` naming a GitLab seat's account.
- Instances served under a path, which leaf 1's slug rule refuses, and SSH clone URLs, which carry no PAT.
- The cockpit's GitLab form, Institution #245.

## Decision Record impact

`none`: per-seat registry data, with no config leaf. ADR 0019 governs leaf 3's injection.

## Related

Parent #684 · leaf 1 #710 / PR #711 (this leaf builds on `REPO_FORGES`) · #683 (extras) · Institution #245 · #659

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T18:09:45Z. No equivalent; #710 (leaf 1) and #684 (parent) are adjacent.
- Exact: `gh search issues --owner neomjs` for "GitLab clone", "forgeHost", "gitlab PAT host" and "clone credential host" found only #710.
- MC sweep: "private GitLab repository clone fails without auth; seat PAT sent to another host; credential bound to host; Fleet seat repository re-pointed", 6 results. They are #684's own record and Ada's history of the host-scoped helper (#591 / PR #592). No prior decision on binding.
- A2A: the 30 newest messages, all read-states, up to 18:09:45Z. No claim on this scope.
- Own-assignment: #684 (parent) and #710 (leaf 1); neither carries this.
- Structure map (this session, exit 0): `ai/services/fleet`. No new module.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "GitLab seat PAT bound to forgeHost clone credential origin gitCloneCommand defineAgent forge"

🖖 Grace (Claude Opus 5.5, Claude Code)


## Timeline

- 2026-10-01T18:10:28Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T18:10:28Z @neo-opus-grace added the `enhancement` label
- 2026-10-01T18:10:29Z @neo-opus-grace added the `ai` label
- 2026-10-01T18:10:29Z @neo-opus-grace added the `security` label
- 2026-10-01T18:10:29Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T18:10:40Z @neo-opus-grace added parent issue #684
- 2026-10-01T18:10:42Z @neo-opus-grace marked this issue as being blocked by #710
- 2026-10-01T18:21:27Z @neo-fable-clio cross-referenced by #713
- 2026-10-01T18:21:51Z @neo-fable-clio cross-referenced by #714
- 2026-10-01T18:24:47Z @neo-opus-grace cross-referenced by #684

