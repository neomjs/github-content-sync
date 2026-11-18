---
id: 1813
title: Every provider-authenticated transport carries the subject
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-16T17:09:23Z'
updatedAt: '2026-11-17T23:05:21Z'
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
  - '[x] 1812 A protected relation binds each provider subject to its acting identity'
blocking: []
closedAt: '2026-11-17T23:05:21Z'
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

## Intake (2026-11-17, @neo-opus-ada)

**Verdict:** valid-as-written, accepted and sharpened below. The drift probe was non-empty (`Server.mjs` moved with #1812's PR, this leaf's prerequisite), so the full gate ran. #1815 is self-authored. The only blocker, #1812, is closed. ADR 0038 §2.2 fact 5 landed in Institution `dffad4da` after this ticket was filed, and it matches the ticket.

**Premise at Brain `23c077f8`:** `StdioIdentityResolver.resolve()` drops the fetched `id` and returns `{githubLogin, username, source}`. `Server.resolveStdioIdentity()` binds every stdio source by login (`bindAgentIdentity`), even while selection is active. HTTP PAT selects by subject (`buildRequestContext` → `selectAgentIdentityBySubject`).

Prescription checked: `ai/mcp/server/memory-core/Server.mjs` (`selectAgentIdentityBySubject`, `readSubjectSelection`) — owns the concern; stdio reuses it with `create: false`.
Prescription checked: `ai/mcp/server/shared/services/StdioIdentityResolver.mjs` — owns carrying the subject.

**Sharpened:**
1. The `gh-cli` subject's `providerBaseUrl` is the endpoint that `gh api user` answered from: its `url` without `/users/<login>` and without a trailing slash. It is not `auth.githubApiBaseUrl`. A `gh` signed in to another host therefore keys its own subject, never a github.com binding. On github.com it equals `forgeApiBase(auth, 'github')`, so a stdio admission and an HTTP PAT admission of one account key one subject.
2. Once selection is active, a stdio `gh-cli` boot fails by name when its subject is incomplete, unbound, or mapped by the founding record. These are `selectAgentIdentityBySubject`'s refusals. Nothing falls back to the login or to single-tenant mode. While dormant, the login binding is unchanged.
3. `oidc` (AC-2) maps onto the same tuple: `{authProvider: 'oidc', providerBaseUrl: iss, providerUserId: sub}`. That half is read against #1831's `AuthService` when it starts.
4. **Sequencing:** #1831 rewrites `AuthService.mjs`. The stdio arms (AC-1, AC-3) start on `dev`, and the `AuthService` arms (AC-2, AC-4, AC-5) follow after #1831 merges. One PR resolves this ticket.


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
- 2026-11-17T15:35:29Z @neo-opus-ada cross-referenced by #1829
- 2026-11-17T18:01:12Z @neo-gpt-euclid cross-referenced by PR #1830
- 2026-11-17T20:33:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-17T20:55:55Z @neo-opus-ada referenced in commit `2eb5b44` - "feat(memory-core): stdio gh-cli selects its identity by subject (#1813)

StdioIdentityResolver's gh-cli arm now carries the subject its `gh api
user` payload proves: provider github, the API base its own url names,
and the account id. A field the payload lacks stays absent. The
configured API base never stands in, since gh may be signed in to
another host.

Once the cutover activates selection, Server.resolveStdioIdentity acts
as the identity that subject is bound to (selectAgentIdentityBySubject,
create: false), keyed by its bare id. A subject that selects none fails
the boot by name; there is no login fallback.

NEO_AGENT_IDENTITY stays host-trusted: it carries no subject, keeps the
login binding, and never reads the activation (ADR 0038 2.2 fact 5)."
- 2026-11-17T20:55:56Z @neo-opus-ada referenced in commit `03ca849` - "feat(auth): an OIDC request carries its (issuer, sub) subject (#1813)

The OIDC verifier stamps provider oidc, the issuer as providerBaseUrl and
sub as providerUserId (oidcSubject). The issuer is the introspection's
iss, else the one discovery named. When both exist they are compared
exactly, and a mismatch leaves no issuer, so selection refuses the
incomplete subject by name once active. preferred_username stays the
login: it selects only while selection is dormant, and nothing binds
ahead of the gate, since oidc is no create source by default.

Fleet resolves an owner only for forge providers (ADR 0038 2.2 fact 2).
An OIDC subject reaching createFleetRequestContext resolves none, as
before, rather than a forge-registry refusal reason.

The trusted proxy's host-trusted shape is now pinned exactly."
- 2026-11-17T20:57:46Z @neo-opus-ada referenced in commit `e0f4be8` - "chore(auth): state the subject rules in comments, not ADR refs (#1813)"
- 2026-11-17T20:58:57Z @neo-opus-ada cross-referenced by PR #1833
- 2026-11-17T21:31:00Z @neo-opus-ada referenced in commit `5adee70` - "fix(auth): qualify OIDC and stdio subjects; keep dormant and Fleet readers (#1813)"
- 2026-11-17T21:47:46Z @neo-opus-ada referenced in commit `cc69c29` - "test(auth): build the OIDC proxy config with its overrides (#1813)

`Object.assign(aiConfig.auth, authOverrides)` counted as a shared-config
write under ADR-0019 B4 and put TransportService.spec above its ceiling
of 0. The helper now builds the auth block with the overrides, so the
spec writes no config object."
- 2026-11-17T22:16:08Z @neo-opus-ada referenced in commit `147466b` - "fix(auth): only an omitted OIDC iss takes the discovery issuer (#1813)

`oidcSubject` read a present but malformed `iss` ('', null, a number) as
omitted, so it took the qualified discovery issuer. Only an absent `iss`
is covered now; any present value must equal the qualified issuer."
- 2026-11-17T23:05:21Z @tobiu referenced in commit `d172de4` - "feat(auth): provider-authenticated transports carry the subject (#1813) (#1833)

* feat(memory-core): stdio gh-cli selects its identity by subject (#1813)

StdioIdentityResolver's gh-cli arm now carries the subject its `gh api
user` payload proves: provider github, the API base its own url names,
and the account id. A field the payload lacks stays absent. The
configured API base never stands in, since gh may be signed in to
another host.

Once the cutover activates selection, Server.resolveStdioIdentity acts
as the identity that subject is bound to (selectAgentIdentityBySubject,
create: false), keyed by its bare id. A subject that selects none fails
the boot by name; there is no login fallback.

NEO_AGENT_IDENTITY stays host-trusted: it carries no subject, keeps the
login binding, and never reads the activation (ADR 0038 2.2 fact 5).

* feat(auth): an OIDC request carries its (issuer, sub) subject (#1813)

The OIDC verifier stamps provider oidc, the issuer as providerBaseUrl and
sub as providerUserId (oidcSubject). The issuer is the introspection's
iss, else the one discovery named. When both exist they are compared
exactly, and a mismatch leaves no issuer, so selection refuses the
incomplete subject by name once active. preferred_username stays the
login: it selects only while selection is dormant, and nothing binds
ahead of the gate, since oidc is no create source by default.

Fleet resolves an owner only for forge providers (ADR 0038 2.2 fact 2).
An OIDC subject reaching createFleetRequestContext resolves none, as
before, rather than a forge-registry refusal reason.

The trusted proxy's host-trusted shape is now pinned exactly.

* chore(auth): state the subject rules in comments, not ADR refs (#1813)

* fix(auth): qualify OIDC and stdio subjects; keep dormant and Fleet readers (#1813)

* test(auth): build the OIDC proxy config with its overrides (#1813)

`Object.assign(aiConfig.auth, authOverrides)` counted as a shared-config
write under ADR-0019 B4 and put TransportService.spec above its ceiling
of 0. The helper now builds the auth block with the overrides, so the
spec writes no config object.

* fix(auth): only an omitted OIDC iss takes the discovery issuer (#1813)

`oidcSubject` read a present but malformed `iss` ('', null, a number) as
omitted, so it took the qualified discovery issuer. Only an absent `iss`
is covered now; any present value must equal the qualified issuer."
- 2026-11-17T23:05:22Z @tobiu closed this issue

