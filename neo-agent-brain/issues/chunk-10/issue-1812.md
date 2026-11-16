---
id: 1812
title: A protected relation binds each provider subject to its acting identity
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-16T17:09:04Z'
updatedAt: '2026-11-16T18:45:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1812'
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
  - '[ ] 1010 ADR 0038 §2.2 names the identity a request acts as'
blocking:
  - '[ ] 1813 Every provider-authenticated transport carries the subject'
  - '[ ] 1814 Cutover binds existing identities and retires former-login redirects'
  - '[ ] 1513 Use admitted families for routing and publication checks'
---
# A protected relation binds each provider subject to its acting identity

**Graduated from** neomjs/neo-agent-institution#825 at the body edited 2026-11-16T16:33:07Z. Quorum: Claude author signal, plus GPT and Gemini `[GRADUATION_APPROVED]`. The parent is the epic natively linked to this issue.

**Decision Record:** Required. ADR 0038 §2.2 gains fact 5 through its Institution leaf. **Decision Record impact:** depends-on that amendment.

Design authority: ADR 0038 §2.2 keeps four identity facts apart and calls the forge login "a DISPLAY/projection fact only". D825 adds fact 5, the identity a request acts as.

## Context

Today a request's acting identity is chosen by its login, and logins are reusable. GitHub releases a renamed login for anyone to register, and that is legitimate (operator, 2026-11-16: "anyone can register free accounts; nothing wrong with that"). Under provider-PAT auth, a plane that admits a released login binds whoever registers it to the identity that login still names. This is a source-derived exposure class, not an observed incident.

## The Problem

- `buildRequestContext` binds `@<login>` by lookup or provisioning (`ai/mcp/server/memory-core/Server.mjs:495-507` at Brain `d0481ce1`).
  - A seeded node is never compared with the request's `providerUserId`.
  - An auto-provisioned node takes each sign-in's provider fields without comparing them (`:637`).
- The guards key on the login: `auth.allowedUsers` (`AuthService.mjs:1379`) and `pinFirstProviderSubject`, which compares `info.userId` set from `user.login` (`:1245`, `:1290`).
- The GitLab verifier has no pin. The outage disk cache stores `user.id` (`AuthService.mjs:1405-1409` at `436aec63`).
- `validateMailboxTarget` resolves a former-login alias before it checks the exact graph id (`MailboxService.mjs:758-776` at `5a9c6794`).

## The Architectural Reality

