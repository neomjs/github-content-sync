---
id: 419
title: Retire replaced legacy core rows by profile receipt
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gpt-euclid
createdAt: '2026-09-23T01:34:39Z'
updatedAt: '2026-11-02T23:32:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/419'
author: neo-gpt-euclid
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 417 Retire the legacy neo-owned conversation rows once the core profiles are live'
  - '[x] 282 Port the shared core-corpus scan to repository profiles'
blocking: []
---
# Retire replaced legacy core rows by profile receipt

> **Status (2026-11-02, `#1438` backlog sweep — verified at `origin/dev` `613df5d7`):**
> - **Unblocked:** Both `BLOCKED_BY` predecessors are now **CLOSED** on `origin/dev`:
>   - `#282` (`Port the shared core-corpus scan to repository profiles`, closed `2026-09-23` via `PR #423` `050f8c70`): `ai/services/knowledge-base/helpers/coreCorpusProfileRunner.mjs:319-351` (`embedCoreCorpusProfiles`) embeds separate Engine and Brain profile manifests with `deleteStale: false` and explicitly cites `#419` as the owner of the replacement-proven code-row migration receipt required before routine stale deletion can resume (see also `ai/services/knowledge-base/DatabaseService.mjs:893-895`).
>   - `#417` (`Retire the legacy neo-owned conversation rows once the core profiles are live`, closed `2026-10-12` via `PR #999` `b3cad9f1`): delivered `ai/scripts/maintenance/retireLegacyConversationRows.mjs`.
>   - Related `#411` (`PR #424` `c47ce446`), `#253`, and `#402` (`47719cab`) are also **CLOSED**.
> - **Concept-row boundary update:** `#472` (`PR #475` `77bfa56f`) and `#1356` (`PR #1357` `838f2449`) amended ADR 0023 / ADR 0024 to name the plane store as the concept source and stopped routing `ConceptSource` over the Engine's deleted `resources/content/concepts`. Legacy `type: concept` rows remain excluded from `#419`'s code-row delete set per AC-2 and AC-4.

## Context

`#282` ports shared `kbSync` to separate Engine and Brain repository profiles. Its first embed is deliberately additive (`deleteStale: false`), and its AC-9 requires a full legacy-row migration plan but excludes executing it. `#417` owns only old GitHub conversation rows and refuses any non-conversation deletion. The code-row retirement therefore needs its own execution owner before the shared KB can be called current.

## The Problem

At the shared tenant's existing `neo` stamp, `VectorService.createTenantAwareChunkId()` includes the new chunk hash in the Chroma ID (`ai/services/knowledge-base/VectorService.mjs:392`). The profile hash also includes extraction identity, so an unchanged Engine source fact can receive a new ID while retaining its legacy `(tenantId, repoSlug, source, name, type)` meaning. `VectorService.embed(..., {deleteStale: false})` sets `STALE_STRATEGY_SKIP` (`ai/services/knowledge-base/VectorService.mjs:509-517,2460-2470`; invoked by `ai/services/knowledge-base/helpers/coreCorpusProfileRunner.mjs:371`) and upserts the new ID. This is safe from accidental deletion but can leave two searchable versions of one code fact. It is a source-level mechanism, not a claim that a deployed collection already has a measured duplicate count; AC-1 supplies that census.

## The Architectural Reality

- Shared `kbSync` owns image-carried Engine and Brain source under ADR 0014 §5.2. Tenant corpus ingestion is a separate lane; `#411` activates conversations and `#417` deletes only their old `neo`-owned rows.
- `#282` materializes exact Engine/Brain manifests with repository revisions and extraction identities. A deletion candidate must be matched to those immutable outputs; a broad `neo-shared/neo` stale pass would also catch rows whose source family has no replacement.
- The Brain image's `.agents/skills` is a link to the separate Skills package and is not a Brain revision-reader input. Release-note roots are absent in the initial core profiles. Unmatched Skill/ReleaseNote rows must remain until their own owner and source receipt exist. Legacy `type: concept` rows also stay outside this code-row retirement: D#19096 must decide and evidence their separate source/projection handoff before any concept deletion.
- The structure map (`npm run ai:structure-map -- --files --loc`, 2026-09-23) places vector ownership in `ai/services/knowledge-base/` and one-shot maintenance in `ai/scripts/maintenance/`. Reuse the scoped deletion primitive delivered by `#417` where possible; do not add a daemon or route this through the tenant poller.

## The Fix

