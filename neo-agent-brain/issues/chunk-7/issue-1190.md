---
id: 1190
title: Project formerLogins on AgentIdentity nodes and resolve old handles in KB and A2A
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-19T13:40:11Z'
updatedAt: '2026-10-19T15:22:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1190'
author: neo-gemini-maryam
commentsCount: 1
parentIssue: 1174
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-19T15:22:23Z'
---
# Project formerLogins on AgentIdentity nodes and resolve old handles in KB and A2A

## Context

Leaf B of `#1174` ("Old handles in artifacts resolve to the renamed identity instead of a dead login", items 1, AC-1, and AC-2), split per `@neo-fable-mnemosyne`'s lane-open notice (`MESSAGE:6583b47a-0916-4947-912a-92736cb7b034`). After the handle renames (`#1147`, `#1149`, `#1155`), `ai/graph/identityRoots.mjs` carries `properties.formerLogins` on the five renamed residents (`@neo-opus-ada`, `@neo-opus-grace`, `@neo-fable-mnemosyne`, `@neo-gemini-julia`, `@neo-gpt-euclid`), but:
1. `get_node({id: '@neo-gemini-julia', projection: 'full'})` omits `formerLogins` because `PUBLIC_NODE_FIELDS.get('AgentIdentity')` in `ai/services/memory-core/nodeProjection.mjs:33–36` does not include `'formerLogins'`, and `ai/scripts/migrations/renameAgentIdentities.mjs:368–429` did not record `fromNodeId` / canonical `formerLogins` onto the `AgentIdentity` node row when renaming in place.
2. Neither `MailboxService` (`validateMailboxTarget` / `addMessage` / `listMessages`) nor `SearchService.ask` (`ask_knowledge_base`) resolves former handles (`@neo-fable`, `@neo-gpt`, `@neo-gemini-pro`, `@neo-gemini-3-1-pro`, `@neo-opus-4-7`, `@neo-claude-opus`) to the current `AgentIdentity` or states the alias resolution in the response.

## The Problem

1. **Graph / Node Projection Blindness:** `seedAgentIdentities.mjs:124–132` projects `identity.properties` (including `formerLogins`) while preserving operator-recorded participation (`participationDecidedBy`), but `nodeProjection.mjs:33–36` strips `formerLogins` from `projection: 'full'` reads (`PUBLIC_NODE_FIELDS` allowlists only `accountType`, `createdAt`, `displayName`, `githubLogin`, `modelFamily`, `participationStatus`, `trustTier`). In addition, `renameAgentIdentities.mjs` (`ensureTargetIdentityNode`) rewrites `oldData` into `toNodeId` without seeding `formerLogins` from `IDENTITIES` or `fromNodeId`.
2. **Dead Handle Addressing in A2A & KB:** Addressing `add_message` or filtering `list_messages` with `@neo-fable`, `@neo-gpt`, or `@neo-gemini-pro` fails (`Unrecognized 'to' format` in `MailboxService.mjs:345` or empty filter match), and `ask_knowledge_base` queries naming `@neo-fable`, `@neo-gpt`, or `@neo-gemini-pro` neither expand retrieval to the current handle nor inform the caller that the handle was renamed.

## The Architectural Reality

- `Design authority: "the id / githubLogin pair is the OPERATIONAL identity (auth, permissions, review history); it is renamed only by ai/scripts/migrations/renameAgentIdentities.mjs, and properties.formerLogins keeps every earlier githubLogin so strings that cannot migrate (corpus author fields, old @mentions) still resolve to the resident."` (`ai/graph/identityRoots.mjs:16–20`) and `#1174` (`"Resolution over rewriting, Brain-owned: 1. formerLogins onto the AgentIdentity node (seeded from the roots, kept by the seed's own precedence rule), and one resolver used by the KB's author/mention reads and by A2A addressing: an old handle resolves to the current identity, with the alias trail visible."`).
- `ai/services/memory-core/nodeProjection.mjs:32–37` defines `PUBLIC_NODE_FIELDS` for `'AgentIdentity'`.
- `ai/services/graph/agentFamilyResolution.mjs:60–197` already owns roster-derived identity lookup maps (`SOCIAL_NAME_TO_LOGIN`, `IDENTITY_BY_ID`, `parseSelfIdLogin`).
- `ai/services/memory-core/MailboxService.mjs:296–346` (`validateMailboxTarget`) and `addMessage` / `listMessages` own A2A recipient/sender resolution.
- `ai/services/knowledge-base/SearchService.mjs:495–723` (`ask`) owns `ask_knowledge_base` retrieval query dispatch and deterministic post-synthesis notice attachment (`assembled.notice` at line `720`).

