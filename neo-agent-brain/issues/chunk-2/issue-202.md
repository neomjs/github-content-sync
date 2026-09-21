---
id: 202
title: Replace Engine-era learning folders with one Brain journey
state: OPEN
labels:
  - documentation
  - enhancement
  - design
  - ai
  - refactoring
  - architecture
  - agent-os
assignees: []
createdAt: '2026-08-27T15:06:46Z'
updatedAt: '2026-08-28T22:20:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/202'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 195
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 200 Unify embedding admission across provider paths'
  - '[x] 198 Remove Engine projections after Brain source takes ownership'
  - '[x] 199 Move Dream into a Cloud-owned domain'
  - '[x] 197 Establish deploy/host and independent deploy/cloud packages'
blocking: []
---
# Replace Engine-era learning folders with one Brain journey

## Problem

The learning tree reflects Engine-era custody rather than reader intent. `learn/agentos/**` and `learn/benefits/brain/**` mix onboarding, concepts, operations, ADRs, incidents, measurements, and internal process under redundant repository-history names.

The earlier body prescribed final Host/Cloud source paths and treated Dream as a domain. The architecture now has one domain-owned source tree; Host and Cloud are executable profiles, and REM digestion belongs to Evolution.

## Scope

Create `learn/README.md` as the canonical journey. Rewrite, merge, move, archive, or delete current material into a small reader-goal structure that explains product domains first, executable composition second, Host/Cloud operation third, and decisions/history as reference.

Update source references as domain slices land. Primary guides must resolve from a fresh Brain checkout and must not depend on Engine sibling paths.

## Acceptance Criteria

- [ ] `learn/README.md` provides one coherent path from introduction through domains, composition, operation, and reference.
- [ ] `learn/agentos/**` and `learn/benefits/brain/**` no longer exist as redundant roots.
- [ ] Primary learning explains Memory, Knowledge, Evolution/REM, Orchestration, and collaboration before Host/Cloud execution profiles.
- [ ] Valuable content is rewritten or moved by reader goal; stale, duplicate, migration-era, and process-heavy pages leave the primary journey.
- [ ] Primary guides use canonical Brain source and deployment references and require no sibling Engine checkout.
- [ ] Decisions, incidents, measurements, and historical evidence are clearly separated from onboarding and task guides.
- [ ] README learning links enter through this journey; no document registry, link ledger, copied navigation tree, or second index is added.

## Out of scope

Website work, branding, or runtime changes beyond reference corrections.

## Relationships

Parent: #195. Architecture authority: #212. Completed extraction ticket #10 is historical context only.


## Timeline

- 2026-08-27T15:06:48Z @neo-gpt-emmy added the `documentation` label
- 2026-08-27T15:06:48Z @neo-gpt-emmy added the `enhancement` label
- 2026-08-27T15:06:48Z @neo-gpt-emmy added the `design` label
- 2026-08-27T15:06:49Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:06:49Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:06:49Z @neo-gpt-emmy added the `architecture` label
- 2026-08-27T15:06:49Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-28T11:41:19Z @neo-opus-vega cross-referenced by PR #205
- 2026-08-28T22:20:27Z @neo-gpt-emmy cross-referenced by #195
- 2026-08-28T22:22:40Z @neo-gpt-emmy cross-referenced by #10

