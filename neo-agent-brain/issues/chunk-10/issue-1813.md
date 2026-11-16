---
id: 1813
title: Every provider-authenticated transport carries the subject
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees: []
createdAt: '2026-11-16T17:09:23Z'
updatedAt: '2026-11-16T17:09:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1813'
author: neo-opus-ada
commentsCount: 0
parentIssue: 1815
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1812 A protected relation binds each provider subject to its acting identity'
blocking: []
---
# Every provider-authenticated transport carries the subject

**Graduated from** neomjs/neo-agent-institution#825 (row H, Maryam's proposal, adopted) at the body edited 2026-11-16T16:33:07Z. The parent is the epic natively linked to this issue.

**Decision Record:** Required (ADR 0038 §2.2 fact 5). **Decision Record impact:** depends-on its Institution leaf.

Design authority: OIDC Core §5.7 treats issuer + subject as the only stable identifier; `preferred_username` is not one. D825 concept point 4 makes a missing subject nonqualifying, and this leaf makes that hold on every transport.

## Context

#1812 binds the acting identity by provider subject on HTTP PAT. The other transports still bind by login, so a subject-bound identity could still be reached through a login-only path.

## The Problem

At Brain `436aec63`:
- **Stdio.** Both stdio paths bind by login (`Server.mjs:733-746`).
  - The `NEO_AGENT_IDENTITY` env path has no provider id.
  - The `gh-cli` path drops the `id` that `gh api user` returns (`StdioIdentityResolver.mjs:69-128`).
- **OIDC.** The default `oidc` mode binds `preferred_username` when present and keeps no issuer (`AuthService.mjs:340-353`).
- **Proxy.** The proxy middleware binds a username header with no subject (`:419-431`).

Requiring a live subject everywhere, with no local carve-out, would fail env-pinned boots closed.

## The Fix

Each transport carries the subject or is named as host-trusted:
<!-- binding-inventory: d825-transports -->
- `HTTP PAT` — carries the subject; its binding is #1812
- `stdio gh-cli` — keeps the `id` it already fetches and binds through #1812's relation
- `oidc (issuer, sub)` — the issuer is compared exactly; `forgeApiBase` normalization never applies to an issuer
- `NEO_AGENT_IDENTITY` — host-trusted local stdio; no subject-bound proof unless a protected binding corroborates it
- `proxy-header` — host-trusted; no subject-bound proof
<!-- /binding-inventory -->

Placement: `ai/mcp/server/shared/services/` (`StdioIdentityResolver.mjs`, `AuthService.mjs`) and `ai/mcp/server/memory-core/Server.mjs` (structure map at `08fba722`).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `resolveStdioIdentity` (`gh-cli` path) | ADR 0038 §2.2 fact 5 | Carries the fetched `id` and binds by subject | No `id`: nonqualifying, refused by name | `StdioIdentityResolver.mjs` JSDoc | AC-1 |
| `oidc` mode | OIDC Core §5.7, §3.1.3.7, §16.15 | Keys on `(issuer, sub)`; issuer compared exactly | Issuer mismatch refuses; no login fallback | `AuthService.mjs` JSDoc + ADR | AC-2 |
| `NEO_AGENT_IDENTITY`, `proxy-header` | ADR 0038 §2.2 | Host-trusted, never subject-bound proof | Unchanged boot behaviour | ADR | AC-3, AC-4 |

## Acceptance Criteria

- [ ] AC-1: `stdio gh-cli` resolves the subject from the fetched `id` and binds through #1812's relation, with no login fallback.
- [ ] AC-2: `oidc` keys on `(issuer, sub)`. The same `sub` under a different issuer is a different subject, and an issuer mismatch refuses. Neither falls back to the login.
- [ ] AC-3: `NEO_AGENT_IDENTITY` still boots env-pinned harnesses, and it never counts as subject-bound proof without a protected binding.
- [ ] AC-4: `proxy-header` never counts as subject-bound proof.
- [ ] AC-5: An existing `oidc` deployment bound by `preferred_username` stays nonqualifying until its cutover bind (the cutover leaf). Nothing here activates ahead of the epic's gate.

## Out of Scope

- HTTP PAT binding and the relation itself, which belong to #1812.
- The cutover inventory and bind, which belong to the cutover leaf.

## Avoided Traps

- Requiring a live subject on every path with no local carve-out, which fails env-pinned boots closed.
- Normalizing an OIDC issuer like a forge API base.

Owner: unowned-rationale. Claimable by any maintainer; if it is still unclaimed when #1812 merges, the epic steward (`@neo-opus-ada`) takes it.

Live latest-open + closed sweep: the same run as #1812 at 2026-11-16T17:06Z. That covered the latest 20 open Brain issues (156 open) and open and closed searches for `provider subject identity`, `acting identity login`, `former login alias` and `providerUserId binding`. No equivalent was found. The A2A claim sweep found no competing claim; the own-assignment sweep found no overlap.

Origin Session ID: 00199bfa-0a61-420b-a86e-496e506fe51e

Retrieval Hint: "D825 row H transports stdio gh-cli oidc issuer subject"

## Timeline

- 2026-11-16T17:09:25Z @neo-opus-ada added the `enhancement` label
- 2026-11-16T17:09:25Z @neo-opus-ada added the `ai` label
- 2026-11-16T17:09:25Z @neo-opus-ada added the `architecture` label
- 2026-11-16T17:09:26Z @neo-opus-ada added the `security` label
- 2026-11-16T17:09:26Z @neo-opus-ada added the `agent-os` label
- 2026-11-16T17:10:07Z @neo-opus-ada cross-referenced by #1010
- 2026-11-16T17:11:01Z @neo-opus-ada cross-referenced by #1815
- 2026-11-16T17:11:15Z @neo-opus-ada added parent issue #1815
- 2026-11-16T17:11:29Z @neo-opus-ada marked this issue as being blocked by #1812

