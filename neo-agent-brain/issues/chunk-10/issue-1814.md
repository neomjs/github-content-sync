---
id: 1814
title: Cutover binds existing identities and retires former-login redirects
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees: []
createdAt: '2026-11-16T17:09:42Z'
updatedAt: '2026-11-16T17:09:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1814'
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
<!-- /binding-inventory -->
4. **The activation gate.** #1812, the transports leaf and this leaf activate together, after the bind, so no provider-authenticated transport keeps a login fallback.

## Acceptance Criteria

- [ ] AC-1: The inventory exists on the live plane before any bind, with every unresolved row named together with its remedy and binding authority.
- [ ] AC-2: Singleton seeds bind. Ambiguous-founding and non-rostered rows stay nonqualifying until an explicit bind, and no ordinary property is retrofitted.
- [ ] AC-3 (positive): Every row an identity owned under a former spelling stays readable by exactly that identity, and by no identity that later holds the spelling. The widening retires only after this holds.
- [ ] AC-4 (control 4): Provider unavailable, ambiguous founding input and ambiguous legacy ownership stay named unresolved states. Historical labels authorize no delivery, inbox access, grant, read-state change or content-trust tier.
- [ ] AC-5 (control 5): Only the inventory's protected administrative result supplies collision evidence. Ordinary identity properties stay candidates.
- [ ] AC-6 (control 3): An import or cutover claim that conflicts with a bound identity refuses, without denying a valid canonical address or binding.
- [ ] AC-7 (the gate): The three Brain leaves activate together after the bind. Before it, no changed addressing or trust path is active, and any enabled compatibility path passes controls 1 to 5.
- [ ] AC-8: Old handles still resolve in Knowledge Base retrieval and provenance, and stop resolving in A2A addressing.

## Out of Scope

- The relation and HTTP PAT binding, which belong to #1812.
- Stdio and OIDC transports, which belong to the transports leaf.
- Protected aliases owned by the plane (D825 row B, not adopted). Revisit only if persistent, independently administered aliases become a product requirement.

## Avoided Traps

- Retiring the read widening before legacy ownership is applied, which strands pre-rename history.
- Enabling exact-id-first addressing before redirect retirement. In that window `@old-login` still redirects while another subject may hold it.

Owner: unowned-rationale. Claimable by any maintainer; the Fleet bind path is #1297. If it is still unclaimed when #1812 merges, the epic steward (`@neo-opus-ada`) takes it.

Live latest-open + closed sweep: the same run as #1812 at 2026-11-16T17:06Z. It covered the latest 20 open Brain issues (156 open) and open and closed searches for `former login alias` and related nouns. Closed #1174 and #1190 were read; their A2A addressing prescription is superseded here and their retrieval intent kept. No equivalent open owner was found. The A2A claim sweep found no competing claim.

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

