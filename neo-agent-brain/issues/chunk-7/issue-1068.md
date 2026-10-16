---
id: 1068
title: Define readable ADR slug references and retained alias history
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-16T16:50:39Z'
updatedAt: '2026-10-16T19:02:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1068'
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
  - '[ ] 1014 Migrate legacy ADR identities with restartable reconciliation'
  - '[ ] 1013 Read one canonical ADR across repositories and redirect paths'
  - '[ ] 1012 Preserve ADR identity and authority across graph writers'
closedAt: '2026-10-16T19:02:55Z'
---
# Define readable ADR slug references and retained alias history

## Context and problem
[D19816](https://github.com/orgs/neomjs/discussions/19816) selected readable slug references without replacing the accepted immutable decision key. The current ADR 0006 §§2.6–2.10 specifies repository-qualified numbered identity and custody, but does not yet define this slug/history/admission contract. The prior amendment #1011 is closed; this leaf records the newly approved delta under #1010.

Design authority: D19816 body `2026-10-16T16:45:08Z`, selected A, with the signals below. Its outcome is: “A maintainer cites a meaningful slug and reaches the intended decision, or a reader-safe ambiguity/refusal,” without a second registration step.

## Architectural reality and fix
Amend the existing `learn/agentos/decisions/0006-adrs-as-graph-queryable-entities.md` in place. At Brain `924b8bdd`, its four-field numeric canonical key, source/admission authority, historical reservations, redirect convention, reader-before-move sequencing and apoptosis protection remain authoritative. Add the D19816 naming/carrier contract there, replacing superseded reference prose rather than copying the discussion transcript.

No new ADR number, module, service or schema carrier is needed. Normal admission and migration implementation remain in #1012 / #1014; readers/census in #1013; guidance in neomjs/neo-agent-skills#174.

## Contract Ledger
| Surface | Authority | Required behavior | Refusal / boundary | Docs / evidence |
| --- | --- | --- | --- | --- |
| Human references and opening metadata | D19816 selected A §1 | Context-appropriate slug references; filename current spelling; optional source-local Aliases row | No seat/cwd/hosting-repository guess; bounded root-table parser, duplicate refusal and explicit legacy cases | ADR syntax and worked controls |
| Identity-bound admission evidence | D19816 §§2–3, retained ADR0006 key | Normal writer maintains canonical assignment, pinned source, historically scoped aliases, reservations and recovery evidence | No extra author registration; no prefix authority; conflicting same-batch claims admit neither | Create/rename/move/record-loss vectors |
| Authoring and delivery | D19816 §4 and consumer table | Measured author flow, retained reservations, explicit writer/reader/migration/Skills duties | Git history is advisory; no source-only reconstruction, live cut or private-reader widening | Normative contract with implementation boundaries |

## Acceptance criteria
- [ ] AC1: Preserve canonical identity/encoding, type protection, historical-origin/custody separation and the existing redirect convention; meaningful slugs are locators, not replacement keys.
- [ ] AC2: Specify all three citation forms and context per surface; define the opening-table window, token grammar, duplicate-table/row refusal, and the two legacy metadata shapes without body-wide fallback.
- [ ] AC3: Define normal admission's retained record and alias scope at admission. Current/source history union preserves admitted spellings; same-batch conflicts admit neither; moving A→B never fabricates B's old alias.
- [ ] AC4: Work through create, rename, move, combined rename/move, source loss, admission-record loss and reader-permission cases. State that reader/writer/migration receipts remain implementation acceptance.
- [ ] AC5: Preserve admission-owned allocation and historical floors; present/unfiltered-history observations aid authors but incomplete history cannot certify eligibility. Require actual author-action measurement without a second registration surface or auto-numbering prerequisite.
- [ ] AC6: Map the existing consumer responsibilities and make this amendment a merge prerequisite for adopting the delta. Focused ADR checks and cross-family review pass; no live data/file cut is performed.

## Out of scope and avoided traps
No canonical rekey, source move, graph migration, new registry, forced filename renames, source-carried key/scoped metadata, or material change to ADR0040 custody. Do not equate source claims with admitted authority, re-scope history under a new custodian, or treat nominal lint success as current-population admission.

Decision Record: REQUIRED — this amendment to ADR0006 must merge before dependent implementation adopts D19816.
Decision Record impact: amends ADR0006; retains D19427's existing identity/custody decisions.
Future decay: this ADR is the single contract; downstream skills cite it and executable enforcement retires duplicate examples/guidance.

## Signal Ledger
All signals bind D19816 body `2026-10-16T16:45:08Z`.
- `gpt`: Sophie [AUTHOR_SIGNAL](https://github.com/neomjs/neo/discussions/19816#discussioncomment-18978088).
- `gemini`: Julia [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/19816#discussioncomment-18978119).
- `claude`: Ada [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/19816#discussioncomment-18978148).
Independent [STEP_BACK](https://github.com/neomjs/neo/discussions/19816#discussioncomment-18977735), corrected/reconfirmed 2026-10-16T16:43:24Z, acknowledges all eight points.

## Unresolved Dissent
None at the bound body. Ada's advisory authoring-floor expectation is carried: present files plus unfiltered history can aid authors, but only admission's proved population certifies eligibility.

## Unresolved Liveness
The three signing families have active AgentIdentity participationStatus, read through who_is_online at 2026-10-16T16:46:16Z. Kimi identities are operator_benched and supply no signal; no absence is treated as consent. This is not a core-value, critical-gate or consensus-rule amendment.

## Discussion Criteria Mapping
D19816 citation/carrier → AC1–2; normal admission and scope → AC3; falsifying operation vectors → AC4; author flow/allocation → AC5; STEP_BACK consumer acknowledgment and authority sequence → AC6.

## Discovery and ownership
Sophie retains this one-PR documentation leaf. Parent #1010 remains the integrated outcome owner. Own-assignment body sweep found #1010 and unrelated #1005/#700. Historical ADR search found #1011 closed, not an open duplicate; KB returned parent #1010 but no equivalent leaf. MC semantic search was noisy; the current discussion and source are the rationale authority. Brain structure-map exited0 under Node24; the existing decision file is the owning surface. No new .mjs or turn-loaded instruction file is proposed.

Origin Session ID: c8334b16-9aa5-4b9c-99c4-3da0453c0128
Retrieval Hint: D19816 readable ADR slugs historical aliases retained admission source-only reconstruction refusal.

Creation freshness: live created-descending latest20 Brain open issues and all-state newest30 A2As checked immediately before creation at 2026-10-16 16:50 UTC; no competing amendment. Targeted own-session MC recall recovered the prior carrier and operator-direction decisions, both folded into D19816; no prior ruling is displaced.



## Timeline

- 2026-10-16T16:50:40Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-16T16:50:41Z @neo-gpt-sophie added the `documentation` label
- 2026-10-16T16:50:41Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-16T16:50:41Z @neo-gpt-sophie added the `ai` label
- 2026-10-16T16:50:42Z @neo-gpt-sophie added the `architecture` label
- 2026-10-16T16:51:16Z @neo-gpt-sophie added parent issue #1010
- 2026-10-16T16:51:18Z @neo-gpt-sophie marked this issue as blocking #1012
- 2026-10-16T16:51:20Z @neo-gpt-sophie marked this issue as blocking #1013
- 2026-10-16T16:51:22Z @neo-gpt-sophie marked this issue as blocking #1014
- 2026-10-16T16:52:26Z @neo-gpt-sophie cross-referenced by #1010
- 2026-10-16T16:52:29Z @neo-gpt-sophie cross-referenced by #1012
- 2026-10-16T16:52:32Z @neo-gpt-sophie cross-referenced by #1013
- 2026-10-16T16:52:35Z @neo-gpt-sophie cross-referenced by #1014
- 2026-10-16T16:52:40Z @neo-gpt-sophie cross-referenced by #174
### @neo-gpt-sophie - 2026-10-16T18:10:31Z

Intake: the declared edit surface is unchanged through refreshed `dev@924b8bdd`; the resident checkout is clean and the live issue remains assigned to Sophie. No overlapping amendment PR is open.

Prescription checked: `learn/agentos/decisions/0006-adrs-as-graph-queryable-entities.md` owns identity, citation, admission and custody; the amendment belongs there. Julia's corrected D19816 STEP_BACK and an independent source read confirm this boundary. Implementation stays in the existing writer/reader/migration/Skills leaves. The graph pre-brief returned NODE_NOT_FOUND for this qualified issue token; live issue/discussion/source and the recovered own-session rationale provide the brief.

Change class: zero-delta (`docs`). Validation will use the existing ADR checks with explicit repository paths and a contract/acceptance-vector review; no runtime implementation claim.

- 2026-10-16T18:15:36Z @neo-gpt-sophie cross-referenced by PR #1069
- 2026-10-16T19:02:55Z @tobiu referenced in commit `9b6c9af` - "docs(adr): define readable slug references (#1068) (#1069)"
- 2026-10-16T19:02:55Z @tobiu closed this issue

