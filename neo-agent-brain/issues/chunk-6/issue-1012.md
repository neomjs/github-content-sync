---
id: 1012
title: Preserve ADR identity and authority across graph writers
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gemini-julia
createdAt: '2026-10-13T21:15:03Z'
updatedAt: '2026-10-20T17:01:26Z'
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
  - '[x] 1068 Define readable ADR slug references and retained alias history'
  - '[x] 1011 Define repository-scoped ADR identity and custody contracts'
blocking:
  - '[ ] 1014 Migrate legacy ADR identities with restartable reconciliation'
  - '[ ] 1013 Read one canonical ADR across repositories and redirect paths'
closedAt: '2026-10-20T17:01:26Z'
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

- [ ] AC6 (D19816): Resolve all three selected slug forms using evidenced citation context and reader authorization. Org Discussion hosting repositories and MC/A2A seats/cwd/related-ticket lists supply no inferred repository context. Canonical endpoint typing derives from admitted authority, never prefix alone.
- [ ] AC7 (D19816): Normal create/rename admission maintains the identity-bound record specified in the amended ADR: assignment evidence, pinned source tuple, historically scoped alias bindings, source-repository reservations and recovery provenance. Authors need no second registration/manifest edit.
- [ ] AC8 (D19816): Removing an admitted spelling refuses and retains its reservation; two competing new claims in one batch admit neither. Rename/move/combined-move controls preserve A's historical scopes and do not fabricate B's old alias. Loss of source/evidence has explicit refusal/stale behavior.
- [ ] AC9 (D19816): Measure normal author actions, including alias-row/legacy-table edits. Present files plus unfiltered history may expose a candidate floor; only admission's proved population certifies eligibility. Incomplete history is explicit.

## Out of scope and avoided traps
No live graph rekey or file move. Do not widen LLM extraction types, protect arbitrary adr-prefixed strings, invent issue/PR nodes, or attribute the live anomaly to an untraced writer.
Decision Record impact: depends-on amended ADR 0006; aligned-with ADR 0040.

unowned-rationale: planned dependency-bound delivery; a peer self-selects when the native prerequisites clear. Sophie retains coordination on the parent.

## Related and retrieval
Source: [D19427](https://github.com/orgs/neomjs/discussions/19427) §§1–7. Decision Record: Required — ADR 0006 / 0031 / 0040 amendments; the applicable amendment must be accepted before the dependent implementation merges. Human-only PR merge authority is unchanged.
Origin Session ID: 2bee7cb2-c7fa-4367-a104-25694d0050c3
Retrieval Hint: "D19427 repository-owned ADR bare number collision shared summary vector redirect reader custody migration". Source cuts: Brain `d628999a`, Engine `731b70d1` (includes the runtime-continuity amendment), Institution `e5aa4238`; revalidate before implementation.

Parent: https://github.com/neomjs/neo-agent-brain/issues/1010
Additional design authority: [D19816 selected A](https://github.com/orgs/neomjs/discussions/19816), body 2026-10-16T16:45:08Z, signals below; amendment prerequisite https://github.com/neomjs/neo-agent-brain/issues/1068.

Design authority: [D19427, accepted D + §7 fold](https://github.com/orgs/neomjs/discussions/19427), 2026-10-13T20:55:14Z, with [Ada's version-bound approval](https://github.com/neomjs/neo/discussions/19427#discussioncomment-18909251). Existing behavior is evidence for the amendment, not silently reinterpreted authority.
ADR successor-risk: adr-authority-pending — the named accepted ADR amendment must precede dependent implementation; no live cut or PR is authorized by ticket creation alone.

## Signal Ledger
D19816 body `2026-10-16T16:45:08Z`: `gpt` [Sophie AUTHOR_SIGNAL](https://github.com/neomjs/neo/discussions/19816#discussioncomment-18978088); `gemini` [Julia APPROVED](https://github.com/neomjs/neo/discussions/19816#discussioncomment-18978119); `claude` [Ada APPROVED](https://github.com/neomjs/neo/discussions/19816#discussioncomment-18978148). [Corrected STEP_BACK](https://github.com/neomjs/neo/discussions/19816#discussioncomment-18977735) acknowledges all eight points.

## Unresolved Dissent
None at that bound fold. Retained evidence loss refuses reconstruction; authoring history remains advisory to admission-owned eligibility.

## Unresolved Liveness
All three signing families were active in the 2026-10-16T16:46:16Z AgentIdentity read. Kimi identities are operator_benched; no absent signal is counted as consent. No Tier-2 gate/value change.

## Discussion Criteria Mapping
The original ACs implement D19427 as mapped in the Contract Ledger. The D19816 ACs above implement the selected naming/carrier contract and the consumer acknowledgment in its body at 2026-10-16T16:45:08Z. Decision Record: REQUIRED — Brain #1068's ADR0006 amendment must merge before this leaf adopts that delta. The same parent #1010 retains integrated acceptance.

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
- 2026-10-16T16:50:41Z @neo-gpt-sophie cross-referenced by #1068
- 2026-10-16T16:51:18Z @neo-gpt-sophie marked this issue as being blocked by #1068
- 2026-10-19T00:45:00Z @neo-opus-grace cross-referenced by #686
- 2026-10-19T11:36:36Z @neo-opus-grace cross-referenced by #1181
- 2026-10-19T11:46:19Z @neo-opus-grace cross-referenced by PR #1182
- 2026-10-20T14:47:53Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-20T15:28:53Z @neo-gemini-julia cross-referenced by PR #1229
- 2026-10-20T16:23:52Z @neo-gemini-julia referenced in commit `ae2a7b4` - "fix(adr): rehydrate catalog from graph, require ADR type at write seam, and extract codec/parsers (#1012)"
- 2026-10-20T17:01:26Z @tobiu closed this issue
- 2026-10-20T17:01:27Z @tobiu referenced in commit `9668697` - "feat(adr): preserve ADR identity and authority across graph writers (#1012) (#1229)

* feat(adr): preserve ADR identity and authority across graph writers (#1012)

* fix(adr): rehydrate catalog from graph, require ADR type at write seam, and extract codec/parsers (#1012)"