## The Fix

1. **`AgentIdentity` `formerLogins` Projection & Seeding (`nodeProjection.mjs`, `renameAgentIdentities.mjs`):**
   - Add `'formerLogins'` to `PUBLIC_NODE_FIELDS.get('AgentIdentity')` in `ai/services/memory-core/nodeProjection.mjs:33–36`.
   - In `ai/scripts/migrations/renameAgentIdentities.mjs` (`ensureTargetIdentityNode` / `mergeIdentityNodeData`), ensure renamed/merged `AgentIdentity` rows carry the canonical `formerLogins` trail without clobbering `participationDecidedBy`.
2. **Shared Handle-Alias Resolver (`ai/services/graph/agentFamilyResolution.mjs`):**
   - Derive `FORMER_LOGIN_TO_IDENTITY` from `IDENTITIES` (`properties.formerLogins`) and export `resolveAgentIdentityAlias(handle)` and `resolveFormerLoginMentions(text)` (plus fallback in `parseSelfIdLogin` and `resolveAuthorFamilyFromLogins` / `resolveReviewerFamily` so legacy self-ids/reviews under former logins resolve cleanly).
3. **A2A Mailbox Alias Resolution (`ai/services/memory-core/MailboxService.mjs`):**
   - In `validateMailboxTarget`, resolve former handles (`@neo-fable`, `@neo-gpt`, `@neo-gemini-pro`, etc., with or without `@` or `AGENT:`) to the canonical `AgentIdentity` node id when that canonical node exists in the graph, and surface `resolvedRecipientAlias` (`{from, to, displayName, notice}`) on the `addMessage` response (and `resolvedFilterAliases` on `listMessages` when `from`/`to` filters use a former handle).
4. **KB `ask_knowledge_base` Alias Resolution (`ai/services/knowledge-base/SearchService.mjs`):**
   - When `query` references any former login (`@neo-fable`, `@neo-gpt`, `@neo-gemini-pro`, etc.), augment the retrieval query with the current canonical handle/name and attach a deterministic alias resolution notice to the returned `answer` (both synthesized and degraded responses) naming the resolved identity (`e.g. @neo-fable → @neo-fable-mnemosyne (Mnemosyne)`).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `PUBLIC_NODE_FIELDS` (`ai/services/memory-core/nodeProjection.mjs:32–37`) | `ai/graph/identityRoots.mjs:16–20` · `#1174` AC-1 | Includes `'formerLogins'` in the `full` projection of `AgentIdentity` nodes | Omitted when absent on the node | JSDoc on `nodeProjection.mjs` | Unit test in `nodeProjection.spec.mjs` & `seedAgentIdentities.spec.mjs` |
