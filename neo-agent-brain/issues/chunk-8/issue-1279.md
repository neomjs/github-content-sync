---
id: 1279
title: 'Trust-tier resolution for graph identities, formerLogins, and GitHub authorAssociation'
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-25T22:51:47Z'
updatedAt: '2026-10-26T01:57:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1279'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: 875
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 2
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-26T01:57:51Z'
---
# Trust-tier resolution for graph identities, formerLogins, and GitHub authorAssociation

## Context

Observed live on `2026-10-25T22:32Z–22:37Z` during the first `/peer-role` cycle on `neomjs/neo` Discussion `#20111` (`DC_kwDODSospM4BJLuI`, comment `19184520`) and captured in `@neo-fable-clio`'s defect note (`MESSAGE:8d7e2228-41e9-44a3-a912-966d365f5b20`):
1. Calling `get_discussion_conversation({ repo: 'neomjs/neo', discussion_number: 20111, last_n: 5 })` classified `@neo-gemini-maryam`'s comment (`DC_kwDODSospM4BJLuI`) as `authorTrust: "external"` and set `contentTrust.quarantined: 14`, rewriting every `[QUARANTINED_URL: github.com]` line citation to `[QUARANTINED_URL: github.com]`.
2. Calling `query_raw_memories` returned `trustTier: "unclassified"` for memories authored by both `@neo-gemini-pro` (`Julia`) and `@neo-gemini-maryam` (`Maryam`), while returning `trustTier: "peer-trusted"` for Claude and GPT seats.

This ticket promotes `MESSAGE:8d7e2228-41e9-44a3-a912-966d365f5b20` and serves as the **trust-tiers leaf** of Epic #875 (`Runtime identity facts come from the plane's graph, not identityRoots`, which named `trust tiers` in its problem scope alongside participation gates `#874`/`#879`/`#880`/`#916`).

Design authority:
- Epic #875 (`"Every runtime reader takes the graph's answer. Plane-side services read identity nodes in-process."`)
- `#700` (`#700#issuecomment-6240532059`: `@neo-gemini-maryam` remains absent from `ai/graph/identityRoots.mjs` as the live acceptance control for a non-rostered Fleet-Manager-created seat)
- `ai/graph/identityRoots.mjs:18-20` (`"properties.formerLogins keeps every earlier githubLogin so strings that cannot migrate (corpus author fields, old @mentions) still resolve to the resident."`)

## The Problem

Three trust-classification seams currently ignore `formerLogins`, live `AgentIdentity` graph nodes, and GitHub's `authorAssociation`:

1. **`MemoryService.resolveMemoryTrustTier` (`ai/services/memory-core/MemoryService.mjs:400, 442`) ignores both `formerLogins` and graph-provisioned `AgentIdentity` nodes:**
   - Line `400` builds `static identityTrustTiers = new Map(IDENTITIES.map(identity => [identity.id, identity.properties?.trustTier || TRUST_TIERS.UNCLASSIFIED]))` purely from `identity.id` in `identityRoots.mjs`.
   - When a resident was renamed in `identityRoots.mjs` (`#1155`: `@neo-gemini-julia` carrying `formerLogins: ['@neo-gemini-pro', '@neo-gemini-3-1-pro']`, `@neo-fable-mnemosyne` carrying `formerLogins: ['@neo-fable']`, `@neo-gpt-euclid` carrying `formerLogins: ['@neo-gpt']`, `@neo-opus-ada` carrying `formerLogins: ['@neo-opus-4-7']`, `@neo-opus-grace` carrying `formerLogins: ['@neo-claude-opus']`), every memory row stamped with a `formerLogins` `agentIdentity` resolves to `TRUST_TIERS.UNCLASSIFIED`.
   - When `Server.mjs:626-662` auto-provisions an authenticated `AgentIdentity` node in `GraphService` (such as `@neo-gemini-maryam`), it stamps `properties.trustTier: TRUST_TIERS.INTERNAL_AUTHORED` (`'internal-authored'`) on the graph node, but `MemoryService.resolveMemoryTrustTier` never reads the `AgentIdentity` node in `GraphService`, so every memory written by an auto-provisioned seat resolves to `'unclassified'`.
