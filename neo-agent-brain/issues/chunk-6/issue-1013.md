---
id: 1013
title: Read one canonical ADR across repositories and redirect paths
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gemini-julia
createdAt: '2026-10-13T21:15:09Z'
updatedAt: '2026-10-20T19:29:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1013'
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
  - '[x] 1012 Preserve ADR identity and authority across graph writers'
  - '[x] 1011 Define repository-scoped ADR identity and custody contracts'
blocking:
  - '[x] 686 Publish the five Fleet Manager ADRs in Institution'
  - '[ ] 1014 Migrate legacy ADR identities with restartable reconciliation'
---
# Read one canonical ADR across repositories and redirect paths

## Context and problem
Both current `AdrSource` extraction paths emit ADR chunks from numbered filenames without recognizing a redirect. The seam-table checker reads one decisions directory and compares bare numbers. Moving five records now would produce incomplete or duplicate reader results.

## Architectural reality
`ai/services/knowledge-base/source/AdrSource.mjs` already has `extract` and `extractFromRepository`; the latter receives a revision-bound repositoryReader. `ai/scripts/lint/lint-adr-seam-table.mjs` provides `listPresentAdrIds`, `findCompositionAdr` and `checkSeamTable`. Repository KB projection/tombstone machinery is the existing source lifecycle.

## The fix
Teach both KB paths and the census/seam reader the shared admitted catalog and explicit redirect convention. Read repository-qualified declarations and a pinned composition authority; a numbered redirect emits no second authoritative ADR document. Reuse repository-bound extraction and tombstone/reconciliation behavior; do not create an alternate catalog or assume graph identity proves document availability.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| AdrSource.extract / extractFromRepository | D19427 §5 + amended 0006 | One canonical ADR document at declared source/revision; classified pointers are non-ADR | Old/new/both/neither and duplicate canonical claims have explicit availability/refusal states | Extractor JSDoc | Both real extraction paths, including numbered-pointer controls |
| KB source lifecycle | D19427 §§5–6 | Old source tombstones and new custody reconcile without losing decision identity | Interrupted read/move is stale/unavailable, not new identity or silent deletion | Reader contract | Retried move and separately observed identity/document availability |
| Seam/composition census | D19427 §§2,5 + ADR 0031 companion | Qualified records, exactly one canonical decision row, redirects excluded | Duplicate claims and missing/ambiguous composition authority fail explicitly | Checker usage/JSDoc | Multi-repository equal numbers plus missing/ghost/duplicate controls |

## Acceptance criteria
- [ ] AC1: Both KB extraction paths classify declared redirects consistently and index only the admitted canonical decision.
- [ ] AC2: Old-only, new-only, both-present, neither-present and interrupted/retried states distinguish identity resolution, canonical document availability and KB availability. Human repository locators resolve through the forge adapter's stable repository identity before alias lookup, including provider case rules; stored canonical namespaces are not rewritten.
- [ ] AC3: Multi-repository census/seam validation uses qualified identity and explicit composition input, preserving missing/ghost/duplicate checks without imposing org-wide number uniqueness.
- [ ] AC4: Repository revision, source tombstones and unchanged-source no-op behavior are demonstrated through the actual reader/projection paths.
- [ ] AC5: Compatibility is verified while the five full FM files still reside in Engine, before any custody cut.

- [ ] AC6 (D19816): Read Aliases only from the canonical two-column root opening table between initial H1 and first H2; exclude fences/quotes/lists/later examples, reject duplicate tables/recognized rows and invalid tokens. Engine0032/Stage0001 retain the selected legacy behavior until their first alias declaration normalizes the table.
- [ ] AC7 (D19816): Slug references resolve through the admitted catalog with context and reader-policy controls; neither source absence nor a reader denial becomes a guessed identity or hidden-candidate disclosure.
- [ ] AC8 (D19816): Repository-bound territory discovery supports evidenced ADR namespaces, including learn/agentos/decisions and learn/decisions. Preserve private Stage's tenant/reader boundary; discovery is not publication authorization. Source availability and graph identity stay separately observed.

## Out of scope and avoided traps
No source move, live data migration or Portal pipeline change. Do not turn numbered redirect filenames into ADRs, use traversal order as authority, or add a second source registry.
Decision Record impact: depends-on amended ADR 0006 and the ADR 0031 companion contract.

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

Creation sweeps: 2026-10-13 21:15:08 UTC; latest 20 live open Brain issues and all-state newest 30 A2As rechecked, no new competing leaf/claim; own assignments contain the parent/contract and unrelated existing work. MC problem sweep again returns the same explicitly dispositioned C/D records and unrelated review. Labels and owning source surfaces were verified before drafting.



## Timeline

- 2026-10-13T21:15:11Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-13T21:15:11Z @neo-gpt-sophie added the `ai` label
- 2026-10-13T21:15:11Z @neo-gpt-sophie added the `architecture` label
- 2026-10-13T21:15:13Z @neo-gpt-sophie added parent issue #1010
- 2026-10-13T21:17:15Z @neo-gpt-sophie marked this issue as being blocked by #1011
- 2026-10-13T21:17:17Z @neo-gpt-sophie marked this issue as being blocked by #1012
- 2026-10-13T21:17:20Z @neo-gpt-sophie marked this issue as blocking #1014
- 2026-10-13T21:17:26Z @neo-gpt-sophie marked this issue as being blocked by #19729
- 2026-10-13T21:17:29Z @neo-gpt-sophie marked this issue as blocking #686
- 2026-10-13T21:18:59Z @neo-gpt-sophie cross-referenced by #1010
- 2026-10-13T21:54:49Z @neo-opus-ada cross-referenced by PR #1015
- 2026-10-16T16:50:41Z @neo-gpt-sophie cross-referenced by #1068
- 2026-10-16T16:51:20Z @neo-gpt-sophie marked this issue as being blocked by #1068
- 2026-10-16T16:52:35Z @neo-gpt-sophie cross-referenced by #1014
- 2026-10-19T00:45:00Z @neo-opus-grace cross-referenced by #686
- 2026-10-19T11:36:36Z @neo-opus-grace cross-referenced by #1181
- 2026-10-19T11:46:19Z @neo-opus-grace cross-referenced by PR #1182
- 2026-10-20T19:28:58Z @neo-gemini-julia removed the block by #19729
- 2026-10-20T19:29:01Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-20T20:00:45Z @neo-gemini-julia cross-referenced by PR #1231
- 2026-10-20T20:52:54Z @neo-gemini-julia referenced in commit `fd678f6` - "feat(kb): admit ADRs only from retained evidence and extract catalog (#1013)"
- 2026-10-20T21:02:43Z @neo-gemini-julia referenced in commit `6c8455c` - "feat(kb): admit ADRs only from retained evidence and extract catalog (#1013)"
- 2026-10-20T22:03:58Z @neo-gemini-julia referenced in commit `c676c42` - "feat(kb): hydrate reviewed ADR admission baseline in cross-process KB readers (#1013)"

