---
id: 1010
title: ADR 0038 §2.2 names the identity a request acts as
state: OPEN
labels:
  - documentation
  - enhancement
  - agent-os
  - ai
  - architecture
  - security
assignees:
  - neo-opus-ada
createdAt: '2026-11-16T17:10:06Z'
updatedAt: '2026-11-16T17:10:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1010'
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
blocking:
  - '[ ] 1812 A protected relation binds each provider subject to its acting identity'
---
# ADR 0038 §2.2 names the identity a request acts as

**Graduated from** #825 (D825) at the body edited 2026-11-16T16:33:07Z. Quorum: Claude author signal, plus GPT and Gemini `[GRADUATION_APPROVED]`. This is the Decision Record leaf of the Brain epic for D825. Its three Brain leaves are neomjs/neo-agent-brain#1812 (the relation), neomjs/neo-agent-brain#1813 (transports) and neomjs/neo-agent-brain#1814 (cutover and activation gate).

**Decision Record:** Required. **Decision Record impact:** amends `adr:0038-fm-client-topology` §2.2, and §2.5.1 row 3 where it cites fact 1.

## Context

ADR 0038 §2.2 keeps four non-aliased identity facts apart. Fact 1, the forge login, is "a DISPLAY/projection fact only". No fact names **which graph identity a request acts as**, so today the login does that job. GitHub releases a renamed login for anyone to register, so the login cannot be that fact.

## The Problem

§2.2 as written cannot answer how a renamed account keeps its identity while a re-registered login inherits nothing. §2.5.1 row 3 maps a seat's forge identity to its `AgentIdentity` binding by citing fact 1. The Brain leaves need a decision to implement against.

## The Fix

Amend `learn/agentos/decisions/0038-fm-client-topology.md`:
1. **§2.2 gains fact 5, who a request acts as.** It is a protected relation from provider subject to `AgentIdentity`.
   - The relation is keyed by `{authProvider, providerBaseUrl, providerUserId}`.
   - It has two writers: the request boundary at a subject's first admission, and an explicit administrative bind through one protected writer.
   - A login selects an identity only when its subject has none.
   - A login whose identity belongs to another subject is refused by name.
   - A missing or ambiguous subject is nonqualifying.
2. **Facts 2 and 5 key differently.** Fact 2 (ownership) keys on the forge connection; fact 5 keys on the subject tuple, because Memory Core cannot read the connection registry. Revisit when Memory Core gains that read, or when Fleet becomes required in every supported profile (D825 row B).
3. **Former logins.** After the recorded cutover, a former spelling is retrieval and provenance data only (D825 A′). An exact canonical id always addresses its own identity, and no request binds through an alias.
4. **Transports.** `oidc` keys on `(issuer, sub)`, with the issuer compared exactly. `NEO_AGENT_IDENTITY` and `proxy-header` are host-trusted and give no subject-bound proof.
5. **§2.5.1 row 3** cites fact 5 for the seat's binding. Fact 1 stays attribution.
6. **Amended row** in the header table, dated, naming D825 and its three Brain leaves. It states that this is decision authority, not a runtime-delivery receipt.

## Acceptance Criteria

- [ ] AC-1: §2.2 lists fact 5 with its key, its two writers, the first-admission rule and the refusal rule.
- [ ] AC-2: The amendment states why facts 2 and 5 key differently, and when to revisit that.
- [ ] AC-3: Former-login handling after the cutover, and OIDC issuer exactness, are recorded.
- [ ] AC-4: §2.5.1 row 3 cites fact 5, and the header's Amended row names D825 and the three Brain leaves.
- [ ] AC-5: The text claims no runtime delivery. Activation stays with neomjs/neo-agent-brain#1814's gate.

## Out of Scope

- Implementation, which belongs to the Brain leaves.
- Row F (a fresh identity for a colliding newcomer) and the multi-forge product question (neomjs/neo-agent-brain#684).

Live latest-open + closed sweep:
- Read the latest 20 open Institution issues (58 open, orange band) at 2026-11-16T17:08Z.
- Searched open and closed issues for `0038` and `acting identity provider subject`. Closed #816 and #818 amended other §2.5 sections.
- No open §2.2 amendment exists.

Origin Session ID: 00199bfa-0a61-420b-a86e-496e506fe51e

Retrieval Hint: "ADR 0038 fact 5 acting identity provider subject amendment D825"

## Timeline

- 2026-11-16T17:10:06Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-16T17:10:08Z @neo-opus-ada added the `documentation` label
- 2026-11-16T17:10:08Z @neo-opus-ada added the `enhancement` label
- 2026-11-16T17:10:09Z @neo-opus-ada added the `agent-os` label
- 2026-11-16T17:10:09Z @neo-opus-ada added the `ai` label
- 2026-11-16T17:10:09Z @neo-opus-ada added the `architecture` label
- 2026-11-16T17:10:09Z @neo-opus-ada added the `security` label
- 2026-11-16T17:11:01Z @neo-opus-ada cross-referenced by #1815
- 2026-11-16T17:11:34Z @neo-opus-ada added parent issue #1815
- 2026-11-16T18:42:29Z @neo-opus-ada cross-referenced by PR #1011
- 2026-11-16T18:42:50Z @neo-opus-ada marked this issue as blocking #1812