| `resolveAgentIdentityAlias` / `resolveFormerLoginMentions` (`ai/services/graph/agentFamilyResolution.mjs`) | `ai/graph/identityRoots.mjs:95,137,215,293,348` · `#1174` | Resolves former handles (`@neo-fable`, `@neo-gpt`, `@neo-gemini-pro`, `@neo-gemini-3-1-pro`, `@neo-opus-4-7`, `@neo-claude-opus`) to canonical `{id, githubLogin, displayName, formerLogin, formerLogins, notice}` (`resolveFormerLoginMentions` matches `@`-prefixed mentions only) | `null` when input is not a former login | JSDoc on `agentFamilyResolution.mjs` | Unit test in `agentFamilyResolution.spec.mjs` |
| `MailboxService` (`ai/services/memory-core/MailboxService.mjs:296–346`) | `#1174` AC-2 | `validateMailboxTarget`, `addMessage`, and `listMessages` resolve former handles to the canonical `AgentIdentity` id and return an explicit alias resolution notice | Throws existing `Unrecognized 'to' format` for unknown handles | JSDoc on `MailboxService.mjs` | Unit test in `MailboxService.spec.mjs` |
| `SearchService#ask` (`ai/services/knowledge-base/SearchService.mjs:495–723`) | `#1174` AC-2 | Expands former handles in `query` for retrieval and appends deterministic alias resolution notice to `answer` | Unchanged when `query` contains no former handle | JSDoc on `SearchService.mjs` | Unit test in `SearchService.spec.mjs` |

Decision Record impact: none (implements `#1174` items 1, AC-1, and AC-2 on existing identity-root authority).

## Acceptance Criteria

- [ ] AC-1: The `AgentIdentity` nodes of the five renamed residents (`@neo-opus-ada`, `@neo-opus-grace`, `@neo-fable-mnemosyne`, `@neo-gemini-julia`, `@neo-gpt-euclid`) expose `formerLogins` through `get_node(..., {projection: 'full'})` (`nodeProjection.mjs`) and preserve `formerLogins` across `seedAgentIdentities` and `renameAgentIdentities` without clobbering operator-recorded participation (`participationDecidedBy`).
- [ ] AC-2: `resolveAgentIdentityAlias` and `resolveFormerLoginMentions` in `ai/services/graph/agentFamilyResolution.mjs` resolve `@neo-fable`, `@neo-gpt`, `@neo-gemini-pro` (and earlier chain handles `@neo-gemini-3-1-pro`, `@neo-opus-4-7`, `@neo-claude-opus`) from `identityRoots.mjs` without touching `ai/graph/identityRoots.mjs` (preserving `@neo-gemini-maryam` non-rostered status per `#700`).
- [ ] AC-3: `MailboxService` (`add_message` / `list_messages`) and `SearchService.ask` (`ask_knowledge_base`) resolve `@neo-fable`, `@neo-gpt`, and `@neo-gemini-pro` to `@neo-fable-mnemosyne`, `@neo-gpt-euclid`, and `@neo-gemini-julia` and explicitly state the alias resolution in their return payload / answer.

## Out of Scope

- `github-content-sync` corpus refetch (`#1174` Leaf A).
- Per-repo markdown/workflow sweeps and seat Fleet-id renames (`#1173`).
- Modifying `ai/graph/identityRoots.mjs` (`#1188` / PR `#1189`).

## Avoided Traps

- Editing `ai/graph/identityRoots.mjs`: `formerLogins` is already present on all five renamed residents in `identityRoots.mjs` (`#1155`), and `@neo-gemini-maryam` must remain absent from `identityRoots.mjs` per `#700`.
- Relying on the LLM prompt alone in `SearchService.ask` to mention the handle rename: just like context-truncation notices (`SearchService.mjs:715–720`), the alias resolution notice must be attached deterministically on the service side of the provider call.

## Related

- Parent: `#1174` ("Old handles in artifacts resolve to the renamed identity instead of a dead login").
- Related: `#1147`, `#1149`, `#1155`, `#1173`, `#1188`.

Live latest-open sweep: checked latest 20 open Brain issues at 2026-10-19T13:38Z (newest `#1188`); parent `#1174` is open and explicitly invites a leaf ticket for the `formerLogins` + KB/A2A resolver leg (`MESSAGE:6583b47a-0916-4947-912a-92736cb7b034`). A2A in-flight claim sweep: checked latest 30 messages (`status: 'all'`) at 2026-10-19T13:38Z; no competing claim on `#1174` Leaf B. Memory Core rationale sweep: `query_raw_memories` on `formerLogins AgentIdentity node projection A2A mailbox ask_knowledge_base handle alias resolver` (`5` results, no conflicting decision). Own-assignment sweep: `0` open issues assigned to `@neo-gemini-maryam` in `neomjs/neo-agent-brain`. Structure map: `npm run ai:structure-map -- --files --loc` verified owning folders `ai/services/graph`, `ai/services/memory-core`, `ai/services/knowledge-base`, `ai/scripts/migrations` (no new `.mjs` files introduced).

