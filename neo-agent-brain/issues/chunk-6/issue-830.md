---
id: 830
title: Consume the published goal-first skills correction in Brain
state: CLOSED
labels:
  - enhancement
  - dependencies
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-03T20:47:23Z'
updatedAt: '2026-10-04T01:04:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/830'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 140
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-04T01:04:35Z'
---
# Consume the published goal-first skills correction in Brain

## Context

Planned Brain consumer step of neomjs/neo-agent-skills#140, graduated from [D#19384](https://github.com/neomjs/neo/discussions/19384). The accepted correction is published as Skills 0.1.29. Euclid owns integration/fresh-session load/replay; Engine's companion is neomjs/neo#19390 / PR neomjs/neo#19391.

## The Problem

Brain's `package.json` and lock still select the earlier 0.1.23 baseline. Source merges in Skills cannot change what a Brain consumer installs. A consumer update is already part of the accepted plan; this leaf supplies its one-PR source change, not a new policy.

## The Architectural Reality

The dependency and lock own selection. `ai/scripts/setup/prepare.mjs` uses the dependency's declared materializer bin, guarded against writing another consumer's facade. `.agents/skills` and `.claude/skills` are ignored projections; this Brain tree has no tracked root AGENTS file. Preserve these ownership boundaries.

## The Fix

Update Brain's declared dependency and npm lock to resolve 0.1.29. Verify the published package and the existing materialization recipe in isolation; record which corrected skill text the resolved package supplies. Do not hand-edit instruction text, introduce a generated root file, or change any application/deployment behavior.

Prescription checked: `package.json`, `package-lock.json`, `ai/scripts/setup/prepare.mjs` — dependency selection and the existing package-owned materializer own this concern; changing Brain services would treat the wrong layer.

## Contract Ledger

| Surface | Authority | Change | Boundary | Evidence |
|---|---|---|---|---|
| Skills dependency/lock | published 0.1.29; Skills #140 | fresh locked install resolves 0.1.29 | no unrelated dependency changes | manifest/lock diff and installed metadata |
| Skills facade | dependency's declared materializer via Brain prepare | existing paths resolve the new package text | no hand-edited/generated tracked rules | isolated materializer/path readback |

Decision Record impact: aligned-with ADR 0007; no instruction policy changed.
Decision Record: Not needed (D#19384 classification retained).
Discussion criteria mapping: #140 consumer-pin obligation → AC-1/2; actual recipient behavior stays on #140.

## Acceptance Criteria

- [ ] AC-1: manifest and lock resolve neo-agent-skills 0.1.29, with no unrelated dependency, Engine pin or runtime changes.
- [ ] AC-2: an isolated install/materialization resolves the package-owned skills and reads the accepted goal-first selection and learning-closure text; record the resolved version and paths.
- [ ] AC-3: the PR records applicable validation and hands its exact source head to Skills #140, explicitly separating this source/fixture evidence from a fresh live recipient load.

## Post-Merge Validation

Fresh Brain consumer loading and behavioral replay remain owned by neomjs/neo-agent-skills#140. This leaf does not close that outcome or authorize daemon/harness restart.

## Out of Scope

FM features, new semantic rules, Engine/Institution consumer changes, live deployment, account/credential changes.

## Avoided Traps

Editing the installed package as canonical source; treating an isolated install as the recipient's load receipt; adding duplicate instruction carriers.

## Sweeps

Live latest 20 open Brain issues and open PRs checked immediately before filing; no consumer-pin equivalent. All-state A2A latest 30 showed Euclid's Engine claim and invitation for remaining consumers, no Brain claim. Own assignments are #426/#306/#48, none overlapping. Memory rationale search returned no relevant result; KB search found package-materialization precedents but no current pin leaf. Skills #140 is the explicit outcome authority. Structure map ran successfully; existing prepare/materializer is the owner, no new source placement.

Related: neomjs/neo-agent-skills#140 · neomjs/neo#19390 · neomjs/neo#19384

Origin Session ID: 01a102a5-481d-7581-9819-eeaf08f87236
Retrieval Hint: "Brain consumer published goal-first skills 0.1.29 package pin isolated materializer load receipt"


## Timeline

- 2026-10-03T20:47:23Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-03T20:47:25Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-03T20:47:25Z @neo-gpt-emmy added the `dependencies` label
- 2026-10-03T20:47:25Z @neo-gpt-emmy added the `ai` label
- 2026-10-03T20:47:25Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-03T20:48:20Z @neo-gpt-emmy added parent issue #140
- 2026-10-03T20:55:35Z @neo-gpt-emmy cross-referenced by PR #831
- 2026-10-04T01:04:35Z @tobiu referenced in commit `ffb4bf0` - "build(skills): consume the goal-first correction (#830) (#831)"
- 2026-10-04T01:04:35Z @tobiu closed this issue

