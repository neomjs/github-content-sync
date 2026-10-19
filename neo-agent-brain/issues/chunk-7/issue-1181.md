---
id: 1181
title: A redirect at an old ADR path is never read as an ADR
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-19T11:36:35Z'
updatedAt: '2026-10-19T12:32:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1181'
author: neo-opus-grace
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
closedAt: '2026-10-19T12:32:51Z'
---
# A redirect at an old ADR path is never read as an ADR

## Context

Operator direction, 2026-10-19:
- The five Fleet Manager ADRs leave the Engine for Institution before Engine v13.3 is cut.
- Every repository keeps its decision records at one path, `learn/agentos/decisions`. Stage's single record moves there under its own neo-stage leaf.

ADR 0040 §2.7.1 already names Institution's `learn/agentos/decisions/` as the FM destination and says the old Engine paths become classified pointers. ADR 0006 §2.9 defines that pointer (`type: redirect` frontmatter). It also requires every reader to classify a document *before* applying the numbered-filename heuristic, and to do so before the first move.

## The Problem

Both KB extraction paths and the graph's ADR ingestor select a record by its filename alone (`^\d{4}-.*\.md$`). A pointer left at `learn/agentos/decisions/0034-electron-shell-architecture.md` would therefore be indexed as an ADR chunk: a second, contentless authority for a decision that now lives in Institution. D19427's fold named this exact risk (Clio's condition 3; Sophie's 10-06 exploration: "flat numbered redirects are still selected by readers").

## The Architectural Reality

- `ai/services/knowledge-base/source/AdrSource.mjs` has two paths. `extract()` reads the local root (`aiConfig.neoRootDir` + `sourcePaths.AdrSource`). `extractFromRepository()` reads one assigned repository territory and reports skipped entries with a `reason`. Both match on the filename only.
- `ai/services/ingestion/AdrIngestor.mjs` `syncAdrsToGraph()` scans one directory. Its only caller, `src/evolution/RemDigestion.mjs`, passes the default `sourceRoot`, which is the Brain checkout.
- The KB's Engine corpus is the Brain image's `node_modules/neo.mjs` (`helpers/imageRevisionReader.mjs`), pinned in `package.json` to an Engine commit. Engine pointers therefore reach the KB only when that pin moves past the Engine cut. ADR 0006 §2.10 still orders this reader before the move.
- ADR 0006 §2.11 lists two discovery territories (`learn/agentos/decisions` and `learn/decisions`). The operator's direction leaves one.

## The Fix

1. One shared predicate: a markdown document whose frontmatter `type` is exactly `redirect` is a pointer (ADR 0006 §2.9). Callers apply it before the numbered-filename heuristic. Placement follows the structural pre-flight against the two callers' sibling helpers.
2. `AdrSource.extract()` skips a pointer. `extractFromRepository()` reports it as skipped, the way it reports any non-ADR file.
3. `AdrIngestor.syncAdrsToGraph()` skips a pointer: no ADR node, no ADR edges.
4. ADR 0006 §2.11: discovery reads the one territory `learn/agentos/decisions`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `AdrSource.extract` / `extractFromRepository` | ADR 0006 §2.9 | A `type: redirect` document yields no ADR chunk; the repository path reports it as skipped | No frontmatter, or another `type` → unchanged: indexed as an ADR | JSDoc | Spec arms on both paths: pointer skipped, numbered ADR beside it kept |
| `AdrIngestor.syncAdrsToGraph` | ADR 0006 §2.9 | A pointer produces no ADR node or ADR edge | Same as above | JSDoc | Spec arm |
| ADR 0006 §2.11 territory line | Operator direction 2026-10-19 | One discovery territory, `learn/agentos/decisions` | Stage's record moves in its own leaf | ADR text | — |

## Acceptance Criteria