Origin Session ID: 623823b7-3c27-4f59-a8e9-b04710554494

Retrieval Hint: "formerLogins nodeProjection resolveAgentIdentityAlias MailboxService SearchService ask_knowledge_base 1174"



## Timeline

- 2026-10-19T13:40:13Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-19T13:40:13Z @neo-gemini-maryam added the `ai` label
- 2026-10-19T13:40:13Z @neo-gemini-maryam added the `architecture` label
- 2026-10-19T13:40:14Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-19T13:40:14Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-19T13:40:41Z @neo-gemini-maryam added parent issue #1174
- 2026-10-19T14:03:15Z @neo-gemini-maryam cross-referenced by PR #1192
### @neo-gemini-maryam - 2026-10-19T14:03:33Z

## Contract Ledger (#1190)

| Surface | Path & Anchor | Contract |
| :--- | :--- | :--- |
| `PUBLIC_NODE_FIELDS` (`AgentIdentity`) | `ai/services/memory-core/nodeProjection.mjs` (`lines 32–37`) | Exposes `formerLogins` on `get_node(..., {projection: 'full'})` for `AgentIdentity` nodes while `lean` remains the six hoisted fields. |
| `getFormerLoginsForTarget` / `withFormerLogins` | `ai/scripts/migrations/renameAgentIdentities.mjs` (`lines 368–463`) | Preserves and seeds ordered `formerLogins` on merged and newly inserted canonical `AgentIdentity` rows during `runGraphMigration(db, true)`. |
| `FORMER_LOGIN_TO_IDENTITY` / `resolveAgentIdentityAlias` / `resolveFormerLoginMentions` / `parseSelfIdLogin` | `ai/services/graph/agentFamilyResolution.mjs` (`lines 70–169`, `292–304`) | Resolves former handles (`@neo-fable`, `@neo-gpt`, `@neo-gemini-pro`, etc.) to their current canonical `AgentIdentity` descriptor (`{id, githubLogin, displayName, formerLogin, formerLogins, notice}`), expands whole-handle mentions without matching sibling prefixes (`@neo-gpt-emmy`) or email addresses (`neo-fable@neomjs.com`), and resolves legacy `Authored by … @former-handle` lines. |
| `MailboxService.addMessage` / `MailboxService.listMessages` / `getMailboxIdentityStorageVariants` / `validateMailboxTarget` | `ai/services/memory-core/MailboxService.mjs` (`lines 262–280`, `309–327`, `2830–2834`, `3036–3071`, `3670–3680`, `3798–3808`) | Routes `addMessage({to: '<former-login>'})` to the canonical `AgentIdentity` with `resolvedRecipientAlias`, and expands `listMessages({to, fromIdentity})` across `formerLogins` storage variants with `resolvedIdentityAliases`. |
| `SearchService.ask` | `ai/services/knowledge-base/SearchService.mjs` (`lines 496–515`, `552–556`, `579–586`) | Expands `query` via `resolveFormerLoginMentions` before `QueryService.queryDocuments` and `enrichWithConceptWalk`, prepends `formerMentions.notice` to `answer`, and attaches `resolvedIdentityAliases`. |

— Maryam 🧭


- 2026-10-19T14:23:15Z @neo-gemini-maryam referenced in commit `39ccc19` - "fix(ai): restrict resolveFormerLoginMentions to @-prefixed handles (#1190)"
- 2026-10-19T15:21:58Z @tobiu referenced in commit `0d70f80` - "feat(ai): project formerLogins and resolve old handles in KB and A2A (#1190) (#1192)

* feat(ai): project formerLogins and resolve old handles in KB and A2A (#1190)

* fix(ai): restrict resolveFormerLoginMentions to @-prefixed handles (#1190)"
- 2026-10-19T15:22:23Z @tobiu closed this issue

