---
id: 387
title: Emit origin-qualified GitHub corpus without consumer derivation
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - architecture
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-09-19T19:50:15Z'
updatedAt: '2026-09-19T21:13:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/387'
author: neo-gpt
commentsCount: 0
parentIssue: 17416
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-19T21:13:33Z'
---
# Emit origin-qualified GitHub corpus without consumer derivation

## Context

[D#17846](https://github.com/neomjs/neo/discussions/17846) graduated its producer-only boundary: Brain retains generator code; a job in `neomjs/github-content-sync` uses a pinned Brain runtime and publishes only there. The destination now exists with `dev` as its default branch. The [scoped approval](https://github.com/neomjs/neo/discussions/17846#discussioncomment-18519041) separates this runtime from later consumer activation.

## The Problem

The existing CLI cannot supply that job. `syncGithubWorkflow.mjs --emit-only` calls `emitGeneratedContentAndDerive`: it avoids local-to-GitHub issue pushes but still emits release-note files and derives Portal artifacts. The per-type output paths and metadata file independently derive from the working directory. The declared corpus root must own relocatable children; the global index and origin-local buckets must not share an overloaded root.

The index has a second, independent blocker: `indexKey` uses `type:id` and normalization discards `repoSlug`. Two repositories' issue 1 collapse into one row. Factories, integrity scans and `PullRequestSyncer`'s numeric map must preserve the same identity; adding a directory prefix alone fixes none of them.

## The Architectural Reality

- `ai/services/github-workflow/SyncService.mjs` owns facet orchestration, release-history dependencies and per-facet metadata rollback.
- `ai/scripts/maintenance/syncGithubWorkflow.mjs` owns the CLI and maintenance-lease boundary.
- `ai/mcp/server/github-workflow/configBase.mjs` owns source identity and output-path declarations; ADR 0019 owns their resolution.
- `shared/contentIndex.mjs`, `contentPath.mjs`, `contentInventory.mjs` and `reconcileActiveChunks.mjs` own identity, placement and integrity; the three syncers construct entries and the pull syncer also reconstructs a numeric index.
- `LocalFileService` is an immediate lookup caller. Supplying its already-selected source origin is contract adaptation, not activation against the new corpus.
- Existing `ContentPath`, `contentInventory`, syncer, `SyncService.Stage2` and CLI specs are the proof homes. No new orchestration layer or chunking primitive is needed.

## The Fix

Deliver one coherent Brain runtime suitable for the corpus-owned job: explicit source origin and destination, a three-conversation-facet emission path, and one origin-qualified index/placement/integrity contract. Keep release-history retrieval as the bucketing prerequisite while excluding release-note materialization, Portal/SEO derivation, local issue pushes and git publication from this path.

Declarations belong to the config owner and must resolve before emission. Do not mutate the singleton to redirect an already-loaded service, re-read env at consumers, or infer a source repository from a content path. Keep runtime installation, source repository and destination checkout distinct. The invoking corpus job will own schedule, write credentials and publication in its own repository.

New active paths are <repoSlug>/<family>/chunk-N/... and new archived paths are <repoSlug>/archive/<family>/<version>/chunk-N/... beneath the shared corpus root. repoSlug is the bare repository name (neo, not neomjs/neo). Do not flatten the archive tier or move the historical source tree. Keep one canonical source repository declaration rather than adding an interchangeable repoSlug config alias.

Preserve existing explicit manual behavior where it does not conflict with the qualified-index contract; adapt immediate callers to a stated source origin. Consumer root switches, ingestion and deployment are separate deliveries.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Corpus CLI/runtime mode | D#17846 §8.9 D2/D3; existing CLI + SyncService | Emit issues, pulls and discussions into an explicitly admitted destination; retain release reference; no consumer derivation, local issue push or git publication | Missing/invalid source or destination refuses before content writes; failed facet gives non-success | CLI JSDoc and operator usage | Actual CLI process with controlled acquisition, destination sentinel and forbidden-effect controls |
| Config ownership | ADR 0019; GitHub Workflow ConfigBase | Declared source/destination resolve once through the Provider; every output and metadata path is coherent with that destination | No ambient runtime-checkout fallback for corpus mode | Leaf/derived-value JSDoc | Fresh-process configuration controls; ordinary mode control |
| Index identity | D#17846 §8.4a / §8.9 D1 | Factories, upsert, lookup, removal and serialization preserve (repoSlug,type,id); query shape validated before lookup | Unqualified query refuses distinctly from a valid miss; unknown origin is not guessed from path | Existing helper contracts | Two origins with equal ids survive write/read/update/removal independently |
| Placement and integrity | ADR 0004 amendment; contentPath/contentInventory | Ordinal-100 and complete membership per origin/bucket; integrity and syncer-local maps use the same origin scope | Wrong-origin/ambiguous corpus is reported, never collapsed or silently excluded | Helper and syncer JSDoc | Equal-id two-origin inventory/integrity + 100/101-item boundary controls |
| Bootstrap and progress | D#17846 §8.4a / §8.9 D2 | Legacy input may be read; every emitted row is qualified, including read-modify-write; progress belongs to its origin and successful work | Failed work retains its previous progress and no clean-run verdict | Runtime result/error contract | Legacy round trip, origin isolation and failure-injection arms |

## Decision Record impact

Depends on the ADR 0004 amendment in [Engine #18997](https://github.com/neomjs/neo/issues/18997); aligned with ADR 0019. The amendment is a merge-order gate, not a prerequisite to authorized implementation. Do not claim merge readiness until its authority is landed, included through the approved sequencing, or explicitly merge-ordered ahead.

**Decision Record: REQUIRED — ADR 0004 amendment.**

## Acceptance Criteria

- [ ] **AC-1** — The corpus-only path executes the existing three conversation syncers and release-history prerequisite, with no release-note files, Portal/SEO derivation, local-to-GitHub issue push or git publication.
- [ ] **AC-2** — Fresh-process source/destination configuration uses ADR-0019 declarations; missing/invalid corpus destination is refused before writes, and emitted content/index/metadata stay in the admitted tree.
- [ ] **AC-3** — Two origins' equal identifiers survive create, read, update and removal independently through the real factories and index; unqualified lookup refuses before matching, with a qualified absent-id control.
- [ ] **AC-4** — The same two-origin fixture exercises inventory, integrity, reconciliation and the pull syncer's local map. Ordinal-100 membership is per origin and bucket; the 100/101 boundary is preserved.
- [ ] **AC-5** — Legacy unqualified input is accepted only at the read/bootstrap boundary; every write path emits qualified rows using an explicit legacy ownership context, never an inferred/defaulted origin for arbitrary legacy data. A legacy read-modify-write fixture proves the distinction.
- [ ] **AC-6** — Metadata/progress is isolated by origin. Injected release-reference and facet failures retain the appropriate previous progress and produce a truthful non-success outcome.
- [ ] **AC-7** — An actual CLI process from a clean runtime installation exercises the producer into a disposable destination with controlled source responses, plus a positive content control and negative forbidden-write/effect controls. Existing non-corpus behavior is covered by its controls.
- [ ] **AC-8** — The delivered runtime contract and required caller inputs are documented so the corpus-owned publishing workflow can pin it. No scheduled publisher or consumer activation is claimed by this Brain PR.

## Out of Scope

The corpus repository's workflow, schedule, write credentials, actual Git publication and activation; consumer cutovers (KB, Portal/Pages, FM, LocalFileService root, IssueIngestor); historical archive relocation/dissolution; release-note relocation; Engine-targeted bridge producers; Graph ingestion or a second manifest/tenant vocabulary.

## Avoided Traps

- Redirecting only contentRoot leaves independently declared paths behind.
- A repo prefix on disk is not logical identity; every re-keying site must agree.
- A null result for an unqualified query misdiagnoses caller misuse as absent content.
- Read tolerance must not become permission to publish unqualified rows.
- An empty successful fixture is not evidence that the three facets ran.
- Do not remove release retrieval when excluding release-note files.
- Preserve the complete-membership and destination-path lessons from prior archive drift; do not restore delta-only ordinal planning.

## Signal Ledger

Producer boundary only: `claude` author adoption in D#17846 §8.4a/§8.9; `gpt` non-author [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/17846#discussioncomment-18519041) at body 2026-09-19T18:54:54Z, reconciled in the live body 19:29:11Z. The earlier scoped DEFERRED is discharged.

## Unresolved Dissent

None on this bounded runtime contract. Wider distribution, ingestion and consumer decisions remain outside it.

## Unresolved Liveness

No absent family is counted as consent. Revalidate this leaf if D#17846 §8.4a/§8.9 or the ADR amendment changes before merge.

## Discussion Criteria Mapping

B1 and B3 map to AC-3/4/5; B2 to AC-1/2/6; the Brain runtime portion of B4 to AC-7/8. The corpus-owned job retains its live activation proof. B5 is preserved by Out of Scope. B6 is the ADR merge-order obligation; B7 is recorded above.

## Related

Parent outcome: [Engine #17416](https://github.com/neomjs/neo/issues/17416).
Origin Session ID: 553fd0f7-80d4-4937-884a-7dfcad72e19a
Retrieval Hint: D17846 producer-only runtime, repo-qualified index, existing emit-only Portal/release-note coupling.

Ownership: Euclid is taking this runtime leaf. No claim on Vega's ADR or on the corpus repository's publisher.


Creation checks: live latest 20 open Brain issues and the corpus repository's empty issue queue checked immediately before filing (2026-09-19 19:50:09 UTC); no equivalent. All-read-state A2A claim sweep found no competing runtime owner; Vega owns the ADR. Closed Brain #289/#291 were rejected Engine-targeted bridges, whose runtime/destination separation is salvaged but whose publishing destination is not. Own #106/#107 bodies concern community-event/awareness authority, not this mirror runtime. Semantic memory 97ac103e-3e0c-4880-917f-ea1e9c6297b8 and 487748da-c259-48d7-a7fb-a1f704077035 recover the exact index and runner falsifiers. Structure-map executed; existing service/helper/maintenance homes own the change. Source surfaces are unchanged between aa6901c and current dev aea8f2a. No new file or directory is prescribed.

## Timeline

- 2026-09-19T19:50:15Z @neo-gpt assigned to @neo-gpt
- 2026-09-19T19:50:16Z @neo-gpt added the `enhancement` label
- 2026-09-19T19:50:16Z @neo-gpt added the `ai` label
- 2026-09-19T19:50:17Z @neo-gpt added the `testing` label
- 2026-09-19T19:50:17Z @neo-gpt added the `architecture` label
- 2026-09-19T19:50:17Z @neo-gpt added the `agent-os` label
- 2026-09-19T20:14:33Z @neo-gpt cross-referenced by PR #19002
- 2026-09-19T20:48:07Z @neo-opus-vega cross-referenced by #1
- 2026-09-19T20:52:03Z @neo-opus-vega cross-referenced by PR #2
- 2026-09-19T20:53:30Z @neo-gpt cross-referenced by PR #388
- 2026-09-19T20:56:01Z @neo-gpt referenced in commit `15576b8` - "test(github): prove per-origin corpus progress isolation (#387)

Co-Authored-By: Euclid <neo-gpt@neomjs.com>"
- 2026-09-19T21:13:33Z @tobiu referenced in commit `91bbf14` - "Merge pull request #388 from neomjs/codex/387-corpus-producer

feat(github): emit origin-qualified conversation corpus (#387)"
- 2026-09-19T21:13:33Z @tobiu closed this issue
- 2026-09-19T21:13:56Z @neo-opus-vega cross-referenced by #3
- 2026-09-19T21:48:43Z @neo-gpt cross-referenced by PR #4
- 2026-09-21T12:04:23Z @neo-opus-vega cross-referenced by #403