- [ ] AC1: Both KB extraction paths skip a `type: redirect` document at a numbered path and still index the numbered ADR beside it, red-first against `dev`.
- [ ] AC2: The graph ingestor produces no ADR node for a pointer.
- [ ] AC3: ADR 0006 §2.11 names one discovery territory, `learn/agentos/decisions`.
- [ ] AC4: ADR 0040 §2.7.1 records the FM cut's order and why it is safe ahead of custody admission. The order is readers (this ticket), then publication (neomjs/neo-agent-institution#686), then Engine pointers (neomjs/neo#19730). Admission and guidance follow. The reader outcome is pointers. (Added 2026-10-19 11:5xZ: the accepted text put admission and guidance first, and the operator made the move a v13.3 blocker.)

## Out of Scope

- The admitted catalog and canonical-only indexing (#1012 and the rest of #1013 AC1).
- The alias and slug reader (#1013 AC6–7) and census/composition (#1013 AC3).
- Migration (#1014) and authoring guidance (neomjs/neo-agent-skills#174).
- Institution's records reaching the KB (a corpus-membership decision; it must land before Brain's `neo.mjs` pin moves past the Engine cut).
- Moving any file.

## Decision Record impact

Amends ADR 0006 §2.11 (the territory line) and ADR 0040 §2.7.1 (the FM cut's order and its pointer reader outcome). Implements ADR 0006 §2.9's reader classification for both KB paths and the graph ingestor.

## Related

Parent: #1010 · delivers the redirect-classification half of #1013 AC1 · neomjs/neo-agent-institution#686 · neomjs/neo#19729 · neomjs/neo#19730 · neomjs/neo-agent-skills#174

Creation sweeps, 2026-10-19 11:3xZ:
- Live latest 20 open Brain issues: no equivalent. #1013 is the scope this carves from.
- Newest 30 A2As, all states: no claim on ADR readers or paths.
- MC rationale: D19427 condition 3 and Ada's STEP_BACK consumer partial agree.
- Own assignments: #1142, #684, #54, all unrelated.
- `npm run ai:structure-map -- --files --loc`: exit 0; owning files exist, no new folder.

Origin Session ID: 728205e6-cebc-4b9c-9eb3-2bc53de75c2b
Retrieval Hint: "ADR redirect pointer classified before the numbered filename heuristic; FM ADR custody cut before Engine v13.3; one decisions path learn/agentos/decisions"


## Timeline

- 2026-10-19T11:36:35Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-19T11:36:36Z @neo-opus-grace added the `enhancement` label
- 2026-10-19T11:36:36Z @neo-opus-grace added the `ai` label
- 2026-10-19T11:36:36Z @neo-opus-grace added the `architecture` label
- 2026-10-19T11:36:36Z @neo-opus-grace added the `agent-os` label
- 2026-10-19T11:36:51Z @neo-opus-grace added parent issue #1010
- 2026-10-19T11:46:19Z @neo-opus-grace cross-referenced by PR #1182
- 2026-10-19T11:52:58Z @neo-opus-grace referenced in commit `9b62cf9` - "docs(adr): ADR 0040 orders the FM record move ahead of custody admission (#1181)"
- 2026-10-19T12:09:34Z @neo-opus-grace referenced in commit `6ac96e7` - "docs(adr): ADR 0040 separates no new admission from existing legacy graph state (#1181)"
- 2026-10-19T12:32:52Z @tobiu closed this issue
- 2026-10-19T12:32:52Z @tobiu referenced in commit `455e892` - "feat(adr): readers skip a redirect left at a moved record's old path (#1181) (#1182)

* feat(adr): readers skip a redirect left at a moved record's old path (#1181)

A record that moves to another repository leaves a pointer at its old
path (ADR 0006 §2.9: frontmatter type: redirect). Both KB extraction
paths and the graph ingestor selected records by the numbered filename
alone, so that pointer would have been indexed as a second record of
the decision it points away from. One shared reader,
ai/services/shared/adrRedirect.mjs, now classifies a document before
the filename heuristic: AdrSource skips it (the repository path reports
it as skipped, reason redirect) and AdrIngestor creates no node, edge or
vector, nor counts it in adrsProcessed.

ADR 0006 §2.11 names one discovery territory, learn/agentos/decisions,
per the operator's one-path direction for every repository.

* docs(adr): ADR 0040 orders the FM record move ahead of custody admission (#1181)

* docs(adr): ADR 0040 separates no new admission from existing legacy graph state (#1181)"

