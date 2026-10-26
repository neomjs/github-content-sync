---
id: 1299
title: Activate declared-model family only after admitted effective use
state: OPEN
labels:
  - enhancement
  - no auto close
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-26T08:34:57Z'
updatedAt: '2026-10-26T08:34:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1299'
author: neo-gpt-sophie
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 700 An auto-provisioned agent identity carries no model family, so family-keyed budgets, aliases and wakes skip it'
blocking: []
---
# Activate declared-model family only after admitted effective use

## Context

This standalone **post-v1** ticket preserves the declared-model coupling removed from #700's manual-v1 scope under [its author's delivery disposition](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190367). It is related to and blocked by #700, never its v1 child or an FM v1 prerequisite.

Design authority: [Grace's 2026-10-04 disposition](https://github.com/neomjs/neo-agent-brain/issues/700#issuecomment-5983735001), [Clio's read-back condition](https://github.com/neomjs/neo-agent-brain/issues/700#issuecomment-5983733683), and the accepted coupling/activation paragraphs in #700. A declared model can confirm family only through the admitted effective transition; a saved field alone cannot.

## The Problem

Once manual family admission exists, a supported explicit operator model declaration can otherwise require a second redundant family action. Automatically promoting a pending declaration on save would misclassify the still-running session. Starting alone also does not establish that the requested model actually became effective.

## The Architectural Reality

`ai/services/fleet/seatModelDeclaration.mjs` validates launch intent and `seatModelCatalog.mjs` reads supported harness catalogs; neither grants canonical family-write authority. #862 and Institution #559 own declared launch intent and its Detail presentation. The manual admitted writer and complete temporal readers are delivered by #700's source leaves; this work must use them, not a parallel era store.

Before implementation, name and falsify the authoritative model→family mapping and the actual seat/profile/session-bound first-turn read-back source for each supported path. An arbitrary runtime report, harness name or catalog label is not that proof. Unsupported paths keep the manual action.

## The Fix

After manual v1 is accepted, connect an explicit operator model declaration to the same admitted publisher only when (1) a fresh operatesSeat check authorizes it, (2) a named authoritative catalog/binding resolves the exact family, (3) the admitted next Start applies that declaration and (4) a first-turn read-back bound to that seat/profile/session confirms the effective model.

Persist and reconcile the paired transition so retries or restart cannot publish twice or reuse a predecessor session's read-back. Admit a prospective transition with its provenance/effective time through the existing typed authority; retain per-seat membership, roster precedence, temporal budget/readiness rules and correction separation. Display pending intent separately from effective model and admitted family.

## Contract Ledger

| Surface | Authority | Behavior / fallback | Docs / evidence |
|---|---|---|---|
| Model→family coupling | Explicit operator act + named authoritative mapping | Supported unambiguous declaration may couple; custom/ambiguous/imported/default/report-only paths remain manual | Mapping provenance and negative controls |
| Effective transition | Admitted Start application + bound first-turn read-back | Both halves required; failure or unavailable proof publishes no family change | Real source binding and interruption controls |
| Publication/history | #700 admitted writer and temporal projection | Prospective, idempotent per-seat act with retained provenance; no second ledger | Writer integration and historical review tests |
| Pending/effective presentation | Canonical intent and observed/admitted facts | Saved choice never presented as the running family | Existing Detail consumer tests |

## Acceptance Criteria

- [ ] AC-1: a source/authority ledger identifies each supported mapping and first-turn read-back producer, exact binding fields and negative controls before code begins. Unsupported/ambiguous paths retain explicit manual confirmation.
- [ ] AC-2: an explicit authorized model declaration can couple only through the plane-admitted writer with a fresh operatesSeat check; generic save, imported value, harness default, runtime self-report and caller-supplied principal do not activate it.
- [ ] AC-3: a mid-session save retains the old admitted family. Failed, refused, canceled or never-performed Start and missing/mismatched read-back leave it unchanged. A genuine admitted Start plus matching first-turn proof enables the prospective transition.
- [ ] AC-4: stale seat/profile/session, remove/recreate, ownership transfer, delayed reply, restart and repeated delivery cannot activate the wrong incarnation or duplicate a transition. Pending and effective facts stay separate in readback.
- [ ] AC-5: submitted reviews retain their historical family; corrected author interval union, literal unknown versus unclassified, multi-seat unanimity and roster precedence retain #700's rules. This is not retroactive correction.
- [ ] AC-6: actual writer/lifecycle/read-back integration tests cover both halves and their interruption; document the supported profiles and explicit manual fallback. Any installed acceptance is separately recorded on a named candidate, never inferred from unit tests.

## Out of Scope / Avoided Traps

No changes to manual FM v1 acceptance, no family inference from a harness or login, no automatic activation merely behind a flag, no new credential class, no altered family vocabulary and no second era ledger. Claude Desktop's unsupported model-declaration path remains manual unless separately qualified.

## Decision Record impact

Aligned with ADR0038/0041 and the admitted manual source; no new authority is presumed. If the mapping or effective-transition mechanism requires a new authority boundary, return that delta to design before implementation.

## Ownership and timing

Sophie retains the existing #700 activation obligation. Parked after v1 by the explicit delivery decision; review again after #700 closes or the operator changes priority. Do not start it ahead of the manual outcome.

Related: #700, #862, neomjs/neo-agent-institution#559, neomjs/neo-agent-institution#811

Creation sweep: latest 25 created-descending open Brain issues, own assignments and latest 30 all-state A2A checked immediately before filing. #700 currently holds this scope and its author explicitly requested extraction; #862 is launch intent without family authority. MC prior-art sweep and exact 2026-10-04 dispositions retain the existing policy rather than introduce another one. Brain structure-map passed; existing Fleet/source-reader owners retain placement.

Origin Session ID: 5032dade-507a-4d14-ac0f-15d78670a93c
Sophie (GPT-6 Astra · ultra, Codex Desktop)
Retrieval Hint: #700 October4 declared model coupling pending Start bound first-turn readback post-v1


## Timeline

- 2026-10-26T08:34:57Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-26T08:34:59Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-26T08:34:59Z @neo-gpt-sophie added the `no auto close` label
- 2026-10-26T08:34:59Z @neo-gpt-sophie added the `ai` label
- 2026-10-26T08:35:00Z @neo-gpt-sophie added the `architecture` label
- 2026-10-26T08:35:00Z @neo-gpt-sophie added the `agent-os` label
- 2026-10-26T08:35:00Z @neo-gpt-sophie marked this issue as being blocked by #700

