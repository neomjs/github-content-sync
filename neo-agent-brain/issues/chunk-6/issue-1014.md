---
id: 1014
title: Migrate legacy ADR identities with restartable reconciliation
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gemini-julia
createdAt: '2026-10-13T21:15:15Z'
updatedAt: '2026-10-21T00:16:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1014'
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
  - '[ ] 1013 Add ADR redirect, alias and qualified-census compatibility'
  - '[x] 1012 Preserve ADR identity and authority across graph writers'
blocking:
  - '[x] 19730 Replace old FM ADR paths with explicit canonical redirects'
---
# Migrate legacy ADR identities with restartable reconciliation

## Context and problem
Future writes alone cannot repair the 41 existing bare ADR identities, 11 observed type/vector anomalies, their incident edges, or separately repository-bound KB documents. The [census](https://github.com/neomjs/neo/discussions/19427#discussioncomment-18908324) is a series of lean reads, not an apply-ready backup. One observed summary pointer is shared by `adr-0038` and a non-ADR node.

## Architectural reality
Graph and KB/vector state have separate lifecycles. `ai/scripts/migrations/normalizeGraphIdentities.mjs` is a verified one-shot dry-run/apply/idempotence sibling, but its SQLite-only transaction and duplicate-edge dropping are NOT sufficient for this cross-store, provenance-preserving migration.

## The fix
Provide one manifest-driven migration entry under `ai/scripts/migrations/`, with a thin command over reusable owning-domain behavior where needed. Default to dry-run. Produce the complete source/data-bound inventory, explicit phases, preconditions, resumable application and verified recovery/rollback. Identity normalization can occur while custody remains Engine; later custody publication consumes the same manifest contract. Do not perform live apply merely by merging this code.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| Migration command and reviewed manifest (new) | D19427 §§2,6 | Per-record legacy/canonical identity, repo proof, custody/ref/path/hash, aliases, historical references, node/vector/edge state and semantic disposition | Ambiguous assignments, drifted preconditions or incomplete inventory refuse before writes | Recovery runbook + command help | Dry-run against complete disposable state and reviewed real 41-record inventory |
| Cross-store apply/reconcile (new) | D19427 §§4–6 | Correct ADR type/document binding; preserve every incident edge/property/provenance and unrelated vector owner | Durable phase records reconcile partial completion; repeated completed phase is no-op | Failure/rollback contract | Failure injection at every store boundary, resume and rollback controls |
| Final reader verification | D19427 §§3,5–6 | Old and qualified references resolve to one canonical decision with honest document/KB availability | Neither-present retains identity and reports availability separately | Cut checklist | Real reader proofs, not counts alone |

## Acceptance criteria
- [ ] AC1: The reviewed 41-record inventory includes all fields above and every owner of any reused vector pointer; source/data preconditions are explicit. The lean census is not substituted.
- [ ] AC2: Dry-run is default; inconsistent identity proof, duplicate canonical claims, lost provenance or changed source/data refuse safely.
- [ ] AC3: Disposable-state tests cover the 11 anomaly shapes, a shared summary vector with a non-ADR owner, all incident edge metadata and canonical issue/PR origins.
- [ ] AC4: Interrupted cross-store phases resume deterministically; completed reruns are no-ops; a validated backup/recovery route restores graph, vectors and KB state without harming unrelated consumers.
- [ ] AC5: A runbook binds an approved manifest, installed reader versions, writer coordination and final native graph/KB readback. Live apply/readback is explicitly post-merge acceptance retained on the parent until witnessed.

- [ ] AC6 (D19816): The reviewed manifest/recovery route preserves the same identity, historical alias scopes and numeric/locator reservations maintained by normal admission. It is not a separate human-edited registration surface for new records.
- [ ] AC7 (D19816): Rebuild and interrupted-phase controls cover rename, move, combined A-old→B-new, source loss and admission-record loss; missing proof refuses rather than reconstructing assignment from a basename or re-scoping history to current custody.
- [ ] AC8 (D19816): Consume #1013's verified namespace-discovery and reader capability rather than adding a second reader implementation; revalidate source revisions/hashes and reader authorization before a real cut.

## Out of scope and avoided traps
No automatic live cut, secret export, unrelated retirement/renumber semantics, blind vector deletion, or separate shadow graph. Do not lift the sibling's duplicate-edge dropping or infer success from equal counts.
Decision Record impact: depends-on accepted identity/custody amendments.
Structural pre-flight: one-shot migration role matches `ai/scripts/migrations/normalizeGraphIdentities.mjs`; no new directory. The sibling's correctness limits above remain explicit.

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

Creation sweeps: 2026-10-13 21:15:14 UTC; latest 20 live open Brain issues and all-state newest 30 A2As rechecked, no new competing leaf/claim; own assignments contain the parent/contract and unrelated existing work. MC problem sweep again returns the same explicitly dispositioned C/D records and unrelated review. Labels and owning source surfaces were verified before drafting.


## Timeline

- 2026-10-13T21:15:16Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-13T21:15:17Z @neo-gpt-sophie added the `ai` label
- 2026-10-13T21:15:17Z @neo-gpt-sophie added the `architecture` label
- 2026-10-13T21:15:19Z @neo-gpt-sophie added parent issue #1010
- 2026-10-13T21:17:18Z @neo-gpt-sophie marked this issue as being blocked by #1012
- 2026-10-13T21:17:20Z @neo-gpt-sophie marked this issue as being blocked by #1013
- 2026-10-13T21:17:30Z @neo-gpt-sophie marked this issue as blocking #19730
- 2026-10-13T21:18:59Z @neo-gpt-sophie cross-referenced by #1010
- 2026-10-16T16:50:41Z @neo-gpt-sophie cross-referenced by #1068
- 2026-10-16T16:51:22Z @neo-gpt-sophie marked this issue as being blocked by #1068
- 2026-10-19T00:45:00Z @neo-opus-grace cross-referenced by #686
- 2026-10-19T11:36:36Z @neo-opus-grace cross-referenced by #1181
- 2026-10-19T11:46:19Z @neo-opus-grace cross-referenced by PR #1182
- 2026-10-19T11:50:53Z @neo-opus-grace cross-referenced by PR #741
- 2026-10-21T00:16:18Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-21T00:40:52Z @neo-gemini-julia cross-referenced by PR #1234
- 2026-10-21T00:53:38Z @neo-gemini-julia referenced in commit `98b6a35` - "feat(migrations): migrate legacy ADR identities with restartable reconciliation (#1014)"

