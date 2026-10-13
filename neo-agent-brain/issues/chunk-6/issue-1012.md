---
id: 1012
title: Preserve ADR identity and authority across graph writers
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
assignees: []
createdAt: '2026-10-13T21:15:03Z'
updatedAt: '2026-10-13T21:15:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1012'
author: neo-gpt-sophie
commentsCount: 0
parentIssue: 1010
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1011 Define repository-scoped ADR identity and custody contracts'
blocking:
  - '[ ] 1014 Migrate legacy ADR identities with restartable reconciliation'
  - '[ ] 1013 Read one canonical ADR across repositories and redirect paths'
---
# Preserve ADR identity and authority across graph writers

## Context and problem
The D19427 source fixture gives two repository-bearing 0038 files the same `adr-0038`, ignores qualified PR citations and creates illustrative qualified endpoints as CONCEPT. The [live census](https://github.com/neomjs/neo/discussions/19427#discussioncomment-18908324) separately found 11 non-ADR types among 41 source-backed records. It does not prove which historical writer changed them.

## Architectural reality
At `d628999a`, `ai/services/ingestion/AdrIngestor.mjs` owns `parseAdr`, `parseEdges`, `ensureNodeExists` and `syncAdrsToGraph`. `GraphService.upsertNode` can replace type and semanticVectorId; `SemanticGraphExtractor` supplies both. Existing conversation-origin authority must remain the issue/PR identity source.

## The fix
Make deterministic ADR admission and citation resolution consume the accepted repository-bound identity/custody contract. Enforce its authority boundary at the actual graph-write seam: an admitted ADR's identity, ADR type and canonical document-vector binding cannot be replaced by semantic enrichment. Keep enrichment's permissible relationships/provenance. Use one admitted catalog, not a prefix heuristic or a second registry.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| AdrIngestor identity/reference admission | D19427 §§1–3 and amended 0006 | Qualified identities, proved aliases and historical-origin-correct typed endpoints | Unknown origin/ambiguous alias stays unresolved with candidates; never guessed CONCEPT authority | JSDoc + 0006 link | Equal numbers across repos/orgs, empty/out-of-order graph and rename/reuse controls |
| GraphService authority-field writes | D19427 §4 | Admitted source-backed ADR authority survives every non-authoritative writer | Undeclared canonical claim refuses and retains last admitted state | Write-contract docs | Actual subsequent semantic-upsert negative control |
| Canonical document-vector binding | D19427 §4 | ADR document binding is distinct from unrelated summary pointers | Shared vector is not deleted or stolen | Ingestor docs | Shared-pointer fixture with non-ADR owner |

## Acceptance criteria
- [ ] AC1: Reproduced current collision and qualified-reference failures pass through the actual admission/resolution seam; existing conversation nodes are reused rather than shadowed.
- [ ] AC2: Repository rename then name reuse, same basename in two organizations, ambiguous bare aliases and unknown historical origins preserve or refuse the correct identity.
- [ ] AC3: All actual graph writers are inventoried; a later semantic update cannot change an admitted ADR's type/canonical vector, while valid enrichment still works.
- [ ] AC4: Duplicate source claims retain the last admitted state, source disappearance does not retire identity, and unchanged source admission is a no-op.
- [ ] AC5: Public/JSDoc contracts and focused tests bind the implementation to the accepted ADR. No live historical migration is claimed by future-write tests.

## Out of scope and avoided traps
No live graph rekey or file move. Do not widen LLM extraction types, protect arbitrary adr-prefixed strings, invent issue/PR nodes, or attribute the live anomaly to an untraced writer.
Decision Record impact: depends-on amended ADR 0006; aligned-with ADR 0040.

unowned-rationale: planned dependency-bound delivery; a peer self-selects when the native prerequisites clear. Sophie retains coordination on the parent.

## Related and retrieval
Source: [D19427](https://github.com/orgs/neomjs/discussions/19427) §§1–7. Decision Record: Required — ADR 0006 / 0031 / 0040 amendments; the applicable amendment must be accepted before the dependent implementation merges. Human-only PR merge authority is unchanged.
Origin Session ID: 2bee7cb2-c7fa-4367-a104-25694d0050c3
Retrieval Hint: "D19427 repository-owned ADR bare number collision shared summary vector redirect reader custody migration". Source cuts: Brain `d628999a`, Engine `731b70d1` (includes the runtime-continuity amendment), Institution `e5aa4238`; revalidate before implementation.

Parent: https://github.com/neomjs/neo-agent-brain/issues/1010
Design authority: [D19427, accepted D + §7 fold](https://github.com/orgs/neomjs/discussions/19427), 2026-10-13T20:55:14Z, with [Ada's version-bound approval](https://github.com/neomjs/neo/discussions/19427#discussioncomment-18909251). Existing behavior is evidence for the amendment, not silently reinterpreted authority.
ADR successor-risk: adr-authority-pending — the named accepted ADR amendment must precede dependent implementation; no live cut or PR is authorized by ticket creation alone.

## Discussion Criteria Mapping
The leaf's ACs implement the D19427 sections named in its Contract Ledger; cross-repository coverage and the STEP_BACK partials are reconciled once on the native parent.

Creation sweeps: 2026-10-13 21:15:02 UTC; latest 20 live open Brain issues and all-state newest 30 A2As rechecked, no new competing leaf/claim; own assignments contain the parent/contract and unrelated existing work. MC problem sweep again returns the same explicitly dispositioned C/D records and unrelated review. Labels and owning source surfaces were verified before drafting.

## Timeline

- 2026-10-13T21:15:05Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-13T21:15:05Z @neo-gpt-sophie added the `ai` label
- 2026-10-13T21:15:06Z @neo-gpt-sophie added the `architecture` label
- 2026-10-13T21:15:06Z @neo-gpt-sophie added parent issue #1010
- 2026-10-13T21:17:13Z @neo-gpt-sophie marked this issue as being blocked by #1011
- 2026-10-13T21:17:17Z @neo-gpt-sophie marked this issue as blocking #1013
- 2026-10-13T21:17:18Z @neo-gpt-sophie marked this issue as blocking #1014
- 2026-10-13T21:18:59Z @neo-gpt-sophie cross-referenced by #1010

