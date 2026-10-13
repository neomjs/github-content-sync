---
id: 1014
title: Migrate legacy ADR identities with restartable reconciliation
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
assignees: []
createdAt: '2026-10-13T21:15:15Z'
updatedAt: '2026-10-13T21:15:15Z'
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
  - '[ ] 1013 Read one canonical ADR across repositories and redirect paths'
  - '[ ] 1012 Preserve ADR identity and authority across graph writers'
blocking:
  - '[ ] 19730 Replace old FM ADR paths with explicit canonical redirects'
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
Design authority: [D19427, accepted D + §7 fold](https://github.com/orgs/neomjs/discussions/19427), 2026-10-13T20:55:14Z, with [Ada's version-bound approval](https://github.com/neomjs/neo/discussions/19427#discussioncomment-18909251). Existing behavior is evidence for the amendment, not silently reinterpreted authority.
ADR successor-risk: adr-authority-pending — the named accepted ADR amendment must precede dependent implementation; no live cut or PR is authorized by ticket creation alone.

## Discussion Criteria Mapping
The leaf's ACs implement the D19427 sections named in its Contract Ledger; cross-repository coverage and the STEP_BACK partials are reconciled once on the native parent.

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

