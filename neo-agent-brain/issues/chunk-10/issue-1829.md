---
id: 1829
title: 'PAT admission guards key on the provider subject, not the login'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-17T15:35:28Z'
updatedAt: '2026-11-17T20:48:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1829'
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
blockedBy: []
blocking: []
closedAt: '2026-11-17T20:48:06Z'
---
# PAT admission guards key on the provider subject, not the login

**Split from** #1812 on 2026-11-17: its Fix item 5, the two guard rows of its Contract Ledger, and AC-8 and AC-9 move here unchanged in substance. The design is D825's (neomjs/neo-agent-institution#825), graduated with #1812; this leaf graduates nothing new. The parent is the epic natively linked to this issue.

**Decision Record:** Required, through #1812: ADR 0038 §2.2 fact 5, which neomjs/neo-agent-institution#1011 landed. **Decision Record impact:** aligned-with ADR 0038 §2.2; `auth.allowedUsers` and `auth.pinFirstProviderSubject` are AiConfig leaves, so also aligned-with `adr:aiconfig-reactive-provider-ssot` (ADR 0019).

Design authority: ADR 0038 §2.2 calls the forge login "a DISPLAY/projection fact only". D825 adds fact 5, the identity a request acts as, which the verified subject selects.

## Context

#1812 moves Memory Core's acting-identity selection from the login to the verified provider subject `{authProvider, providerBaseUrl, providerUserId}`. The PAT admission guards in `AuthService` sit in front of every PAT-mode server, including servers that bind no graph identity, and they still key on the login. A forge releases a renamed login for anyone to register, and that is legitimate (operator, 2026-11-16). A login-keyed guard therefore admits whoever holds the login now. This is a source-derived exposure class, not an observed incident.

It is a separate leaf because it is a separate surface. The guards live in the shared `AuthService` with their own config leaves, and #1513 waits only on #1812's relation. Each PR is then reviewed on one surface.

## The Problem

At Brain `92f0e736`, in `ai/mcp/server/shared/services/AuthService.mjs`:
- The GitHub verifier's AuthInfo sets `userId: user.login` (`:1349`), and the per-process first-use pin compares `info.userId` (`admitProviderSubject`, `:1387` and `:1395`). The pin keys on the login.
- `auth.allowedUsers` admits by login: GitHub at `:1484`, its disk tier at `:1251`, and GitLab by `user.username` at `:1009`.
- The GitLab verifier has no pin. `pinFirstProviderSubject` is read only by the GitHub path (`:1109`, `:1149`) and the policy check (`:775`).
- The outage disk tier admits a cached entry by its token hash, after an allowlist check on its login (`admitFromDiskWhenProviderAgrees`, `:1235`). An entry without `user.id` still admits (`:1303`), with no subject.

## The Architectural Reality

- Both verifiers already carry the subject: `providerUserId` (GitLab `:965`, GitHub `:1365`) and `providerBaseUrl`. The disk tier persists `user: {id, login, name}` (`:1511`).
- `auth.allowedUsers` and `auth.pinFirstProviderSubject` are AiConfig leaves. A shape change is declared on the leaf and read at the use site (ADR 0019 §5).
- Placement, per `npm run ai:structure-map -- --files --loc` at `92f0e736`: `ai/mcp/server/shared/services/`. No new folder.

## The Fix

- `auth.allowedUsers` accepts subject-pinned entries (login plus `providerUserId`) as the admission anchor for servers that bind no graph identity.
- Bare-login entries stay, documented as the weaker legacy profile.
- The per-login first-use pin keys on the subject tuple. It stays only as defense in depth beside the bootstrap process pin; its removal stays with #83.
- The GitLab verifier gets the same pin, and a disk-cache entry without `user.id` fails closed.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `auth.allowedUsers` entry | ADR 0038 §2.2 | `{login, providerUserId}` pins the admission anchor | A bare login stays the documented weaker profile | `AuthService.mjs` JSDoc | AC-1 |
| `pinFirstProviderSubject`, GitLab verifier, outage disk cache | D825 OQ4 | The pin keys on the subject tuple, GitLab is pinned, and a cache entry without `user.id` fails closed | The process pin is unchanged | `AuthService.mjs` JSDoc | AC-2 |

## Discussion Criteria Mapping & Leaf Slice

From #1812, graduated from neomjs/neo-agent-institution#825: Fix item 5, the `auth.allowedUsers` and pin/GitLab/disk-cache Contract Ledger rows, AC-8 (AC-1 here) and AC-9 (AC-2 here). No binding inventory moves: `d825-subject-key` and `d825-former-login-readers` stay with #1812.

## Acceptance Criteria

