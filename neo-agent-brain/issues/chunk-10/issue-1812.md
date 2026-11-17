---
id: 1812
title: A protected relation binds each provider subject to its acting identity
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-16T17:09:04Z'
updatedAt: '2026-11-17T20:27:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1812'
author: neo-opus-ada
commentsCount: 2
parentIssue: 1815
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1010 ADR 0038 §2.2 names the identity a request acts as'
blocking:
  - '[ ] 1813 Every provider-authenticated transport carries the subject'
  - '[ ] 1814 Cutover binds existing identities and retires former-login redirects'
  - '[ ] 1513 Use admitted families for routing and publication checks'
closedAt: '2026-11-17T20:27:54Z'
---
# A protected relation binds each provider subject to its acting identity

**Graduated from** neomjs/neo-agent-institution#825 at the body edited 2026-11-16T16:33:07Z. Quorum: Claude author signal, plus GPT and Gemini `[GRADUATION_APPROVED]`. The parent is the epic natively linked to this issue.

**Decision Record:** Required. ADR 0038 §2.2 gains fact 5 through its Institution leaf. **Decision Record impact:** depends-on that amendment.

Design authority: ADR 0038 §2.2 keeps four identity facts apart and calls the forge login "a DISPLAY/projection fact only". D825 adds fact 5, the identity a request acts as.

**Split 2026-11-17:** The Fix item 5 (the PAT admission guards), their two Contract Ledger rows, and AC-8 and AC-9 moved to #1829. The guards are their own surface in the shared `AuthService`, and #1513 waits only on this leaf's relation.

## Context

Today a request's acting identity is chosen by its login, and logins are reusable. GitHub releases a renamed login for anyone to register, and that is legitimate (operator, 2026-11-16: "anyone can register free accounts; nothing wrong with that"). Under provider-PAT auth, a plane that admits a released login binds whoever registers it to the identity that login still names. This is a source-derived exposure class, not an observed incident.

## The Problem

- `buildRequestContext` binds `@<login>` by lookup or provisioning (`ai/mcp/server/memory-core/Server.mjs:495-507` at Brain `d0481ce1`).
  - A seeded node is never compared with the request's `providerUserId`.
  - An auto-provisioned node takes each sign-in's provider fields without comparing them (`:637`).
- The PAT admission guards key on the login too. #1829 owns them.
- `validateMailboxTarget` resolves a former-login alias before it checks the exact graph id (`MailboxService.mjs:758-776` at `5a9c6794`).

## The Architectural Reality

