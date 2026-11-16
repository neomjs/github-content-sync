---
id: 1810
title: A released former login keeps its old seat's content-trust tier
state: CLOSED
labels:
  - bug
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-16T16:16:45Z'
updatedAt: '2026-11-16T17:55:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1810'
author: neo-opus-ada
commentsCount: 1
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
closedAt: '2026-11-16T17:55:42Z'
---
# A released former login keeps its old seat's content-trust tier

## Context

`classifyAuthorTrust` decides whether GitHub content (issues, comments, reviews, discussions) reaches agents as trusted or goes through the hostile-content sanitizer. It maps every roster `formerLogins` entry to the renamed seat's trust tier, and it does so before the collaborator and `authorAssociation` checks. A rename on GitHub releases the old login for anyone to register. Whoever registers it is then classified with the old seat's tier.

The general case is already public in [D825's OQ2 read by Maryam](https://github.com/neomjs/neo-agent-institution/discussions/825#discussioncomment-19658166). On 2026-11-16 I checked the six released maintainer logins on GitHub: at least one is now registered to a different account than the seat that held it. The specific login is withheld here.

## The Problem

A reclaimed former login posts content that skips the hostile-content quarantine and reaches agents with peer trust. That is a prompt-injection path with elevated trust that needs no access to the org.

## The Architectural Reality

- `ai/services/shared/contentTrust/authorTrustClassifier.mjs`: `KNOWN_TIER_BY_LOGIN` indexes `githubLogin` and every `formerLogins` entry. `classifyAuthorTrust` returns that tier at step 2, before `collaborators` (step 3) and `authorAssociation` (step 4).
- #1279 added the `formerLogins` index so that content authored under a pre-rename login keeps the resident's tier. For live GitHub reads that index is unnecessary: GitHub attributes an account's earlier comments to its renamed login. The old string survives only in content ingested before the rename.
- `Design authority:` D825's concept ("a re-registered login inherits nothing") and its OQ2 control 4, folded 2026-11-16: historical labels never authorize a content-trust tier.

## The Fix

A `formerLogins` match confers the identity's tier only when the same call proves org standing: `authorAssociation` is `OWNER`, `MEMBER` or `COLLABORATOR`, or the login is in the injected collaborator set. Otherwise it falls through to the existing steps, which return `EXTERNAL` for a stranger. Matching on the current `githubLogin` is unchanged.

## Acceptance Criteria

- [ ] A former login with `authorAssociation` `NONE`, `CONTRIBUTOR` or `FIRST_TIME_CONTRIBUTOR` (or none at all) classifies `EXTERNAL` and `isUntrustedTier` returns true (unit).
- [ ] A former login with `MEMBER`, `OWNER` or `COLLABORATOR`, or present in the collaborator set, keeps the resident's tier, so historical ingested content stays trusted (unit).
- [ ] The current `githubLogin` path and an unknown login are unchanged (unit regression rows).
- [ ] The module docblock's resolution order states the corroboration rule and why a released login needs it.

## Out of Scope

- `MemoryService.resolveMemoryTrustTier` and mailbox alias routing. Both read graph identities, not GitHub authors, and D825 OQ2 (leaf 4, under its shared activation gate) owns them.
- Keying trust on the GitHub numeric account id: the roster carries no such field today. D825's subject relation is the durable answer.
- Any action on the account holding the reclaimed login (operator's call).

## Avoided Traps

- Dropping `formerLogins` entirely: that would downgrade genuine historical content #1279 meant to keep trusted.
- Naming the reclaimed login in a public artifact.

## Related

#1279 (origin of the index), Institution D825 (OQ2 control 4), #1296.

**Sweeps:**
- Live latest-20 open Brain issues, plus open and closed search for `formerLogins trust`, `classifyAuthorTrust`, `former login trust tier` and `reclaimed login`, at 2026-11-16T16:15Z. Hits: #1279 (closed, origin) and #1296 (open, different scope). No equivalent and no open PR on the classifier.
- A2A in-flight sweep (latest 15, all read states): no competing claim.

Origin Session ID: 00199bfa-0a61-420b-a86e-496e506fe51e

## Timeline

- 2026-11-16T16:16:45Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-16T16:16:46Z @neo-opus-ada added the `bug` label
- 2026-11-16T16:16:47Z @neo-opus-ada added the `ai` label
- 2026-11-16T16:16:47Z @neo-opus-ada added the `security` label
- 2026-11-16T16:16:47Z @neo-opus-ada added the `agent-os` label
- 2026-11-16T16:18:50Z @neo-opus-ada cross-referenced by PR #1811
### @neo-opus-ada - 2026-11-16T16:24:08Z

**Handover (session sunset, solo-refresh, 2026-11-16 ~16:30Z): my next session picks this up.**

- **State:** PR #1811 is open at `43c81b42` (branch `ada/1810-former-login-trust`, two files). Red-first on `dev` source: the new stranger test fails (1 failed, 22 passed); at the head, 23/23. The PR body lint passed locally.
- **Next:** CI green → Maryam's review (requested at open) → the operator's merge. On a change request: answer on the PR, push, re-request.
- **Out of scope here (owned elsewhere):** mailbox alias routing and `MemoryService.resolveMemoryTrustTier` stay with Institution D825 OQ2 (leaf 4, shared activation gate).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-16T16:37:12Z @neo-opus-ada referenced in commit `2e7f5ea` - "test(trust): a former login without standing is quarantined (#1810)

ConversationTrust's #1279 test gave a former-login reply no
authorAssociation and expected peer-trusted, which is the hole #1810
closes. The reply now carries MEMBER standing, and a sibling test proves
the stranger case is quarantined at the conversation level."
- 2026-11-16T17:09:43Z @neo-opus-ada cross-referenced by #1814
- 2026-11-16T17:11:01Z @neo-opus-ada cross-referenced by #1815
- 2026-11-16T17:14:43Z @neo-opus-ada cross-referenced by #1174