2. **`authorTrustClassifier.mjs` (`ai/services/shared/contentTrust/authorTrustClassifier.mjs:37-41`) ignores `formerLogins`:**
   - `KNOWN_TIER_BY_LOGIN` only maps `identity.properties.githubLogin`, ignoring `identity.properties.formerLogins`. Any GitHub issue, PR, or discussion comment authored under a `formerLogins` login (`neo-gemini-pro`, `neo-fable`, `neo-gpt`, etc.) falls through to `TRUST_TIERS.EXTERNAL` unless rescued by `collaborators`.
3. **`github-workflow` conversation projection (`conversationTrust.mjs`, `IssueService.mjs`, `DiscussionService.mjs`, `PullRequestService.mjs`, `DiscussionSyncer.mjs`, `IssueSyncer.mjs`, `PullRequestSyncer.mjs`) omits GitHub `authorAssociation`:**
   - `IssueService.getConversation` (`IssueService.mjs:113`) and `DiscussionService.getConversation` (`DiscussionService.mjs:106`) call `projectConversationTrust(...)` without passing `collaborators`, and `GET_DISCUSSION_CONVERSATION` (`discussionQueries.mjs:57-114`) / `GET_ISSUE_CONVERSATION` (`issueQueries.mjs`) do not select `authorAssociation` on the root node, comments, or replies.
   - Consequently, any GitHub organization member or repo collaborator not hardcoded in `identityRoots.mjs` (including `@neo-gemini-maryam` and external human maintainers on other deployments) is classified as `authorTrust: "external"` and has all links — including first-party `[QUARANTINED_URL: github.com]` links — rewritten to `[QUARANTINED_URL: github.com]` by `astroturfSanitizer.mjs:128-139`.
   - Meanwhile, `ai/services/fleet/openWorkHolder.mjs:36` already treats `['OWNER', 'MEMBER', 'COLLABORATOR'].includes(authorAssociation)` as maintainer-trusted.

## The Architectural Reality

- `ai/services/memory-core/MemoryService.mjs` (`lines 400-458`): owns `identityTrustTiers`, `resolveMemoryTrustTier(metadata)`, and `matchesMinTrustTier(metadata, minTrustTier)`. Has in-process access to `GraphService` (`GraphService.db?.nodes` / `GraphService.getNode`) and `resolveAgentIdentityAlias` (`ai/services/graph/agentFamilyResolution.mjs:114`).
- `ai/services/shared/contentTrust/authorTrustClassifier.mjs` (`lines 37-81`): pure classifier `classifyAuthorTrust(login, { collaborators, authorAssociation })`.
- `ai/services/github-workflow/shared/conversationTrust.mjs` (`lines 41-64`): `projectNode(node, ctx, path)` calls `classifyAuthorTrust(node.author?.login, { collaborators: ctx.collaborators, authorAssociation: node.authorAssociation })`.
- `ai/services/github-workflow/queries/discussionQueries.mjs`, `issueQueries.mjs`, `pullRequestQueries.mjs`: GraphQL queries for live conversation reads and markdown syncers.

Structure map (`npm run ai:structure-map -- --files --loc` run `2026-10-25T22:46Z`): `ai/services/memory-core`, `ai/services/shared/contentTrust`, `ai/services/github-workflow/shared`, and `ai/services/github-workflow/queries` are the existing owning folders; no new `.mjs` files or directory moves required.

## The Fix

1. **Graph-first + `formerLogins` trust resolution in `MemoryService.resolveMemoryTrustTier` (`MemoryService.mjs:400-445`):**
   - Read the `AgentIdentity` node's `properties.trustTier` from `GraphService` when available (resolving `formerLogins` via `resolveAgentIdentityAlias` or graph `formerLogins` property when the direct node ID is a legacy handle).
   - Seed `MemoryService.identityTrustTiers` with both `identity.id` and every handle in `identity.properties.formerLogins` so unit tests or unseeded in-memory runs resolve `@neo-gemini-pro`, `@neo-fable`, `@neo-gpt`, `@neo-opus-4-7`, and `@neo-claude-opus` to their resident's `trustTier` (`peer-trusted`).
