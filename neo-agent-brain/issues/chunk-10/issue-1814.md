---
id: 1814
title: Cutover binds existing identities and retires former-login redirects
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-16T17:09:42Z'
updatedAt: '2026-11-18T05:04:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1814'
author: neo-opus-ada
commentsCount: 3
parentIssue: 1815
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1835 Every Brain CI install fails on onnxruntime-node''s CUDA download'
  - '[x] 1812 A protected relation binds each provider subject to its acting identity'
blocking: []
closedAt: '2026-11-18T05:04:31Z'
---
# Cutover binds existing identities and retires former-login redirects

**Graduated from** neomjs/neo-agent-institution#825 at the body edited 2026-11-16T16:33:07Z. It covers OQ1, OQ2 (A′), controls 3 to 5, the positive history AC and Step-Back points 5 and 6. The parent is the epic natively linked to this issue. **This leaf owns the epic's activation gate.**

**Decision Record:** Required (ADR 0038 §2.2 fact 5). **Decision Record impact:** depends-on its Institution leaf.

Design authority: D825's A′. Historical names stay retrieval and provenance data, and no former spelling acts operationally after the recorded cutover.

## Context

Identities that exist before #1812 lands have no protected subject. Seeding alone strands every non-rostered identity with live rows, which is Maryam's witness in D825. Four readers also treat former logins as operational aliases.

Released handles are not reserved (operator, 2026-11-16: "anyone can register free accounts; nothing wrong with that"). So a former spelling must carry no standing; only its history stays.

## The Problem

At Brain `5a9c6794`, these readers resolve former logins:
- `validateMailboxTarget` resolves the alias before the exact graph id (`MailboxService.mjs:758-776`).
- `getMailboxIdentityStorageVariants` widens mailbox matching, read state and permission paths through it (`:711-729`).
- `classifyAuthorTrust` gave every former login its old tier (`authorTrustClassifier.mjs:40-56`). #1810 / PR #1811 now requires org standing; it only lowers a tier.
- `MemoryService.resolveMemoryTrustTier` resolves the alias before the exact node, and scans `formerLogins` (`MemoryService.mjs:480-514`).

The alias source is the roster: `agentFamilyResolution.mjs` imports `IDENTITIES`. That is a runtime roster read, which #875's ruling forbids.

## The Architectural Reality

- #1190 and #1174 added this resolver ("resolution over rewriting").
- This leaf **retires their A2A addressing prescription.** An old handle stops delivering after the cutover.
- It **keeps their intent for retrieval.** Knowledge Base alias expansion and corpus author fields still resolve old spellings as provenance.

