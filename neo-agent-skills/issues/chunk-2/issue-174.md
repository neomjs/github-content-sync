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
updatedAt: '2026-10-13T21:16:00Z'
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

## Out of scope and avoided traps
No parser implementation, live cut, new registration ceremony or repeated tool parameter tables. Do not copy the ADR's schema into every authoring skill.
Decision Record impact: aligned-with amended ADR 0006/0031; no new decision record.

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