- The family guard (#1510, landed through #1786) covers only ids under `FAMILY_RECORD_PREFIX` and types under `FAMILY_GRAPH_TYPE_PREFIX`. The relation therefore needs its own protection.
- There is a precedent for writing an identity and its binding in one transaction and reading both in one snapshot: `ai/graph/identityPublicationProtection.mjs:105-134` and `ai/services/memory-core/helpers/identityPublicationAdmission.mjs:432-476` (at `b6905782`).
- The founding compiler keeps every roster entry that shares a subject (`ai/graph/identityFoundingSeed.mjs:56-63`).
- Placement, per `npm run ai:structure-map -- --files --loc` at `08fba722`: `ai/mcp/server/memory-core/`, `ai/mcp/server/shared/services/`, `ai/graph/` and `ai/services/memory-core/`, plus the existing `ai/scripts/maintenance/` for the administrative path. No new folder.

## The Fix

1. **The record.** A protected relation maps one provider subject to one `AgentIdentity`, keyed by:
<!-- binding-inventory: d825-subject-key -->
- `authProvider`
- `providerBaseUrl`
- `providerUserId`
<!-- /binding-inventory -->
   - It lives in its own table on the graph's SQLite connection (`SubjectBinding`, beside the one-row `SubjectBindingActivation`), outside Nodes/Edges, so no graph write, edge cascade, decay or cold-storage pass reaches it. The identity-publication fence table is the precedent.
   - Negative controls cover ordinary edit and delete, endpoint cascade, cold storage, and restore or migration.
   - A migration that renames identities carries each binding with its identity.
   - A graph backup carries it as its own record types (`subject-binding` rows, then one `subject-binding-activation`), which only its exporter produces. A restore never overwrites a binding: an identical row is skipped and a conflicting one refused by name. `replace` swaps the binding rows only when the bundle carries one, and the activation only moves forward. Empty-target recovery admits, stages, validates and promotes the relation with the graph, and refuses a live graph that already holds one.
   - It never waits on the family delivery as a whole.
2. **Two writers.**
   - Memory Core's request boundary writes it once, at a subject's first admission, in the same transaction as the identity.
   - An explicit administrative bind writes it through one named protected writer, which records the retained founding source's revision (`provenance.sourceRevision`), qualified the way identity admission qualifies it: the plane is bound, and the source carries that lineage's issuer and head.
   - No tool payload or ordinary node property writes it.
   - The administrative bind needs explicit administrative or cutover authority over the target identity. A fresh `operatesSeat` (`ai/services/fleet/FleetRegistryService.mjs:1048` at `e2791078`) proves only that the seat exists and that its operator store names the principal. It never authorizes a bind, and no Fleet wire reaches the writer: ADR 0038 keeps the Fleet administer family and the Memory Core content family "separately receipted and never aggregated".
   - Three cases. A matching relation is an idempotent read, not a new bind. A new identity binds only through first admission. Binding or re-binding an existing or unresolved legacy identity needs the target authority; a singleton reviewed founding mapping (item 4) is its own authority.
   - The administrative bind is the plane-local `ai/scripts/maintenance/subjectBindings.mjs` (`list`, `bind`, `seed`), shaped like `ai/scripts/fleet/forgeConnections.mjs`, ADR 0038's only writer of forge connections. Host access to the Memory Core data root is its authority and it runs on the plane host only. Every mutation is a dry run on a read-only connection unless `--apply`, and an applied write must target the configured live graph. `bind` needs an identity the graph holds. No MCP tool, HTTP route or Fleet wire reaches the writer. The Fleet observes subjects (#1297) and reads the relation, but never binds.
3. **HTTP PAT binding.** `buildRequestContext` selects the identity by subject.
   - A login names a new identity only when the subject has none.
   - A login whose identity belongs to another subject is refused by name. Following OQ3, no identity is allocated and nothing is overwritten.
   - A missing or ambiguous subject is nonqualifying, with no fallback to the login or to `currentSeatId`.
   - Once active, an unbound subject that the founding record maps, to one identity or several, is nonqualifying too. A first admission never claims it for a new identity, which would lock the founding identities out, and the refusal names the mapping.
4. **Seeding.** Only a singleton founding mapping seeds a binding. `subjectBindings.mjs seed` binds each subject the qualified founding record maps to exactly one identity the graph holds, and only a free pair: an identity an administrative move placed elsewhere is held, and a subject bound elsewhere refuses by name. A subject that the founding record maps to more than one identity, or to one the graph lacks, binds nothing and is named as unresolved.
5. **Guards.** Moved to #1829.
6. **Exact-id-first addressing**, shipped dormant behind the epic's activation gate:
<!-- binding-inventory: d825-former-login-readers -->
- `validateMailboxTarget` — the exact canonical id is checked before any alias
<!-- /binding-inventory -->

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Acting identity of an HTTP PAT request (`buildRequestContext`) | ADR 0038 §2.2 fact 5 | The verified subject selects the identity | Missing or ambiguous subject, or an unbound subject the founding record maps: nonqualifying, refused by name | ADR + `Server.mjs` JSDoc | AC-1 to AC-4 |
| Subject → identity record | The named protected writer | Written only at first admission or by the administrative path (`bind`, `seed`); a migration and a graph restore carry existing bindings and create none | Any other write refuses | ADR + record JSDoc | AC-5, AC-7 |
| Graph backup bundle (`graph-backup` JSONL) and every restore of it | The record's AC-5 preservation | Carries `subject-binding` rows and one `subject-binding-activation` record after the edges; each restore path admits and writes them through the relation's restore writer | A malformed record is refused by line (admission, receipts validator) or counted failed (import), never stored | `subjectBinding.mjs`, importer and admission JSDoc | AC-5 |
| Administrative bind caller | ADR 0038 §2.2 fact 2's plane-local administrative path; §2.3 keeps the Fleet and content grant families separate | Only the plane-local `ai/scripts/maintenance/subjectBindings.mjs` (host access to the Memory Core data root) binds an existing identity; no MCP tool, HTTP route or Fleet wire reaches the writer, and seat operation never authorizes a bind | Refused by name; nothing is written; a dry run unless `--apply` | Script and writer JSDoc | AC-4, AC-7, AC-11 |

## Acceptance Criteria

- [ ] AC-1: A rename keeps the identity. After S renames, its requests still act as I.
- [ ] AC-2: A re-registered login gets nothing (control 1).
  - Sequence: S owns I at login L and renames; T registers L while unseen; T authenticates first.
  - Each step decides from locally held evidence, and T receives no rows or grants of I.
- [ ] AC-3: A foreign subject is refused by name, and nothing is overwritten.
- [ ] AC-4: A missing or ambiguous subject is nonqualifying, including one that the founding record maps to more than one identity. The refusal names the mapping. Once active, a first admission never claims a subject the founding record maps.
- [ ] AC-5: No ordinary property or tool payload writes the record. Ordinary edit, delete, endpoint cascade, cold storage, and restore or migration each refuse or preserve it.
- [ ] AC-6: Control 3. A collision refuses the conflicting claim, never a valid address or binding, and makes no first-match choice:
  - S keeps I_S = `@L` after changing its forge login.
  - An already-bound T that acquires L still acts as I_T.
  - `@L` and S's `@me` resolve I_S, with no alias rewriting.
  - An administrative claim `@L → I_T` refuses, while I_S's address and T's binding stay valid.
- [ ] AC-7: An endpoint re-bind (`providerBaseUrl` changes) through the administrative bind keeps the subject's family history reachable.
- ~~AC-8: A bare-login allowlist entry is shown to be weaker than a subject-pinned one on a fresh process, on a cache miss, and at a reclaimed login's first presentation.~~ Moved to #1829 as its AC-1.
- ~~AC-9: The pin keys on the subject tuple, the GitLab verifier is pinned, and a disk-cache entry without `user.id` fails closed.~~ Moved to #1829 as its AC-2.
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
- Carrying the relation in a bundle as `node` records with a reserved label: an ordinary graph write could forge one, and a restore would turn it into a binding.

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
- 2026-11-16T18:42:50Z @neo-opus-ada marked this issue as being blocked by #1010
- 2026-11-16T19:02:55Z @neo-gpt-sophie cross-referenced by #1296
- 2026-11-16T19:07:19Z @neo-opus-ada referenced in commit `e220e18` - "feat(memory-core): the subject selects the acting identity (#1812)

Before the cutover's activation record exists, login selection is unchanged and a first admission binds its subject only to an identity it creates, through GraphService.createBoundIdentity: the identity upsert and the binding share one physical transaction, and a rollback evicts the cached node and restores the GraphLog cursor. After activation, a bound subject keeps its identity whatever its login spells; a login naming an existing identity, bound or legacy, refuses by name; a request without a complete subject is nonqualifying; nothing falls back to the login."
- 2026-11-16T19:10:21Z @neo-opus-ada referenced in commit `3a81328` - "feat(memory-core): an exact id addresses itself before an alias (#1812)

Once the cutover has recorded its activation, validateMailboxTarget checks the exact canonical id before any former-login alias, so a newcomer who reclaims a former spelling keeps its own mail. Before activation the alias still resolves first; a connection that cannot read the activation record stays alias-first."
- 2026-11-16T19:14:45Z @neo-opus-ada referenced in commit `b01d881` - "feat(memory-core): identity migrations carry subject bindings (#1812)

renameBoundIdentities moves each binding to the id a migration gives its identity, in two passes so a swap never meets the uniqueness constraint, and refuses a rename that would leave two subjects on one identity, which rolls the migration's transaction back. The identity rename and canonicalize migrations call it inside their preservation transaction and report subjectBindingsMoved."
- 2026-11-16T19:15:39Z @neo-opus-ada referenced in commit `935d80f` - "test(memory-core): canonicalize carries an alias's binding (#1812)"
- 2026-11-16T21:56:17Z @neo-opus-ada referenced in commit `4dac0dd` - "feat(memory-core): a graph bundle carries the subject binding (#1812)

The relation lives outside Nodes/Edges, so the graph export dropped it and a restore into a fresh plane lost every binding and the activation record. The export now appends the relation as its own record types (subject-binding, subject-binding-activation), counted in the export's expected/exported verdict; only this exporter produces them, so no ordinary graph row can travel as one.

The importer hands them to the relation's restore writer: an identical binding is skipped, a conflicting one fails by name and never overwrites, replace swaps the binding rows only when the bundle carries one (an older bundle keeps the target's), and the activation record only moves forward."
- 2026-11-16T21:56:17Z @neo-opus-ada referenced in commit `c6c212e` - "feat(memory-core): empty-target restore carries the binding (#1812)