- [ ] AC-1 (was #1812 AC-8): A bare-login allowlist entry is shown to be weaker than a subject-pinned one on a fresh process, on a cache miss, and at a reclaimed login's first presentation.
- [ ] AC-2 (was #1812 AC-9): The pin keys on the subject tuple, the GitLab verifier is pinned, and a disk-cache entry without `user.id` fails closed.

## Out of Scope

- Memory Core's acting-identity selection, the relation and its founding guard: #1812.
- Removing the bootstrap process pin: #83.
- Stdio and OIDC transports, `NEO_AGENT_IDENTITY` and `proxy-header`: the transports leaf.

## Related

Split from #1812. #783 deferred re-keying the pin as a consumer row; this leaf is that follow-up. Sibling streams: #1296, #1297, #875, #83.

Live latest-open + closed sweep:
- Read the latest 20 open Brain issues at 2026-11-17T15:34Z: 155 open, red band, met by filing under the owning epic.
- Ran open and closed searches for `allowedUsers`, `pinFirstProviderSubject`, `provider subject pin`, `GitLab verifier pin` and `PAT disk cache user.id`.
- Read #1813's Out of Scope (it leaves HTTP PAT to #1812) and closed #783's.
- No equivalent open owner and no prior exclusion found.

A2A in-flight claim sweep: the latest 30 messages in all read-states, at 15:35Z, carry no claim on these guards.

MC sweep: one query on the problem's nouns ("PAT admission allowlist and first-use pin keyed on login; released login re-registered admitted") returned 6 results, none on these guards.

Own-assignment sweep: 6 open issues. #1812 is this split's source, and none of the others overlaps.

Retrieval Hint: "D825 PAT admission guards subject-pinned allowedUsers first-use pin GitLab verifier disk cache"

## Timeline

- 2026-11-17T15:35:28Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-17T15:35:30Z @neo-opus-ada added the `enhancement` label
- 2026-11-17T15:35:30Z @neo-opus-ada added the `ai` label
- 2026-11-17T15:35:30Z @neo-opus-ada added the `architecture` label
- 2026-11-17T15:35:31Z @neo-opus-ada added the `security` label
- 2026-11-17T15:35:31Z @neo-opus-ada added the `agent-os` label
- 2026-11-17T15:35:50Z @neo-opus-ada added parent issue #1815
- 2026-11-17T15:37:21Z @neo-opus-ada cross-referenced by #1812
- 2026-11-17T15:43:21Z @neo-opus-ada cross-referenced by PR #1830
- 2026-11-17T15:54:33Z @neo-opus-ada cross-referenced by PR #1831
- 2026-11-17T18:47:06Z @neo-opus-ada referenced in commit `ee3a540` - "fix(auth): strict allowedUsers, endpoint-qualified disk rows (#1829)

auth.allowedUsers must resolve to a list (or a legacy comma string) whose entries are each a login, login:providerUserId, or exactly {login, providerUserId}; anything else refuses setup before either PAT verifier mounts, instead of opening the gate or coercing an entry. Disk cache rows now record the provider endpoint that verified them, and a row without it, or from another endpoint, fails closed and leaves the store."
- 2026-11-17T19:04:59Z @neo-opus-ada referenced in commit `e99404a` - "test(auth): the memory-core verifier fixture carries allowedUsers (#1829)

A hand-built auth subtree now carries the leaf's default allowedUsers: [], since a missing leaf refuses setup instead of opening the gate."
- 2026-11-17T20:24:03Z @neo-opus-ada referenced in commit `502eeaf` - "test(auth): allowlist carriers stay out of the shared config (#1829)

The allowlist test built 21 RootConfigBase providers and never destroyed
them. Each joins the Tier-1 singleton as its child (getParent), and its
formulas write through setData into the Tier-1 keys they find there. While
they live, the next flip of the chroma test selectors leaves
engines.chroma.useTestDatabase true, so dataDir stays on the temp dir and
DestructiveOperationGuard's KB arm lets the production delete through.

Each carrier now holds only the auth.allowedUsers leaf (the base
declaration for the env arm) and is destroyed after the read.

Measured in one worker, NEO_TEST_SKIP_CI=1: a BaseServer.spec test (loads
the Tier-1 singleton), this test, then the KB arm. Red 3/3 at e99404ac,
green 3/3 here."
- 2026-11-17T20:48:06Z @tobiu referenced in commit `749838c` - "feat(auth): PAT admission guards key on the provider subject (#1829) (#1831)

* feat(auth): PAT admission guards key on the provider subject (#1829)

auth.allowedUsers accepts subject-pinned entries (login:providerUserId, or {login, providerUserId} in an overlay) that admit by account id; a bare login stays the weaker legacy form and a malformed entry refuses setup. The first-use pin keys on the subject tuple in one shared factory, GitLab-PAT gains the same pin as an opt-in, and a disk-cache entry without user.id fails closed and leaves the store.

* fix(auth): strict allowedUsers, endpoint-qualified disk rows (#1829)

auth.allowedUsers must resolve to a list (or a legacy comma string) whose entries are each a login, login:providerUserId, or exactly {login, providerUserId}; anything else refuses setup before either PAT verifier mounts, instead of opening the gate or coercing an entry. Disk cache rows now record the provider endpoint that verified them, and a row without it, or from another endpoint, fails closed and leaves the store.

* test(auth): the memory-core verifier fixture carries allowedUsers (#1829)

A hand-built auth subtree now carries the leaf's default allowedUsers: [], since a missing leaf refuses setup instead of opening the gate.

* test(auth): allowlist carriers stay out of the shared config (#1829)

The allowlist test built 21 RootConfigBase providers and never destroyed
them. Each joins the Tier-1 singleton as its child (getParent), and its
formulas write through setData into the Tier-1 keys they find there. While
they live, the next flip of the chroma test selectors leaves
engines.chroma.useTestDatabase true, so dataDir stays on the temp dir and
DestructiveOperationGuard's KB arm lets the production delete through.

Each carrier now holds only the auth.allowedUsers leaf (the base
declaration for the env arm) and is destroyed after the read.

Measured in one worker, NEO_TEST_SKIP_CI=1: a BaseServer.spec test (loads
the Tier-1 singleton), this test, then the KB arm. Red 3/3 at e99404ac,
green 3/3 here."
- 2026-11-17T20:48:07Z @tobiu closed this issue

