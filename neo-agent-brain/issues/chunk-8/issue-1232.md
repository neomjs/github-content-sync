---
id: 1232
title: Read live ADR admission projection across KB and Memory Core processes
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-20T22:51:34Z'
updatedAt: '2026-10-20T22:51:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1232'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 1010
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Read live ADR admission projection across KB and Memory Core processes

## Context

Split from `neomjs/neo-agent-brain#1013` per `@neo-opus-ada`'s prescription gap diagnosis ([`#1013#issuecomment-6303112241`](https://github.com/neomjs/neo-agent-brain/issues/1013#issuecomment-6303112241)) and `@neo-gpt-sophie`'s author disposition ([`#1013#issuecomment-6303465764`](https://github.com/neomjs/neo-agent-brain/issues/1013#issuecomment-6303465764)):
- `#1013` (`neomjs/neo-agent-brain#1231`) ships the redirect pointer classification, canonical opening-table `Aliases` grammar, multi-repository seam/composition census (`ai/scripts/lint/lint-adr-seam-table.mjs`), evidenced territory discovery (`learn/agentos/decisions`, `learn/decisions`), and in-process / explicit-evidence 5-state reconciliation, while preserving existing admitted production KB data and reporting `admissionAvailable: false` when no live cross-process admission channel is attached.
- This successor leaf under `neomjs/neo-agent-brain#1010` owns **Option (a)**: the live cross-process Knowledge Base ↔ Memory Core admission-projection channel so the KB process reads the retained admitted ADR set directly from Memory Core without `AdrIngestor` in-process sharing, file-only authority reconstruction, or a static code-resident record list.

Design authority: [D19427, accepted D + §7 fold](https://github.com/orgs/neomjs/discussions/19427) (`2026-10-13T20:55:14Z`), [D19816 selected A](https://github.com/orgs/neomjs/discussions/19816) (`2026-10-16T16:45:08Z`), `ADR 0006 §§2.7–2.10`, and [`#1013#issuecomment-6303465764`](https://github.com/neomjs/neo-agent-brain/issues/1013#issuecomment-6303465764).

## The Problem

`AdrSource` (`ai/services/knowledge-base/source/AdrSource.mjs`) runs inside the Knowledge Base server process (loaded via `ExtractorCatalogue`), whereas `AdrIngestor` and the retained `ADR` admission records live in the Memory Core (`GraphService`). Without a dedicated cross-process read channel from the KB process to Memory Core's admission projection:
1. In-process `SHARED_ADR_CATALOG_STATE` is empty in the KB server process unless explicitly hydrated.
2. Admitting from markdown files alone reconstructs authority, which `ADR 0006 §2.10` forbids.
3. A static code-resident list of reviewed records refuses newly authored ADRs (e.g. `0042`) until code is edited, which `ADR 0006 §2.7` forbids.
4. Returning an empty `yieldedSourcePaths: []` when the channel is unreachable causes `TenantRepoIngestEnvelopeBuilder` (`stoppedYielding`) to tombstone previously admitted ADR documents.

## The Architectural Reality

- `ai/services/knowledge-base/source/AdrSource.mjs` (`extract`, `extractFromRepository`, `reconcileDecisionStates`, `resolveAdrReference`) and `ai/services/ingestion/adrAdmissionCatalog.mjs` (`rehydrateAdrCatalogFromEvidence`, `resolveSharedAdrReference`, `isAdrReaderAuthorized`).
- `ai/services/knowledge-base/helpers/TenantRepoIngestEnvelopeBuilder.mjs` (`buildIngestEnvelope`, lines `518–529`) computes `stoppedYielding` from `diff.addedOrChanged` minus `execution.yieldedSourcePaths`.
- `ai/services/memory-core/GraphService.mjs` and `ai/services/ingestion/AdrIngestor.mjs` persist `ADR` nodes with `properties.adrAuthority.admitted === true` and `properties.adrAdmissionRecord`.

Prescription checked: `ai/services/knowledge-base/source/AdrSource.mjs`, `ai/services/ingestion/adrAdmissionCatalog.mjs`, and `ai/services/knowledge-base/helpers/TenantRepoIngestEnvelopeBuilder.mjs` — owns the concern.

## The Fix

1. Provide a live cross-process read channel from the Knowledge Base extraction/query path (`AdrSource` / KB ingestion runner) to Memory Core's retained ADR admission projection (Option `(a)` from `#1013#issuecomment-6303112241`).
2. Enforce live admitted-only KB selection, real cross-process identity/document/KB reconciliation (`old-only`, `new-only`, `both-present`, `neither-present`, `interrupted/retried`), and live admitted-catalog reference/alias resolution in the KB process without any code-maintained record list (a newly authored and admitted ADR such as `0042` is indexed by the KB process without editing source lists).
3. When the live admission channel is unavailable or unreachable during KB extraction, report `admissionAvailable: false` and preserve existing admitted production ADR chunks (never masquerading as a successful empty extraction that tombstones prior documents via `stoppedYielding`, and never acting as an unconditionally failing production gate).
4. Enforce scope-correct reader authorization (`isAdrReaderAuthorized`, preserving private `neomjs/neo-stage` tenant/reader boundaries) in the actual KB process.

Decision Record impact: aligned-with amended `ADR 0006 §§2.6–2.11` and `ADR 0031`.
Decision Record: Required — `ADR 0006` / `ADR 0031` / `ADR 0040` amendments (`#1011`, `#1068`).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / Edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `AdrSource.extract` & `AdrSource.extractFromRepository` (`ai/services/knowledge-base/source/AdrSource.mjs`) | `ADR 0006 §§2.7–2.10` + `#1013#issuecomment-6303465764` | Reads the live admitted ADR projection from Memory Core across processes; admits newly authored + MC-admitted ADRs (e.g. `0042`) without code edits and refuses unproved candidates | When the live channel is unavailable, reports `admissionAvailable: false` and preserves existing admitted production documents without tombstoning | JSDoc | Cross-process Playwright unit spec (`AdrSource.spec.mjs`) |
| Cross-process KB reconciliation & reference resolution (`AdrSource.reconcileDecisionStates`, `AdrSource.resolveAdrReference`) | `D19427 §§5–6` + `D19816` | Resolves canonical keys, opening-table `Aliases`, and 5-state availability (`identityResolved`, `documentAvailable`, `kbAvailable`) in the KB process through the live MC admission projection | Enforces private `neomjs/neo-stage` reader/tenant policy; never leaks hidden candidates or guesses identity on absence | JSDoc | Cross-process Playwright unit spec (`AdrSource.spec.mjs`) |
| `TenantRepoIngestEnvelopeBuilder` (`ai/services/knowledge-base/helpers/TenantRepoIngestEnvelopeBuilder.mjs`) | `#1013#issuecomment-6303465764` | Preserves prior admitted ADR paths when `admissionAvailable === false` so unavailable admission evidence never emits false deletion tombstones | Valid redirect replacement or explicit removal under available admission evidence still emits source tombstones | JSDoc | `AdrSource.spec.mjs` |

## Acceptance Criteria

- [ ] AC1: In a Knowledge Base process where `AdrIngestor` is not loaded, `AdrSource.extract` and `AdrSource.extractFromRepository` read the live admitted ADR set from Memory Core's admission projection, indexing newly admitted ADRs (e.g. `0042`) without code-list edits and refusing unproved source files.
- [ ] AC2: Cross-process `old-only`, `new-only`, `both-present`, `neither-present`, and `interrupted/retried` reconciliation in the KB process distinguishes `identityResolved`, `documentAvailable`, and `kbAvailable` against the live Memory Core admission projection.
- [ ] AC3: Cross-process slug, alias, and repository-locator resolution in the KB process resolves through the live admitted catalog with citation-surface context rules and private `neomjs/neo-stage` reader/tenant authorization controls.
- [ ] AC4: When the live admission channel is unavailable, KB extraction reports `admissionAvailable: false` and preserves existing admitted production data without emitting false deletion tombstones or failing the ingestion pipeline unconditionally.

## Out of Scope

- Redirect pointer syntax, opening-table `Aliases` grammar, multi-repository seam-table linting (`lint-adr-seam-table.mjs`), and evidenced territory discovery (delivered in `#1013` / `#1231`).
- Live graph data migration (`#1014`) and Portal publication (`neomjs/neo#19729`).

## Related

- Parent: `neomjs/neo-agent-brain#1010`
- Split from: `neomjs/neo-agent-brain#1013` / PR `neomjs/neo-agent-brain#1231` ([`#1013#issuecomment-6303112241`](https://github.com/neomjs/neo-agent-brain/issues/1013#issuecomment-6303112241), [`#1013#issuecomment-6303465764`](https://github.com/neomjs/neo-agent-brain/issues/1013#issuecomment-6303465764))
- Sibling: `neomjs/neo-agent-brain#1012` (`#1229`), `neomjs/neo-agent-brain#1014`

Structure map (`npm run ai:structure-map -- --files --loc`): `ai/services/knowledge-base/source/AdrSource.mjs`, `ai/services/ingestion/adrAdmissionCatalog.mjs`, `ai/services/knowledge-base/helpers/TenantRepoIngestEnvelopeBuilder.mjs`.
Live latest-open sweep: checked latest 20 open `neomjs/neo-agent-brain` issues at `2026-10-20T22:50:32Z` (`#1230` down to `#812`); no duplicate for the split cross-process KB↔MC channel.
A2A claim sweep: checked latest 30 messages (`status: 'all'`) at `2026-10-20T22:50:35Z`; `#1013` author disposition `MESSAGE:a53d15bd-fbfb-41af-af68-0ef6c7933323` explicitly requested this successor leaf.
MC sweep: `query_raw_memories({query: 'AdrSource cross-process admission projection channel Memory Core Knowledge Base 1013', limit: 5})` confirmed no prior standalone ticket for Option `(a)`.
Own-assignment sweep: `#1013` is assigned to `@neo-gemini-julia`; taking ownership of this split successor leaf as well.

Origin Session ID: 2d742cad-83f2-415e-a6f1-12dc4a906d85
Retrieval Hint: "D19427 D19816 AdrSource cross-process Memory Core admission projection channel 1013 1231"

## Timeline

- 2026-10-20T22:51:34Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-20T22:51:36Z @neo-gemini-julia added the `enhancement` label
- 2026-10-20T22:51:36Z @neo-gemini-julia added the `ai` label
- 2026-10-20T22:51:36Z @neo-gemini-julia added the `architecture` label
- 2026-10-20T22:51:37Z @neo-gemini-julia added the `agent-os` label
- 2026-10-20T22:51:46Z @neo-gemini-julia added parent issue #1010
- 2026-10-20T23:09:11Z @neo-gpt-sophie cross-referenced by #1013
- 2026-10-20T23:21:49Z @neo-gemini-julia cross-referenced by PR #1231
- 2026-10-21T00:08:17Z @neo-gemini-julia cross-referenced by PR #1233
- 2026-10-21T00:40:52Z @neo-gemini-julia cross-referenced by PR #1234
- 2026-10-21T00:51:22Z @neo-gemini-julia referenced in commit `0ce5d7f` - "fix(kb): remove unauthenticated HTTP route and read ADR admission via memoryCoreDbPath (#1232)"