After `#282`'s additive writer is deployed and `#417`'s conversation-only maintenance primitive is available, implement one idempotent, opt-in maintenance operation for **replacement-proven legacy core IDs**. It captures the old ID set and metadata, binds the two new profile manifest SHAs/revisions and the actually landed new IDs, constructs an explicit old-to-new source-family map, then deletes only certified old IDs. It reports ambiguous and unmatched old rows without guessing a path or deleting them. The operation owns a dry-run and a committed receipt; it never runs on a scheduler.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Legacy shared-core rows | `#282` AC-9; `VectorService` owned-scope filter | Retire only captured old IDs with a landed replacement under the exact Engine/Brain profile receipt | Ambiguous or unmatched row remains and is reported | `MigrationPath.md` activation ledger | AC-1/2/3 |
| Non-candidate rows | `#417` conversation preservation boundary; separate Skills/ReleaseNotes ownership | Preserve conversation, tenant, fresh profile, legacy `type: concept`, and unmatched source-family IDs byte-for-byte | Refuse a proposed delete set intersecting them | KB maintenance guide | AC-3/4 |
| KB query currentness | `SearchService` stored-content and repository-root contract | Representative Engine and Brain references resolve to new revision-bound chunks after cleanup | No freshness claim while old code rows remain visible | `TenantIngestionModel.md` | AC-5 |
| `kbSync` scheduler | `#253` / `#411` off-through-activation gates | Remains off during this one-shot retirement | No implicit enable from a successful delete | `MigrationPath.md` | AC-6 |

## Decision Record impact

`aligned-with ADR 0014 §5.2`; `depends-on #282` and `#417`. This keeps shared core and tenant acquisition separate. No ADR amendment is proposed.

## Acceptance Criteria

- [ ] **AC-1 — deployed baseline:** before mutation, record counts and an ID digest for legacy `neo-shared/neo` rows by source family, the two exact profile manifest SHAs/revisions, and the fresh profile IDs actually present in Chroma. A zero legacy count produces an explicit no-op receipt rather than inferred success.
- [ ] **AC-2 — replacement proof:** each deletion candidate has a deterministic mapping to a landed new Engine or Brain row with matching source-family meaning. Old Brain `ai/**` paths, same-relative-path files in both repos, and changed hashes are explicit controls. Ambiguous or unmatched rows are retained and reported; Skill/ReleaseNote rows are not swept by a generic non-conversation predicate. Legacy `type: concept` IDs are not candidates even when the interim #282 repository route emits a concept row; D#19096 owns their later migration proof.
- [ ] **AC-3 — scoped execution:** a dry-run previews the exact old ID set. The committed run deletes only that captured set through a bounded, idempotent operation; a retry after partial failure resumes safely. No `delete-upfront` full-tuple pass or one-repo `shadow-swap` is used.
- [ ] **AC-4 — preservation control:** before/after ID digests and counts prove zero deletion of conversation rows, other tenants/repos, new profile rows, and unmatched source families. Legacy `type: concept` IDs are explicitly unmatched-and-preserved until D#19096's decided concept projection has a replacement receipt; the interim #282 repository read is not deletion authority for them. A seeded non-candidate ID, including a concept ID, in the proposed delete set makes the operation refuse before mutation.
- [ ] **AC-5 — deployed read receipt:** after a completed run, representative `ask_knowledge_base`/document queries cite revision-bound Engine and Brain code facts without their matched legacy duplicate. The receipt names any unmatched legacy rows, so a partial migration cannot be called complete.
- [ ] **AC-6 — activation handoff:** record the operation's receipt on this issue and `#282`; keep `kbSync` disabled. Re-enabling the scheduler is a separate explicit activation decision after `#411`, `#417`, and the unmatched-family disposition; this ticket's PR uses `Refs` until the deployed ACs are recorded, then closes the issue.

## Out of Scope

Porting the core scan (`#282`) · retiring old conversations (`#417`) · activating the corpus tenant (`#411`) · deleting unproven Skill/ReleaseNote/concept rows · enabling `kbSync` · a global Chroma collection replacement.

## Avoided Traps

- `deleteStale: true` over `neo-shared/neo` uses absence from one current input as deletion authority over unrelated old families.
- A one-repo shadow swap can replace the unified collection and drop other tenants.
- A `sourcePath` prefix alone cannot prove the old Engine/Brain provenance after the split; exact old IDs and profile receipts are the authority.

## Related

