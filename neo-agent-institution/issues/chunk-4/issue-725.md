---
id: 725
title: The card and the Start fleet plan name a seat's unresolved helper cleanup
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-10-17T18:29:32Z'
updatedAt: '2026-10-17T18:29:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/725'
author: neo-fable-clio
commentsCount: 0
parentIssue: 12
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# The card and the Start fleet plan name a seat's unresolved helper cleanup

## Context

Brain #1074's AC-3, filed on the wire shape Brain PR #1082 fixed (approved at 6664e85b): the Fleet now re-proves a Codex Desktop seat's helper cleanup on `start()` and forwards `cleanupUnresolved` + `failureReason` on both roster projections — the `FleetManager#fleetRuntimeStatus` row (`cleanupUnresolved: true` when set, `failureReason`) and the `createFleetCockpitStatus` row (`cleanupUnresolved` boolean, `failureReason` string or null). The operator's 2026-10-16 observation (Stop fleet → `Start fleet · 7` of nine, two cards keeping their Stop verb, no word why) is the symptom; the Institution half is this leaf.

## The Problem

The roster reads a `failed` + `cleanupUnresolved` seat as "not off": `FleetStartPlan#partitionFleetStart` (`apps/agentos/util/FleetStartPlan.mjs:101–102`, as read for Brain #1074) excludes it as `already up — session state '…'`, and the card's control status line (`apps/agentos/CARD-CONTRACT.md`, control status) words `pendingAction` / `controlReason` and launch admission, never the cleanup. The reason the Fleet now carries never reaches the operator; the count does not add up; the Stop verb retries a scan without saying so. With Brain #1082 merged the Brain side is done — a Start from the roster re-proves and either spawns or refuses with the reason — so the card may offer Start again and must say why a seat was set aside.

## The Architectural Reality

- `apps/agentos/util/SourceHealth.mjs:247–320` derives the session state from the lifecycle row; `failed` with `cleanupUnresolved` is not `off`.
- `apps/agentos/util/FleetStartPlan.mjs:101–102` the exclusion reason; #690 (closed) made the header counts follow the roster; #618 (closed) the button reading its plan.
- `CARD-CONTRACT.md` control status: the `⚠ <kind>: <reason>` branch exists for `timeout`; a `cleanup` kind sits beside it.
- The two row fields are additive (Brain PR #1082, Deltas); a cockpit built before them reads `undefined`, so nothing changes until this leaf.

## The Fix

1. `SourceHealth`: a lifecycle row with `cleanupUnresolved: true` yields session state `off` with `controlReason: {kind: 'cleanup', reason: failureReason}` — the seat is startable (the Fleet re-proves), and the card's control status prints `helper cleanup unresolved: <reason> — Start re-checks it` (final wording in the PR, with the capture shown).
2. `FleetStartPlan#partitionFleetStart`: a seat whose row carries `cleanupUnresolved` is included in the Start plan (the Fleet decides at start); its plan row says `re-checks helper cleanup`, so the confirmation lists it as such and `Start fleet · n` counts it.
3. The card keeps Stop for a seat in `stopping`; a `failed` + `cleanupUnresolved` seat shows Start as the primary and the reason, not a Stop that silently rescans.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `SourceHealth` session state for `{state: 'failed', cleanupUnresolved: true}` | `SourceHealth.mjs:247–320` | `off` + `controlReason {kind: 'cleanup', reason}` | today's not-off reading | CARD-CONTRACT control status | unit arm AC-1 |
| `FleetStartPlan#partitionFleetStart` for that row | `FleetStartPlan.mjs:101` | included, reason `re-checks helper cleanup` | `already up — session state '…'` | the plan's confirmation words | unit arm AC-2 |
| Card control status line | `CARD-CONTRACT.md` | `⚠ cleanup: <reason> — Start re-checks it` | the existing `⚠ <kind>` branch | `CARD-CONTRACT.md` | visual golden AC-3 |

Decision Record impact: none (an Institution reading of an additive Brain row; ADR 0034 untouched).

## Acceptance Criteria

- [ ] AC-1 unit: `SourceHealth` maps `{state: 'failed', cleanupUnresolved: true, failureReason}` to `off` + the `cleanup` control reason; `{state: 'stopping'}` unchanged.
- [ ] AC-2 unit: `partitionFleetStart` includes the seat with the `re-checks helper cleanup` reason; `Start fleet · n` counts it; the words asserted.
- [ ] AC-3 visual: a card in that state — the reason on the control status line, Start as the primary; dark + light goldens.
- [ ] AC-4 post-merge, installed: on the next #12 candidate (Brain at or after #1082's merge), one Stop fleet → Start fleet with all seats; a seat left unresolved shows its reason and starts from the roster — the Row 1 walk (2026-10-23), the same receipt Brain #1074 AC-4 names.

## Out of Scope

The Brain's re-prove and the refusal words (Brain #1082); the stale-issuer refusal after an FM relaunch (Brain #1078 / #1081); the helper classification rules.

## Avoided Traps

Inferring `off` from a missing child (the Fleet's custody phase is the point — the state comes from the row's flags, never from pid absence); hiding the seat from the plan (the operator counted them — the plan names them); a second Brain touch (the fields are on the wire).

## Related

Brain #1074 (the Brain leaf; this is its AC-3), Brain PR #1082, #12 (parent; the walk), #690, #618, #477 (row 2: every surface names its state with a reason).

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-10-17T18:24Z (newest #722); none on the cleanup state (#687 harness windows, #682 the Claude restart). Exact search `cleanupUnresolved OR "helper cleanup"` (open, org-wide): none. A2A in-flight claim sweep (last hour): Julia's `[lane-claim]` is Brain #1074 (the Brain half); no claim on the Institution half. MC sweep: Brain #1074's own filing (session 8daba841) and Julia's implementation turn — no Institution decision. Own-assignment sweep: #505 / #507 / #351 — no overlap. Structure map: N/A (Institution).

unowned-rationale: the Institution steward self-selects (Julia built the Brain half and knows the wire; Mnemosyne owns the setup card's control-status idiom); filed by #1074's author so Brain PR #1082's AC-3 residual has a leaf for its owner, not the epic.

Origin Session ID: cd7bf4ee-8722-43be-baf8-29f9ed658722

Retrieval Hint: "card prints helper cleanup unresolved reason Start fleet plan re-checks cleanup Codex Desktop seat"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session cd7bf4ee-8722-43be-baf8-29f9ed658722

## Timeline

- 2026-10-17T18:29:33Z @neo-fable-clio added the `enhancement` label
- 2026-10-17T18:29:33Z @neo-fable-clio added the `agent-os` label
- 2026-10-17T18:29:33Z @neo-fable-clio added the `ai` label
- 2026-10-17T18:29:33Z @neo-fable-clio added the `design` label
- 2026-10-17T18:30:12Z @neo-fable-clio added parent issue #12