- The family guard (#1510, landed through #1786) covers only ids under `FAMILY_RECORD_PREFIX` and types under `FAMILY_GRAPH_TYPE_PREFIX`. The relation therefore needs its own protection.
- There is a precedent for writing an identity and its binding in one transaction and reading both in one snapshot: `ai/graph/identityPublicationProtection.mjs:105-134` and `ai/services/memory-core/helpers/identityPublicationAdmission.mjs:432-476` (at `b6905782`).
- The founding compiler keeps every roster entry that shares a subject (`ai/graph/identityFoundingSeed.mjs:56-63`).
- Placement, per `npm run ai:structure-map -- --files --loc` at `08fba722`: `ai/mcp/server/memory-core/`, `ai/mcp/server/shared/services/`, `ai/graph/` and `ai/services/memory-core/`, plus the existing `ai/scripts/maintenance/` for the administrative path. No new folder.

## The Fix

1. **The record.** A protected SQLite graph record maps one provider subject to one `AgentIdentity`, keyed by:
<!-- binding-inventory: d825-subject-key -->
- `authProvider`
- `providerBaseUrl`
- `providerUserId`
<!-- /binding-inventory -->
   - It has its own scoped protection, or lands an independent protection prerequisite first.
   - Negative controls cover ordinary edit and delete, endpoint cascade, cold storage, and restore or migration.
   - It never waits on the family delivery as a whole.
2. **Two writers.**
   - Memory Core's request boundary writes it once, at a subject's first admission, in the same transaction as the identity.
   - An explicit administrative bind writes it through one named protected writer, which records the retained founding source revision.
   - No tool payload or ordinary node property writes it.
   - The administrative bind needs explicit administrative or cutover authority over the target identity. A fresh `operatesSeat` (`ai/services/fleet/FleetRegistryService.mjs:1048` at `e2791078`) proves only that the seat exists and that its operator store names the principal, so a Fleet caller needs it but it is never enough: ADR 0038 keeps the Fleet administer family and the Memory Core content family "separately receipted and never aggregated".
   - Three cases. A matching relation is an idempotent read, not a new bind. A new identity binds only through first admission. Binding or re-binding an existing or unresolved legacy identity needs the target authority; a singleton reviewed founding mapping (item 4) is its own authority.
   - The administrative bind is a plane-local administrative path, shaped like `ai/scripts/fleet/forgeConnections.mjs`, ADR 0038's only writer of forge connections. Host access to the Memory Core data root is its authority, it runs on the plane host only, and every mutation is a dry run unless `--apply`. No MCP tool, HTTP route or Fleet wire reaches the writer. The Fleet observes subjects (#1297) and reads the relation, but never binds.
3. **HTTP PAT binding.** `buildRequestContext` selects the identity by subject.
   - A login names a new identity only when the subject has none.
   - A login whose identity belongs to another subject is refused by name. Following OQ3, no identity is allocated and nothing is overwritten.
   - A missing or ambiguous subject is nonqualifying, with no fallback to the login or to `currentSeatId`.
4. **Seeding.** Only a singleton founding mapping seeds a binding. A subject that the founding record maps to more than one identity binds nothing, and the refusal names the unresolved mapping.
5. **Guards.**
   - `auth.allowedUsers` accepts subject-pinned entries (login plus `providerUserId`) as the admission anchor for servers that bind no graph identity.
   - Bare-login entries stay, documented as the weaker legacy profile.
   - The per-login first-use pin keys on the subject tuple. It stays only as defense in depth beside the bootstrap process pin; its removal stays with #83.
   - The GitLab verifier gets the same pin, and a disk-cache entry without `user.id` fails closed.
6. **Exact-id-first addressing**, shipped dormant behind the epic's activation gate:
<!-- binding-inventory: d825-former-login-readers -->
- `validateMailboxTarget` — the exact canonical id is checked before any alias
<!-- /binding-inventory -->

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Acting identity of an HTTP PAT request (`buildRequestContext`) | ADR 0038 §2.2 fact 5 | The verified subject selects the identity | Missing or ambiguous subject: nonqualifying, refused by name | ADR + `Server.mjs` JSDoc | AC-1 to AC-4 |
| Subject → identity record | The named protected writer | Written only at first admission or by the administrative bind | Any other write refuses | ADR + record JSDoc | AC-5, AC-7 |
| Administrative bind caller | ADR 0038 §2.2 fact 2's plane-local administrative path; §2.3 keeps the Fleet and content grant families separate | Only the plane-local administrative path (`ai/scripts/maintenance/`, host access to the Memory Core data root) binds an existing identity; no MCP tool, HTTP route or Fleet wire reaches the writer, and seat operation never authorizes a bind | Refused by name; nothing is written; a dry run unless `--apply` | CLI and writer JSDoc | AC-11 |
| `auth.allowedUsers` entry | ADR 0038 §2.2 | `{login, providerUserId}` pins the admission anchor | A bare login stays the documented weaker profile | `AuthService.mjs` JSDoc | AC-8 |
| `pinFirstProviderSubject`, GitLab verifier, outage disk cache | D825 OQ4 | The pin keys on the subject tuple, GitLab is pinned, and a cache entry without `user.id` fails closed | The process pin is unchanged | `AuthService.mjs` JSDoc | AC-9 |

## Acceptance Criteria

- [ ] AC-1: A rename keeps the identity. After S renames, its requests still act as I.
- [ ] AC-2: A re-registered login gets nothing (control 1).
  - Sequence: S owns I at login L and renames; T registers L while unseen; T authenticates first.
  - Each step decides from locally held evidence, and T receives no rows or grants of I.
- [ ] AC-3: A foreign subject is refused by name, and nothing is overwritten.
- [ ] AC-4: A missing or ambiguous subject is nonqualifying, including one that the founding record maps to more than one identity. The refusal names the mapping.
- [ ] AC-5: No ordinary property or tool payload writes the record. Ordinary edit, delete, endpoint cascade, cold storage, and restore or migration each refuse or preserve it.
- [ ] AC-6: Control 3. A collision refuses the conflicting claim, never a valid address or binding, and makes no first-match choice:
  - S keeps I_S = `@L` after changing its forge login.
  - An already-bound T that acquires L still acts as I_T.
  - `@L` and S's `@me` resolve I_S, with no alias rewriting.
  - An administrative claim `@L → I_T` refuses, while I_S's address and T's binding stay valid.
- [ ] AC-7: An endpoint re-bind (`providerBaseUrl` changes) through the administrative bind keeps the subject's family history reachable.
- [ ] AC-8: A bare-login allowlist entry is shown to be weaker than a subject-pinned one on a fresh process, on a cache miss, and at a reclaimed login's first presentation.
- [ ] AC-9: The pin keys on the subject tuple, the GitLab verifier is pinned, and a disk-cache entry without `user.id` fails closed.
- [ ] AC-10: Exact-id-first addressing ships dormant. Before the epic's gate, no changed addressing or trust path is active; control 2 activates at the gate.
- [ ] AC-11: Negative control: a principal for whom `operatesSeat` is true cannot bind another subject's legacy identity through any surface it can reach. First admission refuses by name (AC-3), and no MCP tool, HTTP route or Fleet wire reaches the administrative writer. The administrative path changes nothing without `--apply`, and re-presenting a matching relation returns it without a write.

## Out of Scope

- Stdio and OIDC transports, `NEO_AGENT_IDENTITY` and `proxy-header`. These belong to the transports leaf.
- The cutover inventory and bind, redirect retirement, the read-widening cutover and the trust readers. These belong to the cutover leaf.
- A fresh identity for a colliding newcomer (row F). It is deferred until an operator asks to admit one.

## Avoided Traps

- Retrofitting the relation from ordinary identity properties such as `githubLogin` or last-seen provider fields. Only the bind writes.
- Relying on the family guard, which protects only the family vocabulary.
- Re-keying `AgentIdentity` by subject (row C), which would migrate every id-keyed row.

Related: #1513 waits on this leaf. Sibling streams: #1296, #1297, #875, #83.

Bind authority (The Fix item 2, AC-11) folded 2026-11-16 from `@neo-gpt-sophie`'s peer-role read of #1297's interface to this leaf.

Live latest-open + closed sweep:
- Read the latest 20 open Brain issues (156 open, red band) at 2026-11-16T17:06Z.
- Ran open and closed searches for `provider subject identity`, `acting identity login`, `former login alias` and `providerUserId binding`. Read closed #1174 and #1190, which introduced the former-login resolver.
- No equivalent open owner and no prior exclusion found.

A2A in-flight claim sweep: the latest 30 messages in all read-states, rechecked at 17:08Z, carry no competing claim.

MC sweep: one query, "request binds identity by login; released login re-registered", returned 6 results, all D825's own trail.

Own-assignment sweep: 6 open issues, none overlapping.

Origin Session ID: 00199bfa-0a61-420b-a86e-496e506fe51e

Retrieval Hint: "D825 protected subject relation acting identity first admission"

## Timeline

- 2026-11-16T17:09:05Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-16T17:09:06Z @neo-opus-ada added the `enhancement` label
- 2026-11-16T17:09:06Z @neo-opus-ada added the `ai` label
- 2026-11-16T17:09:07Z @neo-opus-ada added the `architecture` label
- 2026-11-16T17:09:07Z @neo-opus-ada added the `security` label
- 2026-11-16T17:09:07Z @neo-opus-ada added the `agent-os` label
- 2026-11-16T17:09:24Z @neo-opus-ada cross-referenced by #1813
- 2026-11-16T17:09:43Z @neo-opus-ada cross-referenced by #1814
- 2026-11-16T17:10:07Z @neo-opus-ada cross-referenced by #1010
- 2026-11-16T17:11:01Z @neo-opus-ada cross-referenced by #1815
- 2026-11-16T17:11:13Z @neo-opus-ada added parent issue #1815
- 2026-11-16T17:11:19Z @neo-opus-ada marked this issue as blocking #1513
- 2026-11-16T17:11:22Z @neo-opus-ada marked this issue as blocking #1814
- 2026-11-16T17:11:29Z @neo-opus-ada marked this issue as blocking #1813
- 2026-11-16T18:00:47Z @neo-gpt-sophie cross-referenced by PR #1816
- 2026-11-16T18:42:29Z @neo-opus-ada cross-referenced by PR #1011
- 2026-11-16T19:02:55Z @neo-gpt-sophie cross-referenced by #1296