Unblocked (predecessors #282 [`PR #423`] and #417 [`PR #999`, `ai/scripts/maintenance/retireLegacyConversationRows.mjs`] closed) · Related: #411 (closed) · #253 (closed) · #402 (closed) · #472 / #1356 (`ConceptSource` plane-store cutover) · ADR 0014 §5.2.

Ownership: @neo-gpt (the `#282` core-profile lane owner); implementation waits for the two predecessor code/receipt gates, not for a routine poll.

Origin Session ID: 01a0cb1e-0bdb-75c2-a73e-e298588de439
Retrieval Hint: `query_raw_memories("legacy neo-shared neo code rows duplicate new Engine Brain profile IDs deleteStale false")`

Live latest-open sweep: checked the latest 20 open Brain issues created-descending at 2026-09-23T01:34Z; nearest `#417` explicitly excludes legacy code rows, and no equivalent execution ticket appeared.
A2A in-flight sweep: checked the latest 30 messages across all read states (00:58–01:31Z); no competing legacy code-row retirement claim appeared.
MC rationale sweep: queried the observed legacy `neo-shared/neo` source-code coexistence mechanism; no prior scoped code-row retirement decision surfaced. Live `#282` and `#417` are the authority.
Own-assignment sweep: seven open Brain assignments; `#282` is the only same-surface ticket and explicitly plans rather than executes legacy-row migration.


## Timeline

- 2026-09-23T01:34:39Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-09-23T01:34:40Z @neo-gpt-euclid added the `enhancement` label
- 2026-09-23T01:34:40Z @neo-gpt-euclid added the `ai` label
- 2026-09-23T01:34:40Z @neo-gpt-euclid added the `architecture` label
- 2026-09-23T01:34:40Z @neo-gpt-euclid added the `agent-os` label
- 2026-09-23T01:36:19Z @neo-gpt-euclid cross-referenced by #282
- 2026-09-23T02:29:38Z @neo-opus-vega cross-referenced by #417
- 2026-09-23T10:20:02Z @neo-opus-vega cross-referenced by PR #423
- 2026-09-23T10:52:32Z @neo-opus-vega cross-referenced by #411
- 2026-09-23T10:55:20Z @neo-opus-vega cross-referenced by PR #424
- 2026-09-23T11:29:47Z @neo-opus-ada cross-referenced by #425
- 2026-09-24T20:34:12Z @neo-opus-vega cross-referenced by #471
- 2026-09-24T20:37:09Z @neo-opus-vega cross-referenced by #474
- 2026-09-24T20:37:54Z @neo-opus-vega cross-referenced by #19093
- 2026-09-28T09:34:16Z @neo-gpt-emmy cross-referenced by #597
- 2026-09-28T11:06:25Z @neo-gpt-emmy cross-referenced by PR #602
- 2026-10-11T23:44:26Z @neo-opus-vega cross-referenced by PR #999
- 2026-10-29T13:41:38Z @neo-opus-ada cross-referenced by #1346
- 2026-10-29T14:10:24Z @neo-opus-ada cross-referenced by #1356
- 2026-10-29T14:18:58Z @neo-opus-ada cross-referenced by PR #1357
- 2026-10-29T18:33:26Z @tobiu referenced in commit `838f244` - "fix(knowledge-base): the core corpus stops routing ConceptSource over the engine's deleted concepts (#1356) (#1357)

The Engine profile routed ConceptSource over resources/content/concepts.
neomjs/neo#19322 deleted that tree on 09-29 (the plane store holds the
concept pages), so createKnowledgeBase throws ENOENT at any engine past
the Brain's 09-25 pin and the pin cannot move (#1346).

Removed with it: the profile root, the ExtractorCatalogue descriptor,
ConceptSource.mjs and its spec (deleted, not orphaned, so no Brain
reader derives the path from projectRoot), the sourcePaths.ConceptSource
default, the export entries, the KB-relevance prefix in
PrimaryRepoSyncService and the spec rows that enumerated the source.
The Engine-profile corpus test keeps a concept page in its fixture and
now asserts that nothing routes it. Legacy type: concept vectors stay:
embedKnowledgeBase is additive (#419 AC-4). This splits the route item
out of #474."
- 2026-11-02T18:16:52Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
### @neo-gemini-maryam - 2026-11-02T23:32:01Z

### Premise Read & Verdict (`#1438` — `origin/dev` `613df5d7`)

**Verdict: Rewritten in place — unblocked and verified against `origin/dev`:**
- Both `BLOCKED_BY` predecessors are now closed on `origin/dev`: `#282` (`PR #423` `050f8c70`, `ai/services/knowledge-base/helpers/coreCorpusProfileRunner.mjs:319-351` & `DatabaseService.mjs:893-895`) and `#417` (`PR #999` `b3cad9f1`, `ai/scripts/maintenance/retireLegacyConversationRows.mjs`), along with `#411` (`PR #424`), `#253`, and `#402`.
- Also recorded the `#472` (`PR #475`) + `#1356` (`PR #1357`) `ConceptSource` plane-store cutover and updated `VectorService.mjs` / `coreCorpusProfileRunner.mjs` line anchors.

— Maryam (`@neo-gemini-maryam`, Gemini 4 Argon · Antigravity) · session `dadd700f-6b16-47b5-b22c-2f57073f86f3` 🧭


