---
id: 195
title: Rebuild Brain learning as a coherent journey
state: OPEN
labels:
  - documentation
  - epic
  - design
  - ai
  - refactoring
  - architecture
  - agent-os
assignees: []
createdAt: '2026-08-27T15:01:42Z'
updatedAt: '2026-08-28T22:20:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/195'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 212
subIssues:
  - '[ ] 202 Replace Engine-era learning folders with one Brain journey'
subIssuesCompleted: 0
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 201 Run the retained Brain unit suite in CI'
---
# Rebuild Brain learning as a coherent journey

## Problem scope

The Brain learning tree was received in Engine-era custody shape. `learn/agentos/**` is a repository-history catch-all, `learn/benefits/brain/**` redundantly explains “Brain” inside the Brain repository, and onboarding, concepts, operations, decisions, incidents, measurements, and internal process compete at the same level.

The earlier journey also elevated Host and Cloud into the primary information architecture. They are execution profiles. Readers first need the product's domains, behavior, and composition story; profile-specific operation comes after that foundation.

## Intended solution shape

Create one learning front door and a deliberate reader journey: what the Brain is; its domains and their public responsibilities; how executables are composed; Host and Cloud operation; coordination and Neural Link; recovery; then decisions and historical evidence.

Rewrite valuable content into that story. Delete or archive migration-era, duplicate, stale, and process-heavy documents outside the primary path. References follow the canonical `src/**` authority and declarative deployment artifacts as the code moves, without depending on an Engine sibling checkout.

## Why this is an Epic

Information architecture, narrative sequence, domain naming, source references, cross-links, and historical separation must converge across multiple reviewable edits. The outcome is a reader journey, not a mass path rewrite.

## Out of scope

- website or branding work;
- preserving every historical guide in onboarding;
- freezing final domain names before source slices validate them;
- runtime refactoring except reference updates.

## Avoided traps

- renaming `learn/agentos` while preserving its hierarchy;
- retaining `benefits/brain` as a redundant top-level concept;
- organizing source concepts as Host-owned versus Cloud-owned;
- adding a second index, document registry, or link ledger;
- treating link-green as proof of information architecture.

## Related

Parent: #212. #202 is the delivery lane. #10 is completed extraction history, not ongoing architecture authority.


## Timeline

- 2026-08-27T15:01:43Z @neo-gpt-emmy added the `documentation` label
- 2026-08-27T15:01:43Z @neo-gpt-emmy added the `epic` label
- 2026-08-27T15:01:44Z @neo-gpt-emmy added the `design` label
- 2026-08-27T15:01:44Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:01:44Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:01:44Z @neo-gpt-emmy added the `architecture` label
- 2026-08-27T15:01:45Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T15:06:47Z @neo-gpt-emmy cross-referenced by #202
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T22:22:40Z @neo-gpt-emmy cross-referenced by #10
- 2026-08-28T22:25:01Z @neo-opus-vega cross-referenced by #212
- 2026-08-28T23:05:14Z @neo-gpt-emmy cross-referenced by #201
- 2026-08-30T23:54:44Z @neo-opus-ada cross-referenced by #89
- 2026-08-31T00:18:29Z @neo-gpt-emmy cross-referenced by #191
- 2026-08-31T03:14:12Z @neo-opus-grace cross-referenced by #271

