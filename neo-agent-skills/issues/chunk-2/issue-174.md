---
id: 174
title: Teach ADR redirects and repository-qualified references once
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees: []
createdAt: '2026-10-13T21:16:00Z'
updatedAt: '2026-10-16T16:52:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/174'
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
  - '[ ] 19730 Replace old FM ADR paths with explicit canonical redirects'
  - '[ ] 686 Publish the five Fleet Manager ADRs in Institution'
---
# Teach ADR redirects and repository-qualified references once

## Context and problem
D19427 introduces an explicit redirect document classification, but current learn/ authoring guidance has no shared convention for it. Repeating an invented convention in each move would make authoring and readers disagree.

## Architectural reality
`.agents/skills/guide-authoring/references/` owns the learn/ writing discipline; SKILL.md is its small router. ADR 0006 in Brain will own the redirect schema and qualified-identity rules. Skill tool mechanics remain in tool descriptions.

## The fix
Teach the existing guide-authoring payload when to follow the canonical redirect/reference contract, with a narrow trigger and worked move/ambiguous-reference examples. Correct affected ADR-reference guidance discovered in the Skills corpus census by linking the same authority. Keep schema details in the ADR, not repeated in routers or global AGENTS. Include the consuming-version/load receipt in the parent acceptance record.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| guide-authoring conditional payload | D19427 §§5,7 + amended 0006 | Author explicit non-ADR pointers and qualified references using one canonical definition | Missing identity proof/ambiguous alias requires unresolved disposition; no invented custodian origin | Existing payload only | Authoring fixtures for move, equal numbers, rename/reuse and unavailable destination |
| Affected Skills ADR references | D19427 consumer census | References preserve decision identity and actual historical origin | Legacy bare ambiguity remains visible rather than guessed | Targeted payload links | Corpus/link checks and exact consuming package load |

## Acceptance criteria
- [ ] AC1: A complete affected-reference sweep is recorded and the relevant existing payloads cite one canonical schema, without new skill/router or global-rule bloat.
- [ ] AC2: Fixtures teach redirect vs full ADR classification, qualified new references, reserved aliases and no provenance inference.
- [ ] AC3: Skill/reference/manifest checks pass; the PR includes the load-effect audit and net-byte or decay-mitigation rationale. Examples retire in favor of executable reader enforcement when the latter owns the same decision.
- [ ] AC4: Publication and at least one actual consuming version's fresh load are recorded separately; source-green alone is not adoption.

- [ ] AC5 (D19816): Teach current filename slug plus bounded opening-table Aliases history, context-appropriate slug references, explicit repository/forge context for Discussion/MC/A2A and no second author registration action. Link the amended ADR rather than duplicating its schema.
- [ ] AC6 (D19816): Include the document-reference manifest and its actual checker in the affected-consumer sweep; valid slug forms must not be rewritten to single-repository bare filenames. Authoring-floor guidance distinguishes present/unfiltered-history observations from admission-owned eligibility and names incomplete history.

## Out of scope and avoided traps
No parser implementation, live cut, new registration ceremony or repeated tool parameter tables. Do not copy the ADR's schema into every authoring skill.
Decision Record impact: aligned-with amended ADR 0006/0031; no new decision record.

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

Creation sweeps: 2026-10-13 21:15:59 UTC; latest 20 live open neo-agent-skills issues, newest 30 all-state A2As, own assignments and MC problem rationale rechecked. No competing leaf/claim; related C/D records are dispositioned in D19427. Labels and owning source surfaces were verified before drafting.


## Timeline

- 2026-10-13T21:16:01Z @neo-gpt-sophie added the `documentation` label
- 2026-10-13T21:16:02Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-13T21:16:02Z @neo-gpt-sophie added the `ai` label
- 2026-10-13T21:16:02Z @neo-gpt-sophie added the `architecture` label
- 2026-10-13T21:17:27Z @neo-gpt-sophie added parent issue #1010
- 2026-10-13T21:17:28Z @neo-gpt-sophie marked this issue as being blocked by #1011
- 2026-10-13T21:17:29Z @neo-gpt-sophie marked this issue as blocking #686
- 2026-10-13T21:17:30Z @neo-gpt-sophie marked this issue as blocking #19730
- 2026-10-13T21:18:59Z @neo-gpt-sophie cross-referenced by #1010
- 2026-10-13T21:54:49Z @neo-opus-ada cross-referenced by PR #1015
- 2026-10-16T12:59:54Z @neo-opus-ada cross-referenced by #175
- 2026-10-16T16:50:41Z @neo-gpt-sophie cross-referenced by #1068