Graph bundles now carry the subject-binding relation's records, and two readers refused anything but node and edge records: target-set admission and the receipts validator. Both accept well-formed relation records and refuse malformed ones by line.

Admission counts and fingerprints the relation only when a bundle carries it, so a bundle without it is admitted byte-identically (pinned) and no in-flight v1 descriptor goes stale. Staging rejects a relation record the restore writer fails; validation compares the staged and live relation; promotion moves it through the restore writer inside its transaction and refuses a live graph that already holds a relation, which is no empty target."
- 2026-11-16T22:13:46Z @neo-opus-ada referenced in commit `ae75b26` - "feat(memory-core): a plane-local admin path binds an identity (#1812)

ai/scripts/maintenance/subjectBindings.mjs lists the relation and binds a provider subject to an identity the graph already holds; a new identity binds only at its first admission. A bind is a dry run on a read-only connection unless --apply, and an applied bind must target the configured live graph. It records the retained founding source revision, qualified the way identity admission qualifies it (bound lineage, the source's issuer and lineage head), so no caller types the revision. An endpoint move keeps the identity; any other conflict refuses by name.

planAdministrativeBind is the one decision the dry run and the write share. A census spec pins that only this script and the relation's own module name the administrative writer, so no MCP tool, HTTP route or Fleet wire reaches it. The entrypoint reads the boot descriptor, plane target and graph path from AiConfig and passes them to the operation as values (ADR-0019 §5.4 iii)."
- 2026-11-17T00:20:24Z @neo-opus-ada referenced in commit `6280ac9` - "feat(memory-core): seed bindings from the founding record (#1812)

subjectBindings.mjs seed binds what the retained founding record authorizes on its own: each subject it maps to exactly one identity the graph holds, under the qualified founding revision. It names, and binds nothing for, a subject mapped to several identities or to one the graph lacks, so nothing is a first-match choice. It binds only a free pair: an identity an administrative move placed elsewhere is held, and a subject bound elsewhere refuses by name. Dry run on a read-only connection unless --apply.

foundingSubjectMappings groups a seed's entries by subject in the seed's order; the founding subject tuple is the relation's subject key."
### @neo-opus-ada - 2026-11-17T02:39:14Z

**Handover (session sunset, 2026-11-17 ~02:45 CET): Ada's next session resumes this after today's Engine 14 lane.**

- **State:** `ada/1812-subject-binding` @ `6280ac93`, pushed, no PR yet. Its base is dev `e2791078`; rebase onto current dev before the PR.
- **Done:** the relation store; request-boundary selection (dormant until activation); exact-id-first addressing; migration carry; the bundle carry through every restore path (AC-5); and the plane-local `ai/scripts/maintenance/subjectBindings.mjs` with `list`, `bind` and `seed` (AC-7, AC-11, Fix 4's seeding). The body was synced at 00:23Z.
- **Tests:** unit suites green, except the pre-existing `QueryReRanker` #9959 failure, which is red on base too and filed as a defect note.
- **Next, in order:**
  1. **Fix 3's founding guard (AC-4).** In `Server.selectAgentIdentityBySubject`, after `if (bound) return …`:
     - read `aiConfig.identityPublicationBoot` and the plane target at the use site; a null boot means no founding record (standalone), so proceed;
     - otherwise do the qualified founding read (move `qualifiedFounding` out of the CLI into a shared helper) and check `foundingSubjectMappings`;
     - a mapped unbound subject is refused, naming its identities; a failed read fails closed.
     - Tests: helper level with the lineage fixture from `subjectBindings.spec.mjs`. The server arm needs ADR-0019 §5.4 rung (iv), or a seam.
  2. Decide whether AC-8/9 (the AuthService guards) split into their own leaf.
  3. Rebase, open the PR (`Resolves #1812`), and request Sophie (non-Claude) at green.
- **Owner:** `@neo-opus-ada`.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-17T15:13:14Z

**Handover addendum (2026-11-17, 15:13Z): unblocked, and the 02:39Z handover above stands.**

- **Unblocked at 15:10Z:** neomjs/neo-agent-institution#1011 merged (ADR 0038 §2.2 fact 5), closing the only blocker, neomjs/neo-agent-institution#1010.
- **Branch:** `ada/1812-subject-binding` @ `6280ac93` is now 9 commits behind and 9 ahead of dev `92f0e736`. Rebase before the PR.
- **Next, in the order above:** the founding guard (AC-4), then the AC-8/9 split decision, then the PR. AC-4 reads `aiConfig.identityPublicationBoot`, so ADR 0019 comes first.
- **Reviewer at green:** Sophie is dark. Request @neo-gpt-euclid or @neo-gemini-julia instead (non-Claude, check `who_is_online` first).
- **Owner:** `@neo-opus-ada`, next session.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-17T15:35:29Z @neo-opus-ada cross-referenced by #1829
- 2026-11-17T15:37:41Z @neo-opus-ada referenced in commit `3a32d7c` - "feat(memory-core): a protected subject-to-identity relation (#1812)

A SubjectBinding table outside Nodes/Edges maps the verified provider subject to the one AgentIdentity a request acts as. Two writers: first admission binds only the identity its transaction created; the administrative bind binds an existing identity with its founding revision and moves an identity to its subject's new endpoint. Conflicting claims refuse by name and write nothing; a matching claim reads back. Selection stays dormant until the cutover records its activation."
- 2026-11-17T15:37:41Z @neo-opus-ada referenced in commit `3dfad39` - "feat(memory-core): the subject selects the acting identity (#1812)

Before the cutover's activation record exists, login selection is unchanged and a first admission binds its subject only to an identity it creates, through GraphService.createBoundIdentity: the identity upsert and the binding share one physical transaction, and a rollback evicts the cached node and restores the GraphLog cursor. After activation, a bound subject keeps its identity whatever its login spells; a login naming an existing identity, bound or legacy, refuses by name; a request without a complete subject is nonqualifying; nothing falls back to the login."
- 2026-11-17T15:37:41Z @neo-opus-ada referenced in commit `0d313e8` - "feat(memory-core): an exact id addresses itself before an alias (#1812)

Once the cutover has recorded its activation, validateMailboxTarget checks the exact canonical id before any former-login alias, so a newcomer who reclaims a former spelling keeps its own mail. Before activation the alias still resolves first; a connection that cannot read the activation record stays alias-first."
- 2026-11-17T15:37:42Z @neo-opus-ada referenced in commit `34d20ea` - "feat(memory-core): identity migrations carry subject bindings (#1812)

renameBoundIdentities moves each binding to the id a migration gives its identity, in two passes so a swap never meets the uniqueness constraint, and refuses a rename that would leave two subjects on one identity, which rolls the migration's transaction back. The identity rename and canonicalize migrations call it inside their preservation transaction and report subjectBindingsMoved."
- 2026-11-17T15:37:42Z @neo-opus-ada referenced in commit `78695b0` - "test(memory-core): canonicalize carries an alias's binding (#1812)"
- 2026-11-17T15:37:43Z @neo-opus-ada referenced in commit `f78440b` - "feat(memory-core): a graph bundle carries the subject binding (#1812)

The relation lives outside Nodes/Edges, so the graph export dropped it and a restore into a fresh plane lost every binding and the activation record. The export now appends the relation as its own record types (subject-binding, subject-binding-activation), counted in the export's expected/exported verdict; only this exporter produces them, so no ordinary graph row can travel as one.

The importer hands them to the relation's restore writer: an identical binding is skipped, a conflicting one fails by name and never overwrites, replace swaps the binding rows only when the bundle carries one (an older bundle keeps the target's), and the activation record only moves forward."
- 2026-11-17T15:37:43Z @neo-opus-ada referenced in commit `25c0eaf` - "feat(memory-core): empty-target restore carries the binding (#1812)

Graph bundles now carry the subject-binding relation's records, and two readers refused anything but node and edge records: target-set admission and the receipts validator. Both accept well-formed relation records and refuse malformed ones by line.

Admission counts and fingerprints the relation only when a bundle carries it, so a bundle without it is admitted byte-identically (pinned) and no in-flight v1 descriptor goes stale. Staging rejects a relation record the restore writer fails; validation compares the staged and live relation; promotion moves it through the restore writer inside its transaction and refuses a live graph that already holds a relation, which is no empty target."
- 2026-11-17T15:37:43Z @neo-opus-ada referenced in commit `0fb7b90` - "feat(memory-core): a plane-local admin path binds an identity (#1812)

ai/scripts/maintenance/subjectBindings.mjs lists the relation and binds a provider subject to an identity the graph already holds; a new identity binds only at its first admission. A bind is a dry run on a read-only connection unless --apply, and an applied bind must target the configured live graph. It records the retained founding source revision, qualified the way identity admission qualifies it (bound lineage, the source's issuer and lineage head), so no caller types the revision. An endpoint move keeps the identity; any other conflict refuses by name.

planAdministrativeBind is the one decision the dry run and the write share. A census spec pins that only this script and the relation's own module name the administrative writer, so no MCP tool, HTTP route or Fleet wire reaches it. The entrypoint reads the boot descriptor, plane target and graph path from AiConfig and passes them to the operation as values (ADR-0019 §5.4 iii)."
- 2026-11-17T15:37:44Z @neo-opus-ada referenced in commit `804706e` - "feat(memory-core): seed bindings from the founding record (#1812)

subjectBindings.mjs seed binds what the retained founding record authorizes on its own: each subject it maps to exactly one identity the graph holds, under the qualified founding revision. It names, and binds nothing for, a subject mapped to several identities or to one the graph lacks, so nothing is a first-match choice. It binds only a free pair: an identity an administrative move placed elsewhere is held, and a subject bound elsewhere refuses by name. Dry run on a read-only connection unless --apply.

foundingSubjectMappings groups a seed's entries by subject in the seed's order; the founding subject tuple is the relation's subject key."
- 2026-11-17T15:37:44Z @neo-opus-ada referenced in commit `2d10f0b` - "feat(memory-core): a founding subject never binds at first admission (#1812)

Once selection by subject is active, an unbound subject the qualified founding record maps, to one identity or several, is refused by name (NEO_AGENT_IDENTITY_SUBJECT_FOUNDING_MAPPED) instead of creating a new identity that would lock its founding identities out. An unreadable record fails closed. The qualified founding read moves from the administrative CLI into subjectBinding.mjs, and the request path reaches it through IdentityPublicationService's boot generation."
- 2026-11-17T15:43:21Z @neo-opus-ada cross-referenced by PR #1830
- 2026-11-17T18:06:29Z @neo-gpt-euclid cross-referenced by PR #1831
- 2026-11-17T19:24:07Z @neo-opus-ada referenced in commit `84f4c1d` - "fix(memory-core): an unreadable activation refuses by name (#1812)

isSubjectSelectionActive reads active or known dormant and throws SUBJECT_BINDING_UNAVAILABLE for a missing, closed or unreadable connection. The request boundary maps that, and a graph that is not ready, to NEO_AGENT_IDENTITY_SUBJECT_UNAVAILABLE instead of falling back to the login, and mailbox addressing refuses instead of resolving a former-login alias."
- 2026-11-17T19:24:08Z @neo-opus-ada referenced in commit `2d6bff7` - "fix(memory-core): a first admission decides inside its transaction (#1812)

createBoundIdentity refuses a node that exists once the write lock is taken (SUBJECT_BINDING_IDENTITY_EXISTS) and runs the caller's admit check before any write; the subject selector passes the founding guard as that check and maps the refusal to a subject conflict. An unbound identity another writer created after the request's own read is kept, never bound or overwritten."
- 2026-11-17T19:24:08Z @neo-opus-ada referenced in commit `38d1ad7` - "fix(memory-core): a restore commits the relation as one unit (#1812)

restoreSubjectBindingRelation admits and writes a bundle's relation in one transaction after the graph rows: a malformed or conflicting record, a duplicate inside the bundle included, refuses every record and the target keeps its relation and activation. DatabaseService no longer reports such an import as successful, and the empty-target proof counts a live relation as occupancy before anything is staged or promoted."
- 2026-11-17T19:24:08Z @neo-opus-ada referenced in commit `d054e05` - "fix(memory-core): a bound identity keys every PAT request (#1812)

Once selection is active, every request that carries a provider subject
or may create an identity selects by subject, lookup-only planes
included: a bound subject acts as its identity, an unbound one is
refused by name (NEO_AGENT_IDENTITY_SUBJECT_UNBOUND) where its source
creates none. The bound identity's bare id is the request's isolation
key, so a renamed login keeps its identity's rows and never reads what
an earlier holder of the new login wrote."
- 2026-11-17T19:24:09Z @neo-opus-ada referenced in commit `41dd356` - "fix(memory-core): an endpoint move keeps the endpoint it leaves (#1812)

Family history is keyed by the full subject tuple, so an administrative
endpoint move records the endpoints the binding leaves in a
formerBaseUrls column. The history admitted under an earlier tuple
stays reachable from the binding through the family-history reader."
- 2026-11-17T19:27:41Z @neo-opus-ada referenced in commit `bce1eff` - "style(memory-core): align the blocks this change touched (#1812)"
- 2026-11-17T20:01:59Z @neo-opus-ada referenced in commit `0cc7a12` - "fix(memory-core): a stamped request without a login selects too (#1812)

The empty-login fallthrough ran before the subject decision, so once
selection was active a provider-stamped request that carried no login
returned an empty context instead of its bound identity or a named
refusal. It now returns empty only for a request with neither a login
nor a provider stamp, and before activation for one without a login."
- 2026-11-17T20:27:54Z @tobiu referenced in commit `23c077f` - "feat(memory-core): the subject selects the acting identity (#1812) (#1830)

* feat(memory-core): a protected subject-to-identity relation (#1812)

A SubjectBinding table outside Nodes/Edges maps the verified provider subject to the one AgentIdentity a request acts as. Two writers: first admission binds only the identity its transaction created; the administrative bind binds an existing identity with its founding revision and moves an identity to its subject's new endpoint. Conflicting claims refuse by name and write nothing; a matching claim reads back. Selection stays dormant until the cutover records its activation.

* feat(memory-core): the subject selects the acting identity (#1812)

Before the cutover's activation record exists, login selection is unchanged and a first admission binds its subject only to an identity it creates, through GraphService.createBoundIdentity: the identity upsert and the binding share one physical transaction, and a rollback evicts the cached node and restores the GraphLog cursor. After activation, a bound subject keeps its identity whatever its login spells; a login naming an existing identity, bound or legacy, refuses by name; a request without a complete subject is nonqualifying; nothing falls back to the login.

* feat(memory-core): an exact id addresses itself before an alias (#1812)

Once the cutover has recorded its activation, validateMailboxTarget checks the exact canonical id before any former-login alias, so a newcomer who reclaims a former spelling keeps its own mail. Before activation the alias still resolves first; a connection that cannot read the activation record stays alias-first.

* feat(memory-core): identity migrations carry subject bindings (#1812)

renameBoundIdentities moves each binding to the id a migration gives its identity, in two passes so a swap never meets the uniqueness constraint, and refuses a rename that would leave two subjects on one identity, which rolls the migration's transaction back. The identity rename and canonicalize migrations call it inside their preservation transaction and report subjectBindingsMoved.

* test(memory-core): canonicalize carries an alias's binding (#1812)

* feat(memory-core): a graph bundle carries the subject binding (#1812)

The relation lives outside Nodes/Edges, so the graph export dropped it and a restore into a fresh plane lost every binding and the activation record. The export now appends the relation as its own record types (subject-binding, subject-binding-activation), counted in the export's expected/exported verdict; only this exporter produces them, so no ordinary graph row can travel as one.

The importer hands them to the relation's restore writer: an identical binding is skipped, a conflicting one fails by name and never overwrites, replace swaps the binding rows only when the bundle carries one (an older bundle keeps the target's), and the activation record only moves forward.

* feat(memory-core): empty-target restore carries the binding (#1812)

Graph bundles now carry the subject-binding relation's records, and two readers refused anything but node and edge records: target-set admission and the receipts validator. Both accept well-formed relation records and refuse malformed ones by line.

Admission counts and fingerprints the relation only when a bundle carries it, so a bundle without it is admitted byte-identically (pinned) and no in-flight v1 descriptor goes stale. Staging rejects a relation record the restore writer fails; validation compares the staged and live relation; promotion moves it through the restore writer inside its transaction and refuses a live graph that already holds a relation, which is no empty target.

* feat(memory-core): a plane-local admin path binds an identity (#1812)

ai/scripts/maintenance/subjectBindings.mjs lists the relation and binds a provider subject to an identity the graph already holds; a new identity binds only at its first admission. A bind is a dry run on a read-only connection unless --apply, and an applied bind must target the configured live graph. It records the retained founding source revision, qualified the way identity admission qualifies it (bound lineage, the source's issuer and lineage head), so no caller types the revision. An endpoint move keeps the identity; any other conflict refuses by name.

planAdministrativeBind is the one decision the dry run and the write share. A census spec pins that only this script and the relation's own module name the administrative writer, so no MCP tool, HTTP route or Fleet wire reaches it. The entrypoint reads the boot descriptor, plane target and graph path from AiConfig and passes them to the operation as values (ADR-0019 §5.4 iii).

* feat(memory-core): seed bindings from the founding record (#1812)

subjectBindings.mjs seed binds what the retained founding record authorizes on its own: each subject it maps to exactly one identity the graph holds, under the qualified founding revision. It names, and binds nothing for, a subject mapped to several identities or to one the graph lacks, so nothing is a first-match choice. It binds only a free pair: an identity an administrative move placed elsewhere is held, and a subject bound elsewhere refuses by name. Dry run on a read-only connection unless --apply.

foundingSubjectMappings groups a seed's entries by subject in the seed's order; the founding subject tuple is the relation's subject key.

* feat(memory-core): a founding subject never binds at first admission (#1812)

Once selection by subject is active, an unbound subject the qualified founding record maps, to one identity or several, is refused by name (NEO_AGENT_IDENTITY_SUBJECT_FOUNDING_MAPPED) instead of creating a new identity that would lock its founding identities out. An unreadable record fails closed. The qualified founding read moves from the administrative CLI into subjectBinding.mjs, and the request path reaches it through IdentityPublicationService's boot generation.

* fix(memory-core): an unreadable activation refuses by name (#1812)

isSubjectSelectionActive reads active or known dormant and throws SUBJECT_BINDING_UNAVAILABLE for a missing, closed or unreadable connection. The request boundary maps that, and a graph that is not ready, to NEO_AGENT_IDENTITY_SUBJECT_UNAVAILABLE instead of falling back to the login, and mailbox addressing refuses instead of resolving a former-login alias.

* fix(memory-core): a first admission decides inside its transaction (#1812)

createBoundIdentity refuses a node that exists once the write lock is taken (SUBJECT_BINDING_IDENTITY_EXISTS) and runs the caller's admit check before any write; the subject selector passes the founding guard as that check and maps the refusal to a subject conflict. An unbound identity another writer created after the request's own read is kept, never bound or overwritten.

* fix(memory-core): a restore commits the relation as one unit (#1812)

restoreSubjectBindingRelation admits and writes a bundle's relation in one transaction after the graph rows: a malformed or conflicting record, a duplicate inside the bundle included, refuses every record and the target keeps its relation and activation. DatabaseService no longer reports such an import as successful, and the empty-target proof counts a live relation as occupancy before anything is staged or promoted.

* fix(memory-core): a bound identity keys every PAT request (#1812)

Once selection is active, every request that carries a provider subject
or may create an identity selects by subject, lookup-only planes
included: a bound subject acts as its identity, an unbound one is
refused by name (NEO_AGENT_IDENTITY_SUBJECT_UNBOUND) where its source
creates none. The bound identity's bare id is the request's isolation
key, so a renamed login keeps its identity's rows and never reads what
an earlier holder of the new login wrote.

* fix(memory-core): an endpoint move keeps the endpoint it leaves (#1812)

Family history is keyed by the full subject tuple, so an administrative
endpoint move records the endpoints the binding leaves in a
formerBaseUrls column. The history admitted under an earlier tuple
stays reachable from the binding through the family-history reader.

* style(memory-core): align the blocks this change touched (#1812)

* fix(memory-core): a stamped request without a login selects too (#1812)

The empty-login fallthrough ran before the subject decision, so once
selection was active a provider-stamped request that carried no login
returned an empty context instead of its bound identity or a named
refusal. It now returns empty only for a request with neither a login
nor a provider stamp, and before activation for one without a login."
- 2026-11-17T20:27:56Z @tobiu closed this issue
- 2026-11-17T20:58:57Z @neo-opus-ada cross-referenced by PR #1833

