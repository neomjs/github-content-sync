---
id: 1011
title: Define repository-scoped ADR identity and custody contracts
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-13T21:14:15Z'
updatedAt: '2026-10-13T22:40:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1011'
author: neo-gpt-sophie
commentsCount: 1
parentIssue: 1010
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 686 Publish the five Fleet Manager ADRs in Institution'
  - '[ ] 174 Teach ADR redirects and repository-qualified references once'
  - '[ ] 19729 Keep ADR composition and publication correct after custody moves'
  - '[ ] 1013 Read one canonical ADR across repositories and redirect paths'
  - '[ ] 1012 Preserve ADR identity and authority across graph writers'
closedAt: '2026-10-13T22:40:06Z'
---
# Define repository-scoped ADR identity and custody contracts

## Context and problem
[D19427](https://github.com/orgs/neomjs/discussions/19427) converged on explicit legacy namespace assignment. A mutable repository slug alone would still allow a renamed repository's old URL to retarget a decision. A later file move must change custody, not identity.

## Architectural reality
Brain owns `learn/agentos/decisions/0006-adrs-as-graph-queryable-entities.md` and `0040-agentos-extraction-topology.md`; Engine owns ADR 0031. Current `AdrIngestor.parseAdr` emits bare `adr-NNNN`, while ADR 0040 §2.7 already classifies custody by subject.

## The fix
Amend ADRs 0006 and 0040 in one decision PR. Specify the serialized identity and canonical manifest/alias contract for forge authority, evidenced stable repository identity, the full organization/repository namespace at assignment, and number. Specify a deterministic repository-local allocation rule, including duplicate/reserved-slot refusal and qualified new references. Define the redirect frontmatter convention once in ADR 0006; ADR 0031 and Skills consume it by reference. Record reader-before-move sequencing, authority-field ownership, source-unavailability behavior, and the existing no-prune retirement boundary.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| ADR 0006 identity, aliases and allocation (amended) | D19427 §§1–4,7 | Stable, explicitly assigned identity; exact serialization and local allocation rule | Unproved continuity, duplicate/reserved slot or ambiguous alias refuses; no inferred birth namespace | Amend 0006 | Worked equal-number, rename/name-reuse and legacy-reference examples |
| Redirect document contract (new) | D19427 §§5,7 | One explicit non-ADR classification, target/custody evidence and no ADR-authority frontmatter | Missing target reports unavailable; no duplicate ADR | Define once in 0006 | Old/new/both/neither state table usable by all readers |
| ADR 0040 custody boundary (amended) | D19427 §§2,6,7 | Five whole FM records belong to Institution; site outputs stay Engine-owned | Missing file never means decision retirement; 0037 semantics remain separate | Amend 0040 | Cut table and successor-risk comparison |

## Acceptance criteria
- [ ] AC1: The accepted ADR text fixes serialized identity, proof/alias rules and repository-local allocation without an org-wide allocator.
- [ ] AC2: The single redirect convention and canonical-source declaration cover duplicate claims, unavailable sources and reserved aliases; no reader is described as already compatible.
- [ ] AC3: Historical citation authority is independent of custody; ADR 0006's Engine ticket and ADR 0041's Brain PR are worked controls. Existing ADR type/apoptosis protections are preserved.
- [ ] AC4: The amendments state the five custody rows, preserve the runtime-continuity amendments, distinguish publication choice from source custody, and expressly exclude unrelated retirement/renumber semantics.

## Out of scope and avoided traps
No source moves or data writes; no new ADR number or global allocator. Do not invent creation provenance, derive identity from current custody, or turn 0037's move into a semantic repair.
Decision Record impact: amends ADR 0006 and ADR 0040; Engine ADR 0031 is a native-linked companion.

## Related and retrieval
Source: [D19427](https://github.com/orgs/neomjs/discussions/19427) §§1–7. Decision Record: Required — ADR 0006 / 0031 / 0040 amendments; the applicable amendment must be accepted before the dependent implementation merges. Human-only PR merge authority is unchanged.
Origin Session ID: 2bee7cb2-c7fa-4367-a104-25694d0050c3
Retrieval Hint: "D19427 repository-owned ADR bare number collision shared summary vector redirect reader custody migration". Source cuts: Brain `d628999a`, Engine `731b70d1` (includes the runtime-continuity amendment), Institution `e5aa4238`; revalidate before implementation.

Parent: https://github.com/neomjs/neo-agent-brain/issues/1010
Design authority: [D19427, accepted D + §7 fold](https://github.com/orgs/neomjs/discussions/19427), 2026-10-13T20:55:14Z, with [Ada's version-bound approval](https://github.com/neomjs/neo/discussions/19427#discussioncomment-18909251). Existing behavior is evidence for the amendment, not silently reinterpreted authority.
ADR successor-risk: adr-authority-pending — the named accepted ADR amendment must precede dependent implementation; no live cut or PR is authorized by ticket creation alone.

## Discussion Criteria Mapping
The leaf's ACs implement the D19427 sections named in its Contract Ledger; cross-repository coverage and the STEP_BACK partials are reconciled once on the native parent.

Creation sweeps: latest 20 open Brain issues, all-state newest 30 A2As, MC query "AdrIngestor bare adr id numbered decisions files qualified PR reference", and own assignments rechecked at 2026-10-13 21:14:13 UTC. Only the new parent overlaps; no duplicate leaf/claim. Earlier C and subsequent D rationale are dispositioned in the source Discussion.

## Timeline

- 2026-10-13T21:14:15Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-13T21:14:18Z @neo-gpt-sophie added parent issue #1010
- 2026-10-13T21:14:21Z @neo-gpt-sophie added the `documentation` label
- 2026-10-13T21:14:24Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-13T21:14:25Z @neo-gpt-sophie added the `ai` label
- 2026-10-13T21:14:28Z @neo-gpt-sophie added the `architecture` label
- 2026-10-13T21:17:13Z @neo-gpt-sophie marked this issue as blocking #1012
- 2026-10-13T21:17:15Z @neo-gpt-sophie marked this issue as blocking #1013
- 2026-10-13T21:17:27Z @neo-gpt-sophie marked this issue as blocking #19729
- 2026-10-13T21:17:28Z @neo-gpt-sophie marked this issue as blocking #174
- 2026-10-13T21:17:28Z @neo-gpt-sophie marked this issue as blocking #686
- 2026-10-13T21:18:59Z @neo-gpt-sophie cross-referenced by #1010
### @neo-gpt-sophie - 2026-10-13T21:26:09Z

Prescription checked: `learn/agentos/decisions/0006-adrs-as-graph-queryable-entities.md` owns typed decision authority, and `0040-agentos-extraction-topology.md` §2.7 owns subject-based custody. The fix belongs in those contracts before parser/migration changes; a filesystem move or config patch would leave identity and reader authority undefined. Independent prescription evidence: Ada's D19427 STEP_BACK18908908 authority/consumer rows, its exact §7 confirmation18909251, and Euclid18909349. Same-context author carve applies; the six-stage creation chain and source reads remain in context. Refreshed Brain dev440b0227 has no changes to the two ADRs or the inspected identity sources since d628999a. Sole assignment verified; no open Brain PR. This first leaf performs no file custody or live-data mutation.

- 2026-10-13T21:32:13Z @neo-gpt-sophie cross-referenced by PR #1015
- 2026-10-13T22:03:26Z @neo-gpt-sophie referenced in commit `e6d001d` - "docs(agentos): retain ADR numbers after custody moves (#1011)"
- 2026-10-13T22:40:06Z @tobiu referenced in commit `221b28a` - "docs(agentos): define repository-owned ADR contracts (#1011) (#1015)

* docs(agentos): define repository-owned ADR contracts (#1011)

* docs(agentos): retain ADR numbers after custody moves (#1011)"
- 2026-10-13T22:40:07Z @tobiu closed this issue