2. **Index `formerLogins` and accept `authorAssociation` in `authorTrustClassifier.mjs` (`lines 37-81`):**
   - Include `identity.properties.formerLogins` in `KNOWN_TIER_BY_LOGIN`.
   - Accept optional `authorAssociation` in `classifyAuthorTrust(login, { collaborators, authorAssociation } = {})`: when `login` is not in `KNOWN_TIER_BY_LOGIN`, classify `authorAssociation === 'OWNER'` as `TRUST_TIERS.OWNER` (or `REPO_TRUSTED`) and `['MEMBER', 'COLLABORATOR'].includes(authorAssociation)` as `TRUST_TIERS.REPO_TRUSTED`.
3. **Select `authorAssociation` in GitHub conversation/sync GraphQL queries and forward in `conversationTrust.mjs`:**
   - Add `authorAssociation` to authored node selections in `GET_DISCUSSION_CONVERSATION`, `FETCH_DISCUSSIONS_FOR_SYNC`, `GET_ISSUE_CONVERSATION`, `FETCH_ISSUES_FOR_SYNC`, and PR conversation queries.
   - Pass `authorAssociation: node.authorAssociation` into `classifyAuthorTrust` inside `conversationTrust.mjs:46` (`projectNode`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `MemoryService.resolveMemoryTrustTier(metadata)` (`ai/services/memory-core/MemoryService.mjs:442`) | Epic #875 + `identityRoots.mjs:18-20` + `Server.mjs:633` | Resolves `metadata.agentIdentity` against the live `AgentIdentity` graph node's `properties.trustTier` first (including `formerLogins` alias resolution), falling back to the `identityTrustTiers` map (which indexes both `id` and `formerLogins`) | `TRUST_TIERS.UNCLASSIFIED` when neither graph nor seed matches | JSDoc on `resolveMemoryTrustTier` | Unit test covering graph-only auto-provisioned seat (`@neo-gemini-maryam` → `internal-authored`) and `formerLogins` (`@neo-gemini-pro` → `peer-trusted`) |
| `classifyAuthorTrust(login, { collaborators, authorAssociation })` (`ai/services/shared/contentTrust/authorTrustClassifier.mjs:58`) | `identityRoots.mjs:18-20` + `openWorkHolder.mjs:36` | Resolves `login` against both `githubLogin` and `formerLogins` in `KNOWN_TIER_BY_LOGIN`; when unmatched, classifies `collaborators` or `authorAssociation` in `['OWNER', 'MEMBER', 'COLLABORATOR']` as `TRUST_TIERS.REPO_TRUSTED` | `TRUST_TIERS.EXTERNAL` (or `UNCLASSIFIED` for empty login) | JSDoc on `classifyAuthorTrust` | Unit test in `authorTrustClassifier.spec.mjs` covering `formerLogins` (`neo-gemini-pro`) and `authorAssociation: 'MEMBER'` (`neo-gemini-maryam`) |
| `projectNode` (`ai/services/github-workflow/shared/conversationTrust.mjs:41`) + GraphQL queries | `conversationTrust.mjs` JSDoc | Forwards `authorAssociation: node.authorAssociation` to `classifyAuthorTrust`; GraphQL conversation/sync queries select `authorAssociation` on root, comments, and replies | Omitted `authorAssociation` falls back to `collaborators` / `EXTERNAL` | JSDoc on `conversationTrust.mjs` | Unit test in `conversationTrust.spec.mjs` verifying a `MEMBER` comment with `github.com/neomjs/*` links is not quarantined |

## Decision Record impact

`aligned-with` Epic #875 (`Runtime identity facts come from the plane's graph, not identityRoots`) and `#1174` (`Old handles in artifacts resolve to the renamed identity instead of a dead login`).

## Acceptance Criteria

- [ ] `MemoryService.resolveMemoryTrustTier` resolves auto-provisioned `AgentIdentity` graph nodes (e.g. `@neo-gemini-maryam` with `properties.trustTier: 'internal-authored'`) from `GraphService` rather than returning `'unclassified'`.
- [ ] `MemoryService.resolveMemoryTrustTier` and `authorTrustClassifier.classifyAuthorTrust` resolve every handle in `properties.formerLogins` (`@neo-gemini-pro`, `@neo-gemini-3-1-pro`, `@neo-fable`, `@neo-gpt`, `@neo-opus-4-7`, `@neo-claude-opus`) to the owning resident's `trustTier` (`peer-trusted`).
- [ ] `classifyAuthorTrust` and `projectConversationTrust` accept `authorAssociation` (`OWNER`, `MEMBER`, `COLLABORATOR` → `repo-trusted`), and the GitHub GraphQL conversation queries (`GET_DISCUSSION_CONVERSATION`, `GET_ISSUE_CONVERSATION`, and sync queries) select `authorAssociation` on root, comment, and reply nodes so non-rostered org members (`@neo-gemini-maryam`) are not classified `external` or URL-quarantined.
- [ ] Unit tests in `test/playwright/unit/` cover all three arms (`formerLogins`, graph-provisioned `AgentIdentity`, and `authorAssociation: 'MEMBER'`).

## Out of Scope

- Migrating or rewriting historical `agentIdentity` strings in ChromaDB (`#1149` / `#1174`).
- Removing `identityRoots.mjs` imports from unrelated services (`#875` sibling leaves).

## Avoided Traps

- **Adding `@neo-gemini-maryam` to `identityRoots.mjs` to silence the symptom:** Violates `#700#issuecomment-6240532059` (where `@neo-gemini-maryam` is the live acceptance control for a non-rostered FM-created seat) and fails on every other operator's deployment (`#875`).
- **Allowlisting `github.com` URLs globally in `astroturfSanitizer.mjs` for `external` authors:** External spam/astroturf accounts can link to external GitHub repos (`github.com/spammer/tool`); fixing the author-trust classification (`formerLogins` + `authorAssociation` + graph node) is the structural fix.

## Related

- Parent Epic: #875
- Related: #700, #1155, #1174, `neomjs/neo` Discussion `#20111` (`DC_kwDODSospM4BJLuI`), `MESSAGE:8d7e2228-41e9-44a3-a912-966d365f5b20`

unowned-rationale: Parked for pickup after current active PR reviews merge; linked as sub-issue of Epic #875.

Live latest-open sweep: checked latest 25 open issues in `neomjs/neo-agent-brain` at `2026-10-25T22:48Z` and sub-issues of `#875` (`#874`, `#879`, `#880`, `#916` all closed); no equivalent trust-tier leaf found.
A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-25T22:48Z`; only Clio's defect-note `MESSAGE:8d7e2228-41e9-44a3-a912-966d365f5b20` (which this ticket promotes), no competing `[lane-claim]`.
MC sweep: `query_raw_memories` on `authorTrustClassifier formerLogins resolveMemoryTrustTier unclassified external QUARANTINED_URL`, 6 results, no prior decision declining this fix.
Own-assignment sweep: 0 open issues assigned in `neomjs/neo-agent-brain`.

Origin Session ID: 06cf8283-28b4-42b3-b137-ae9da8ab6995
Retrieval Hint: "resolveMemoryTrustTier authorTrustClassifier formerLogins authorAssociation QUARANTINED_URL neo-gemini-maryam neo-gemini-pro"

## Timeline

- 2026-10-25T22:51:48Z @neo-gemini-maryam added the `bug` label
- 2026-10-25T22:51:48Z @neo-gemini-maryam added the `ai` label
- 2026-10-25T22:51:48Z @neo-gemini-maryam added the `architecture` label
- 2026-10-25T22:51:49Z @neo-gemini-maryam added the `security` label
- 2026-10-25T22:51:49Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-25T22:53:18Z @neo-gemini-maryam added parent issue #875
- 2026-10-26T00:59:48Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-26T01:17:54Z @neo-gemini-maryam cross-referenced by PR #1284
- 2026-10-26T01:25:57Z @neo-gemini-maryam referenced in commit `93602b2` - "fix(github-workflow): replace legacy ticket-ref-ok markers in issueQueries (#1279)"
- 2026-10-26T01:57:52Z @tobiu referenced in commit `533724f` - "fix(trust): resolve trust tiers from graph identities, formerLogins, and authorAssociation (#1279) (#1284)

* fix(trust): resolve trust tiers from graph identities, formerLogins, and authorAssociation (#1279)

* fix(github-workflow): replace legacy ticket-ref-ok markers in issueQueries (#1279)"
- 2026-10-26T01:57:52Z @tobiu closed this issue
- 2026-10-26T02:11:06Z @neo-gemini-maryam cross-referenced by PR #1288

