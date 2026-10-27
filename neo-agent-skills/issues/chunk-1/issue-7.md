---
id: 7
title: 'pr-review premise row: graduated-decision reversal inside a PR'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-opus-vega
createdAt: '2026-07-31T04:20:27Z'
updatedAt: '2026-09-15T15:58:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/7'
author: neo-opus-vega
commentsCount: 0
parentIssue: 16212
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-15T15:58:01Z'
---
# pr-review premise row: graduated-decision reversal inside a PR

## Context

Sub of neomjs/neo#16212 (execution-fidelity epic, operator-commissioned 2026-07-31). Empirical anchor: PR `#16188` — nominally a config-default PR, its 28-file surface amended ADR 0014 to elect "No host-edge Orchestrator runs after the cut," reversing the same-day-merged two-role authority (D#15595 ratified; PR `#16173` merged 13:08Z). It was approved 21:15Z and merged 21:16Z; the approving review (mine) validated diff internals and CI and never ran the premise against the graduated authority the diff was reversing.

## The Problem

`pr-review`'s Cycle-1 Premise Pre-Flight already asks *"Upstream not graduated?"* — whether the PR **depends on** an ungraduated artifact. It does not ask the inverse: whether the diff **amends or reverses** an already-graduated decision without a reopened graduation. That is exactly the neomjs/neo#16188 class: the upstream (#16167) was legitimate, so the existing row passed, while the ADR hunk inside the implementation diff carried an unauthorized election change. The consensus-mandate makes such a change require reopened authority; review is the last line where a violation is catchable before merge, and the checklist had no row for it.

## The Architectural Reality

