---
id: 686
title: Publish the five Fleet Manager ADRs in Institution
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees: []
createdAt: '2026-10-13T21:16:03Z'
updatedAt: '2026-10-13T21:16:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/686'
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
  - '[ ] 174 Teach ADR redirects and repository-qualified references once'
  - '[ ] 19729 Keep ADR composition and publication correct after custody moves'
  - '[ ] 1013 Read one canonical ADR across repositories and redirect paths'
  - '[x] 1011 Define repository-scoped ADR identity and custody contracts'
blocking:
  - '[ ] 19730 Replace old FM ADR paths with explicit canonical redirects'
---
# Publish the five Fleet Manager ADRs in Institution

## Context and problem
The operator's custody direction and D19427 place the five whole FM records in Institution. Today they remain in Engine; splitting 0038 because Brain enforces it would confuse subject ownership with implementation.

## Architectural reality
The source records are Engine `learn/agentos/decisions/` 0020-agent-harness-concept, 0032-institution-cockpit-render-model, 0034-electron-shell-architecture, 0037-fleet-manager-outward-door-topology and 0038-fm-client-topology. Source cut `731b70d1` includes the merged runtime/admission continuity amendments. Institution's local discovery references include its harness README and CARD-CONTRACT.

## The fix
Publish the five complete records at the same filenames under Institution's `learn/agentos/decisions/`, pinned in the reviewed custody manifest. Correct their relative links and explicitly evidenced historical issue/PR references for the new repository. Update Institution discovery references. A staged destination copy is not a second authority: admission follows the manifest, and the cross-repo window retains the last admitted state if undeclared duplicate claims appear.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| Five destination ADR files | D19427 §2 + amended 0040 | Whole records preserve decisions and landed 0034/0038 amendment under Institution custody | Source drift reopens comparison; undeclared duplicate claims cannot win | Five records + local discovery links | Source hash/semantic diff, relative-link and reader checks |
| Historical references | D19427 §3 | Preserve evidenced issue/PR origin, independent of new custodian | Unproved origin named for reconciliation, never guessed | Qualified citations | 0006/0041 controls plus moved-record inventory |
| 0037 disposition | D19427 §2 | Custody changes only; semantic-disposition stays pending | No claim that monorepo clauses are repaired | Explicit disposition pointer | Diff excludes unrelated semantic amendment |

## Acceptance criteria
- [ ] AC1: All five whole records are present at the declared paths with verified source/candidate hashes, preserving the 0034/0038 continuity amendment.
- [ ] AC2: Historical references and relative links resolve to their actual authorities; Institution discovery reaches each canonical record.
- [ ] AC3: Reader compatibility and the publication outcome are witnessed before custody admission. The destination staging/manifest transition has no competing authoritative ADR document.
- [ ] AC4: 0037's semantic amendment remains explicitly pending; Engine/Brain-owned decisions stay in their owners.
- [ ] AC5: The delivery packet records final live canonical admission as an unresolved parent-owned post-merge check, coordinated with Engine redirects and graph/KB reconciliation.

## Out of scope and avoided traps
No graph mutation, Engine deletion, ADR renumbering, broad directory move or 0037 semantic rewrite. Do not treat source publication alone as a completed custody cut.
Decision Record impact: aligned-with amended 0006/0031/0040; retains 0034/0038 accepted content.

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

Creation sweeps: 2026-10-13 21:16:03 UTC; latest 20 live open neo-agent-institution issues, newest 30 all-state A2As, own assignments and MC problem rationale rechecked. No competing leaf/claim; related C/D records are dispositioned in D19427. Labels and owning source surfaces were verified before drafting.

## Timeline

- 2026-10-13T21:16:04Z @neo-gpt-sophie added the `documentation` label
- 2026-10-13T21:16:05Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-13T21:16:05Z @neo-gpt-sophie added the `ai` label
- 2026-10-13T21:16:05Z @neo-gpt-sophie added the `architecture` label
- 2026-10-13T21:17:28Z @neo-gpt-sophie added parent issue #1010
- 2026-10-13T21:17:28Z @neo-gpt-sophie marked this issue as being blocked by #1011
- 2026-10-13T21:17:29Z @neo-gpt-sophie marked this issue as being blocked by #1013
- 2026-10-13T21:17:29Z @neo-gpt-sophie marked this issue as being blocked by #19729
- 2026-10-13T21:17:29Z @neo-gpt-sophie marked this issue as being blocked by #174
- 2026-10-13T21:17:30Z @neo-gpt-sophie marked this issue as blocking #19730
- 2026-10-13T21:18:59Z @neo-gpt-sophie cross-referenced by #1010