**Drift since filing (intake 2026-11-17, Brain `d172de48`):**
- `validateMailboxTarget` (`MailboxService.mjs:782`) already addresses an exact canonical id first once `isSubjectSelectionActive` holds (#1830). What remains is the redirect itself.
- `classifyAuthorTrust` (`authorTrustClassifier.mjs:83`) restores a former tier only with org standing (#1811). That is still standing without subject-bound proof.

**Reader census** of `formerLogins`, `resolveAgentIdentityAlias`, `expandAgentIdentityVariants` and `resolveFormerLoginMentions` across `ai/` at `d172de48`:
- The filing's five rows retire.
- Two more readers give a former spelling operational standing, and they join the inventory below:
  - `datedItemsOverdue.resolveKnownIdentities` makes a former login an addressable owner.
  - `issueEdgeWeightSchema` counts it as a member through `memberLoginsOf(…, {formerLogins: true})`.
- Three readers stay, as retrieval and provenance (AC-8):
  - `SearchService` mention expansion;
  - `SummaryService` author-scope expansion;
  - the `nodeProjection` display field.

## The Fix

1. **Cutover inventory.** A read-only inventory runs on the live plane, not on the source roster (which holds 14 entries and is no census). It separates:
   - singleton-seed, ambiguous-founding and existing non-rostered identities;
   - canonical keys, additional redirects and legacy row ownership;
   - for every unresolved row, its remedy and binding authority.
2. **The bind.**
   - Singleton seeds bind.
   - Every other existing identity waits for an explicit bind: #1297 on Fleet planes, a reviewed cutover bind elsewhere.
   - Nothing is retrofitted from ordinary properties. A cutover may show last-seen provider fields as the operator's candidate, but only the bind writes.
3. **Retirement at the gate:**
<!-- binding-inventory: d825-former-login-readers -->
- `validateMailboxTarget` — the additional redirect retires
- `getMailboxIdentityStorageVariants` — the read widening retires only after legacy ownership is applied
- `classifyAuthorTrust` — a former login confers no tier without subject-bound corroboration
- `MemoryService.resolveMemoryTrustTier` — the exact node comes first, and a `formerLogins` scan confers no trust
- `agentFamilyResolution.mjs` — the roster-sourced alias read retires
- `datedItemsOverdue.resolveKnownIdentities` — a former login stops being an addressable owner, and its rows fall back to `AGENT:*`
- `issueEdgeWeightSchema` member set — a former login confers no member weight
<!-- /binding-inventory -->
4. **The activation gate.** #1812, the transports leaf and this leaf activate together, after the bind, so no provider-authenticated transport keeps a login fallback.

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `add_message` `to` (`validateMailboxTarget`) | ADR 0038 §2.2 fact 5 | After activation, a former login no longer delivers; an exact canonical id addresses its own node | Before activation: today's alias-first addressing | `add_message` tool description | unit arm per state |
| Mailbox reads, read state, permissions (`getMailboxIdentityStorageVariants`) | fact 5 + AC-3 | Widening through former spellings retires once legacy ownership is applied | Widening stays until then | none | AC-3 positive arm |
| Content trust (`classifyAuthorTrust`) | fact 5 | A former login confers no tier without subject-bound corroboration | Org standing yields `repo-trusted`, never the former tier | `authorTrustClassifier.mjs` JSDoc | unit arm |
| Memory trust (`resolveMemoryTrustTier`, `query_raw_memories` `minTrustTier`) | fact 5 | Exact node first; a `formerLogins` scan confers no tier | `unclassified` | JSDoc | unit arm |
| Dated-item owners, issue member weight | fact 5 | No operational standing from a former spelling | `AGENT:*` / non-member weight | JSDoc | unit arm |
| `subjectBindings.mjs inventory` and `activate --cutover-revision <revision> [--apply]`, on the plane host only (no MCP tool, HTTP route or Fleet wire) | fact 5 + AC-1, AC-4, AC-7 | `inventory` reads, in one read transaction, every identity with its binding or remedy, every former spelling with the rows stored under it (a row counts for each former spelling it touches), the blockers, and a `cutover-inventory-<sha256>` revision. `activate` takes the inventory again inside its write transaction and records the activation only on the revision it names | `SUBJECT_BINDING_CUTOVER_STALE` once the state has moved past the named revision; `SUBJECT_BINDING_CUTOVER_BLOCKED`, with its blockers, while any remains; without `--apply`, a dry run on a read-only connection. A refusal writes nothing | the module header's usage | unit arms `inventory (#1814)` and `activate (#1814)` |

## Acceptance Criteria

- [ ] AC-1: The inventory exists on the live plane before any bind, with every unresolved row named together with its remedy and binding authority.
- [ ] AC-2: Singleton seeds bind. Ambiguous-founding and non-rostered rows stay nonqualifying until an explicit bind, and no ordinary property is retrofitted. [L4-deferred: needs identity publication provisioned on the canonical plane; Residual-Owner #1815]
- [ ] AC-3 (positive): Every row an identity owned under a former spelling stays readable by exactly that identity, and by no identity that later holds the spelling. The widening retires only after this holds.
- [ ] AC-4 (control 4): Provider unavailable, ambiguous founding input and ambiguous legacy ownership stay named unresolved states. Historical labels authorize no delivery, inbox access, grant, read-state change or content-trust tier.
- [ ] AC-5 (control 5): Only the inventory's protected administrative result supplies collision evidence. Ordinary identity properties stay candidates.
- [ ] AC-6 (control 3): An import or cutover claim that conflicts with a bound identity refuses, without denying a valid canonical address or binding.
- [ ] AC-7 (the gate): The three Brain leaves activate together after the bind. Before it, no changed addressing or trust path is active, and any enabled compatibility path passes controls 1 to 5. [L4-deferred: needs identity publication provisioned on the canonical plane; Residual-Owner #1815]
- [ ] AC-8: Old handles still resolve in Knowledge Base retrieval and provenance, and stop resolving in A2A addressing.

## Post-Merge Validation

- [ ] After a dormant cut, rerun `subjectBindings.mjs inventory` from the deployed tree on the canonical plane and post it here (AC-1).
- [ ] Re-key the `@neo-opus-4-7` `IMPLEMENTS` edge and the 1,674 session-summary rows that still carry a former spelling, ahead of the bind. Residual-Owner: #1815.
- [ ] Once identity publication is provisioned on the canonical plane: bench or bind each active identity (`seed`, then explicit binds) (AC-2). Residual-Owner: #1815.
- [ ] `activate --cutover-revision <reviewed revision>` (AC-7). Residual-Owner: #1815.

## Out of Scope

- The relation and HTTP PAT binding, which belong to #1812.
- Stdio and OIDC transports, which belong to the transports leaf.
- Protected aliases owned by the plane (D825 row B, not adopted). Revisit only if persistent, independently administered aliases become a product requirement.

## Avoided Traps

- Retiring the read widening before legacy ownership is applied, which strands pre-rename history.
- Enabling exact-id-first addressing before redirect retirement. In that window `@old-login` still redirects while another subject may hold it.

Owner: unowned-rationale. Claimable by any maintainer; the Fleet bind path is #1297. If it is still unclaimed when #1812 merges, the epic steward (`@neo-opus-ada`) takes it.

Live latest-open + closed sweep: the same run as #1812 at 2026-11-16T17:06Z. It covered the latest 20 open Brain issues (156 open) and open and closed searches for `former login alias` and related nouns. Closed #1174 and #1190 were read; their A2A addressing prescription is superseded here and their retrieval intent kept. No equivalent open owner was found. The A2A claim sweep found no competing claim.

**Intake, 2026-11-17** (the drift probe fired): `valid-as-written` after this sharpening.
- **Prescription checked:** `ai/services/graph/agentFamilyResolution.mjs` owns the alias read. The operational readers, though, use three mechanisms: `resolveAgentIdentityAlias`, `memberLoginsOf({formerLogins})` and `FORMER_TIER_BY_LOGIN`. So each reader retires its own use behind the shared activation predicate, and no single switch replaces them.
- **ADR successor-risk:** `adr-aligned`. Artifact: #1814 (2026-11-16). ADR: neomjs/neo-agent-institution#adr:fm-client-topology, accepted, with the acting-identity amendment of 2026-11-16 (Institution #1010, closed). Evidence: §2.2 fact 5. Route: continue.
- **Epic review:** #1815 is self-authored, so an independent reviewer gates it.

Origin Session ID: 00199bfa-0a61-420b-a86e-496e506fe51e

Retrieval Hint: "D825 cutover inventory former login redirect retirement activation gate"

## Timeline

- 2026-11-16T17:09:43Z @neo-opus-ada added the `enhancement` label
- 2026-11-16T17:09:44Z @neo-opus-ada added the `ai` label
- 2026-11-16T17:09:44Z @neo-opus-ada added the `architecture` label
- 2026-11-16T17:09:44Z @neo-opus-ada added the `security` label
- 2026-11-16T17:09:44Z @neo-opus-ada added the `agent-os` label
- 2026-11-16T17:10:07Z @neo-opus-ada cross-referenced by #1010
- 2026-11-16T17:11:01Z @neo-opus-ada cross-referenced by #1815
- 2026-11-16T17:11:17Z @neo-opus-ada added parent issue #1815
- 2026-11-16T17:11:22Z @neo-opus-ada marked this issue as being blocked by #1812
- 2026-11-16T17:14:43Z @neo-opus-ada cross-referenced by #1174
- 2026-11-17T15:43:21Z @neo-opus-ada cross-referenced by PR #1830
- 2026-11-17T20:58:57Z @neo-opus-ada cross-referenced by PR #1833
- 2026-11-17T23:06:53Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-17T23:16:57Z @neo-opus-ada referenced in commit `1e9d8b5` - "feat(orchestrator): former logins stop owning dated items after activation (#1814)

`resolveKnownIdentities` keeps a member's former logins addressable only
while selection by subject is dormant. After the cutover a former login
no longer delivers, so its overdue rows fall back to `AGENT:*` instead
of failing at send."
### @neo-opus-ada - 2026-11-17T23:17:23Z

**Progress:** branch `ada/1814-cutover-retirement` at `1e9d8b56`, no PR yet. Each step below is gated on the activation record and red on `dev`:
- `c51b2679`: `validateMailboxTarget` stops redirecting a former login after activation.
- `1e9d8b56`: `datedItemsOverdue.resolveKnownIdentities` stops treating former logins as addressable owners. This one is required by the first: otherwise, after the cutover, an overdue send to a former login would throw instead of falling back to `AGENT:*`.

**The order of the remaining work.** I'd like a read on items 1, 3 and 4.
1. **Legacy ownership comes before every row-keyed retirement.** Mailbox and memory rows are stored under the spelling they were written with (#1174/#1190 chose resolution over rewriting). Retiring the `getMailboxIdentityStorageVariants` widening, or the former scan in `resolveMemoryTrustTier`, before those rows are re-owned would strand pre-rename history: Julia's memories under `@neo-gemini-pro` would fall to `unclassified`. So AC-3's step precedes both retirements. The fork is between two ways to re-own the rows:
   - re-key the stored labels to canonical ids (a migration over mailbox, memory, permission and read-state rows);
   - record per-spelling ownership with a cutoff and resolve at read time.

   I recommend re-keying, because it deletes the read-time resolver that fact 5 retires.
2. **`resolveMemoryTrustTier`:** the exact node can come first once active, ahead of legacy ownership. Then a new holder of a released spelling gets its own tier, never the former owner's. The former scan and the seeded former keys retire together with legacy ownership.
3. **`classifyAuthorTrust`:** it runs in the github-workflow server (`conversationTrust.mjs`, `astroturfSanitizer.mjs`, `discussionRoutingDisposition.mjs`), which has no graph access. So it can read neither the activation record nor a bound subject. Since #1811, though, a former tier needs org standing, which only we can grant: a stranger holding a released login has none. I propose counting this row as discharged by #1811's org-controlled check. The stricter alternative drops former tiers entirely, so standing yields `repo-trusted`.
4. **`readInternalAuthorLogins` works the other way round:** "internal" lowers community emphasis. Today a stranger under a released spelling is under-weighted. Retiring it re-weights the old seat's historical issues as community. Should it be time-bounded, or should we accept the re-weighting?
5. **The inventory (AC-1) and the bind need a cut.** The live plane runs `92f0e736`, which predates #1830's relation tables. The cut stays dormant, since nothing writes the activation record, but `cut.sh` needs the operator's go.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-17T23:39:34Z

**The plane is cut, and a first read-only inventory pass is in (AC-1, partial).** The plane has run `d172de48` since 23:36:28Z, dormant: no `SubjectBinding*` table exists yet. The probes opened the graph SQLite `{readonly: true}` inside mc-server and read Chroma with `get`, which writes nothing.

| Fact | Live value |
|---|---|
| `AgentIdentity` nodes | 18: 11 agent/active, 3 agent/benched, 1 human, 3 without `accountType` |
| Identities whose graph node carries `formerLogins` | 3: Euclid (`@neo-gpt`), Julia (`@neo-gemini-pro`, `@neo-gemini-3-1-pro`), Mnemosyne (`@neo-fable`). `@neo-opus-4-7` and `@neo-claude-opus` live only in `identityRoots.mjs` |
| A node that holds a former spelling (reclaimed) | none |
| `MESSAGE` nodes with `to`/`from` in any former spelling's storage variants | **0 of 76,108** |
| `neo-agent-memory` rows whose `agentIdentity` is a former spelling | **0 of 61,397** |
| Edges whose source or target is a former spelling | **1** (`IMPLEMENTS`) |

**What this changes in my design note (item 1):** legacy ownership is already applied by data. The rename migrations re-keyed both stores, so retiring the `getMailboxIdentityStorageVariants` widening and the former scan in `resolveMemoryTrustTier` at the gate strands no history. The re-key fork collapses to one `IMPLEMENTS` edge. Re-keying stays the rule for any future rename.

**Still open for AC-1:** the founding-source side, which sorts each subject into a singleton seed, an ambiguous founding or a non-rostered identity, with the remedy and binding authority for each. That part goes into a committed, re-runnable inventory script in this leaf's PR, not a one-off probe.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-18T02:12:21Z @neo-opus-ada referenced in commit `179b4e1` - "feat(memory-core): mailbox and trust reads drop former logins (#1814)

Once the cutover activates selection by subject, getMailboxIdentityStorageVariants no longer widens a direct identity through its former logins, listMessages stops resolving a former login in its to and fromIdentity filters, and resolveMemoryTrustTier reads the exact node first and confers no tier from a former spelling. Before activation all three keep today's resolution. An activation that cannot be read refuses in the mailbox and confers no former tier in memory trust, never the dormant answer."
- 2026-11-18T02:12:21Z @neo-opus-ada referenced in commit `2646023` - "feat(graph): former logins confer no member weight at cutover (#1814)

readInternalAuthorLogins counts formerLogins as members only while selection by subject is dormant. GitHub reports a renamed account's history under its current login, so after the cutover a released login confers no member weight on whoever registers it."
- 2026-11-18T02:12:21Z @neo-opus-ada referenced in commit `22bbd74` - "feat(content-trust): a former login confers no tier of its own (#1814)

classifyAuthorTrust drops the former-login tier map. GitHub reports a renamed account's content under its current login, so a former login only ever names whoever registered it since; with org standing that author is repo-trusted, never the renamed resident."
- 2026-11-18T02:12:22Z @neo-opus-ada referenced in commit `7db7ce5` - "feat(scripts): the cutover inventory and its guarded activation (#1814)

subjectBindings.mjs gains inventory, a read-only census of what activating selection by subject would meet: the founding side, every identity with its binding or remedy and binding authority, every former spelling with its stored rows, and the blockers. activate writes the activation record on the reviewed inventory revision only, refusing while the inventory moved or still has a blocker."
- 2026-11-18T02:12:22Z @neo-opus-ada referenced in commit `6c8e7f6` - "test(conversation-trust): standing yields repo-trusted (#1814)

projectConversationTrust follows classifyAuthorTrust: a former login with org standing is repo-trusted and its GitHub URLs stay untouched."
- 2026-11-18T02:13:39Z @neo-opus-ada cross-referenced by PR #1834
- 2026-11-18T02:20:02Z @neo-opus-ada referenced in commit `384b076` - "docs(memory-core): source comments name no ticket (#1814)

Durable comments describe current behavior; the tracking lives in the commits and the PR, as the ticket-archaeology guard requires."
- 2026-11-18T02:20:33Z @neo-opus-ada marked this issue as being blocked by #1835
- 2026-11-18T02:46:40Z @neo-opus-ada referenced in commit `897c28e` - "chore(build): merge dev for the onnxruntime CI fix (#1814)

Brings in #1836 (.npmrc onnxruntime-node-install-cuda=skip) so this branch's npm ci jobs install again."
### @neo-opus-ada - 2026-11-18T03:04:12Z

**Handover, 2026-11-18 03:05Z (session sunset, solo refresh; the ticket stays mine).**

- **State:** PR #1834 is green at `897c28e6` (21 pass, 3 skipped) and requested from @neo-gemini-maryam. That head is a ticketed dev merge that brought in #1836's CI fix.
- **The next session owns:** the review response on #1834. After merge, the Post-Merge Validation above, starting with the inventory rerun from the deployed tree after a dormant cut.
- **Waiting on others:** identity publication on the canonical plane (`publication-not-provisioned`). I asked @neo-gpt-sophie which ticket provisions it; until then the bind (AC-2) and the activation (AC-7) cannot run, and #1815 owns them.
- **Anchors:** the live inventory's revision is `cutover-inventory-401535547c1f…` (the PR body has all of it); Chroma: memories 0 of 61,434 under a former spelling, session summaries 1,674 of 6,048.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-18T04:24:45Z @neo-opus-ada referenced in commit `d36d2ae` - "fix(memory-core): a row counts for every former spelling on it (#1814)

readFormerSpellings credited a message or an edge to its first former spelling only, so a reclaimed second spelling holding rows read as clear instead of ambiguous legacy ownership. Each row now counts once for every distinct former spelling it touches. A spec arm holds a message and an edge between two former spellings and a self-message."
- 2026-11-18T05:04:31Z @tobiu referenced in commit `f84726e` - "feat(memory-core): former logins lose standing at the cutover (#1814) (#1834)

* feat(mailbox): a former login stops redirecting after activation (#1814)

Once the cutover's activation record exists, `validateMailboxTarget`
no longer resolves a former login to the identity that left it: the
spelling is provenance only, so a target with no node of its own is
refused. Dormant planes keep alias-first addressing.

* feat(orchestrator): former logins stop owning dated items after activation (#1814)

`resolveKnownIdentities` keeps a member's former logins addressable only
while selection by subject is dormant. After the cutover a former login
no longer delivers, so its overdue rows fall back to `AGENT:*` instead
of failing at send.

* feat(memory-core): mailbox and trust reads drop former logins (#1814)

Once the cutover activates selection by subject, getMailboxIdentityStorageVariants no longer widens a direct identity through its former logins, listMessages stops resolving a former login in its to and fromIdentity filters, and resolveMemoryTrustTier reads the exact node first and confers no tier from a former spelling. Before activation all three keep today's resolution. An activation that cannot be read refuses in the mailbox and confers no former tier in memory trust, never the dormant answer.

* feat(graph): former logins confer no member weight at cutover (#1814)

readInternalAuthorLogins counts formerLogins as members only while selection by subject is dormant. GitHub reports a renamed account's history under its current login, so after the cutover a released login confers no member weight on whoever registers it.

* feat(content-trust): a former login confers no tier of its own (#1814)

classifyAuthorTrust drops the former-login tier map. GitHub reports a renamed account's content under its current login, so a former login only ever names whoever registered it since; with org standing that author is repo-trusted, never the renamed resident.

* feat(scripts): the cutover inventory and its guarded activation (#1814)

subjectBindings.mjs gains inventory, a read-only census of what activating selection by subject would meet: the founding side, every identity with its binding or remedy and binding authority, every former spelling with its stored rows, and the blockers. activate writes the activation record on the reviewed inventory revision only, refusing while the inventory moved or still has a blocker.

* test(conversation-trust): standing yields repo-trusted (#1814)

projectConversationTrust follows classifyAuthorTrust: a former login with org standing is repo-trusted and its GitHub URLs stay untouched.

* docs(memory-core): source comments name no ticket (#1814)

Durable comments describe current behavior; the tracking lives in the commits and the PR, as the ticket-archaeology guard requires.

* fix(memory-core): a row counts for every former spelling on it (#1814)

readFormerSpellings credited a message or an edge to its first former spelling only, so a reclaimed second spelling holding rows read as clear instead of ambiguous legacy ownership. Each row now counts once for every distinct former spelling it touches. A spec arm holds a message and an edge between two former spellings and a self-message."
- 2026-11-18T05:04:32Z @tobiu closed this issue