- `.agents/skills/pr-review/audits/cycle-1-premise-preflight.md` — the premise table (the "Upstream not graduated" row verified at line 18, 2026-07-31); its verdict semantics: Cycle-1 premise defects take a single Drop+Supersede/block verdict, not iterative Required Actions.
- AGENTS.md consensus-mandate — graduated-decision changes require reopened graduation with family-keyed quorum.
- `learn/agentos/decisions/` — the ADR corpus whose elections are the protected class; ADR bodies carry their own revalidation-trigger clauses (ADR 0014's own clause says changes of this class "reopen this election").
- Manifest byte budgets on skill payloads.

## The Fix

Add one row to the Cycle-1 premise table:

> **Graduated-decision reversal** — does the diff amend or reverse a graduated decision (an ADR election, a Discussion-ratified target, a merged authority model) without a reopened graduation / consensus record? Signals: `learn/agentos/decisions/` hunks inside an implementation PR; diff statements contradicting a `[GRADUATED_TO_TICKET]`-backed body; an ADR's own revalidation-trigger firing. Verdict class: premise (Drop+Supersede or block pending reopened authority), never an iterative RA.

Record `#16188` as the empirical anchor alongside the audit's existing `#11083` anchor.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `cycle-1-premise-preflight.md` table | neomjs/neo#16212 + AGENTS.md consensus-mandate | one added row with signals + verdict class + `#16188` anchor | ADR edits WITH reopened authority pass the row explicitly | the audit file | row text + anchor paragraph |

## Decision Record impact

`none` — review-procedure amendment; protects ADR authority, amends none.

## Acceptance Criteria

- [ ] The row exists with the three signal classes and the premise-verdict (non-iterative) semantics.
- [ ] `#16188` recorded as anchor with the one-minute approve→merge chronology.
- [ ] Explicit legality note: ADR/Discussion changes carrying a reopened graduation / consensus record PASS the row — this is not an ADR-edit ban.
- [ ] Net-byte discipline on the audit file; manifest gates green.
- [ ] `turn-memory-pre-flight` consulted in the PR.

## Out of Scope

- Template/validator changes (`validate_pr_review_body`) — the audit file is reviewer guidance; mechanical validation is a follow-up decision if the row proves load-bearing.
- Retroactive review of merged PRs (#16188's disposition belongs to the neomjs/neo-agent-brain#84 compliance map).

## Avoided Traps

- **A blanket ADR-edit ban.** Elections are amendable — through reopened authority. The row checks for the *record*, not the change.
- **Iterative-RA framing.** A premise defect answered with five RAs is the exact neomjs/neo#11083 anti-pattern this audit file exists to prevent; the new row inherits its verdict semantics.

## Related

- Parent: neomjs/neo#16212. Sequencing: execution gated behind neomjs/neo-agent-brain#84 stabilization (see parent).
- PR `#16188` / D#15595 / PR `#16173` — the anchor chain. `#11083` — the audit's original anchor.

Live latest-open sweep + A2A claim sweep recorded on parent neomjs/neo#16212 at filing time (04:14Z, none competing).

Origin Session ID: dd39c5c1-8773-4c9d-a3d4-664f9fb0f952

Retrieval Hint: `cycle-1 premise row graduated decision reversal ADR election inside implementation PR reopened graduation consensus record drop supersede`

## Timeline

- 2026-07-31T04:20:29Z @neo-opus-vega added the `enhancement` label
- 2026-07-31T04:20:29Z @neo-opus-vega added the `ai` label
- 2026-07-31T04:20:29Z @neo-opus-vega added the `model-experience` label
- 2026-07-31T04:20:42Z @neo-opus-vega added parent issue #16212
- 2026-07-31T04:29:00Z @neo-gpt-emmy cross-referenced by #16212
- 2026-08-27T11:33:46Z @neo-gpt-emmy added parent issue #16212
- 2026-08-27T11:34:49Z @neo-gpt-emmy cross-referenced by #17784
- 2026-08-28T22:06:20Z @neo-gpt-euclid cross-referenced by #13
- 2026-09-15T15:22:07Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-15T15:25:38Z @neo-opus-vega cross-referenced by PR #73
- 2026-09-15T15:45:54Z @neo-opus-vega referenced in commit `d35c09d` - "feat(pr-review): the premise pre-flight asks the inverse question, so a diff cannot quietly reverse what was ratified (#7)

The Cycle-1 trigger catalog asked whether a PR DEPENDS on something ungraduated.
It never asked whether a PR REVERSES something already graduated — an ADR election,
a Discussion-ratified target, a merged authority model. The second is the more
expensive direction: the upstream is legitimate, CI is green, and the diff reads as
implementation while it amends the authority that governs it.

The new row names three signals a reviewer can see without knowing the history —
decision-record hunks inside an implementation PR, statements contradicting a
graduated body, an ADR's own revalidation trigger firing — and inherits the
catalog's existing verdict semantics, so it resolves as a premise rather than as
another iteration list.

It checks for the RECORD, not the change: an amendment carrying a reopened
graduation passes. Elections are amendable through reopened authority, and a row
that forbade ADR edits outright would trade one failure mode for a worse one.

The source-ticket row loses its duplicated five-item enumeration, which the
Source-Ticket Currency Rule section already carries in full. A table row holds the
question; the section holds the procedure.

Placement per turn-memory-pre-flight: not universal, governs one lifecycle event,
so Skill. Verified doubly conditional — this audit is reached only from
pr-review-guide.md, itself loaded only on skill invocation — and check-substrate-size
reports no always-loaded entry points in this repository, so the added bytes sit on
no per-turn surface.

[skill-growth-justified: a premise row cannot be compressed to a trigger — it IS the
trigger, and a reviewer who has not read it cannot apply it. The bytes buy refusal of
a merge class that silently reverses ratified authority, anchored on a PR approved and
merged one minute apart by a review that checked diff internals and never the premise.
They land on zero always-loaded surface: the audit is reachable only from
pr-review-guide.md, itself loaded only on skill invocation, and check-substrate-size
finds no substrate entry points in this repository. Retirement trigger is already
written into #7's Out of Scope — if the row proves load-bearing it graduates to
mechanical validation in validate_pr_review_body and the guidance row retires into the
validator. Partially paid for by deduplicating the source-ticket row against the
Source-Ticket Currency Rule section.]"
- 2026-09-15T15:48:07Z @neo-opus-vega referenced in commit `ab67369` - "feat(pr-review): the premise pre-flight asks the inverse question, so a diff cannot quietly reverse what was ratified (#7)

The Cycle-1 trigger catalog asked whether a PR DEPENDS on something ungraduated.
It never asked whether a PR REVERSES something already graduated — an ADR election,
a Discussion-ratified target, a merged authority model. The second is the more
expensive direction: the upstream is legitimate, CI is green, and the diff reads as
implementation while it amends the authority that governs it.

The new row names three signals a reviewer can see without knowing the history —
decision-record hunks inside an implementation PR, statements contradicting a
graduated body, an ADR's own revalidation trigger firing — and inherits the
catalog's existing verdict semantics, so it resolves as a premise rather than as
another iteration list.

It checks for the RECORD, not the change: an amendment carrying a reopened
graduation passes. Elections are amendable through reopened authority, and a row
that forbade ADR edits outright would trade one failure mode for a worse one.

The source-ticket row loses its duplicated five-item enumeration, which the
Source-Ticket Currency Rule section already carries in full. A table row holds the
question; the section holds the procedure.

Placement per turn-memory-pre-flight: not universal, governs one lifecycle event,
so Skill. Verified doubly conditional — this audit is reached only from
pr-review-guide.md, itself loaded only on skill invocation — and check-substrate-size
reports no always-loaded entry points in this repository, so the added bytes sit on
no per-turn surface.

[skill-growth-justified: a premise row IS the trigger, so it cannot be compressed to one; it buys refusal of a merge class that silently reverses ratified authority, lands on zero always-loaded surface, is partially paid for by deduplicating the source-ticket row, and retires into validate_pr_review_body per #7's Out of Scope if it proves load-bearing]"
- 2026-09-15T15:52:10Z @neo-opus-vega referenced in commit `d0b5fc6` - "chore(release): 0.1.7 carries the graduated-decision-reversal premise row (#7)

Skill changes bump this package's version (README §Consuming), so consumers pick up
the new Cycle-1 trigger the way they pick up any dependency change. Without the bump
the row would sit on `dev` and reach no reviewer's loaded substrate."
- 2026-09-15T15:55:51Z @neo-opus-vega referenced in commit `4e2bcb2` - "chore(release): 0.1.7 carries the graduated-decision-reversal premise row, with the three workflow pins moving (#7)

Skill changes bump this package's version (README §33), so consumers pick up the new
Cycle-1 trigger the way they pick up any dependency change. Without the bump the row
sits on `dev` and reaches no reviewer's loaded substrate — a merged skill PR nobody
loads.

The bump is not only `package.json`. `reusable-pr-baseline.yml` pins the version three
times, once per guard it installs immutably, and `test-reusable-pr-baseline.mjs`
asserts all three against the manifest — it names the drifts individually as package,
substrate and PR-body version drift. A bump that moves one and not the others is the
failure that test exists to catch, and it caught mine."
- 2026-09-15T15:58:01Z @tobiu referenced in commit `9ed944a` - "Merge pull request #73 from neomjs/agent/7-graduated-reversal-premise-row

The premise pre-flight asks the inverse question, so a diff cannot quietly reverse what was ratified (#7)"
- 2026-09-15T15:58:01Z @tobiu closed this issue

