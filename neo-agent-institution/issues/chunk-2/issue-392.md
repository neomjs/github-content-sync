---
id: 392
title: Claude Desktop seat cards name the clone path to open in the Code tab
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-01T14:22:52Z'
updatedAt: '2026-10-01T16:45:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/392'
author: neo-fable
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T16:45:23Z'
milestone: FM v1
---
# Claude Desktop seat cards name the clone path to open in the Code tab

## Context

The Institution half of Brain #669's Fix 3 / AC-5, taken on @neo-gpt's first refusal (A2A 2026-10-01 13:08Z, accepted 14:19Z): Claude Desktop accepts no folder argument, so a Fleet-launched Claude Desktop seat opens a scratch workspace until the operator opens the seat's clone in the Code tab by hand (Ada's witness turn, 2026-10-01; Brain #669's Context). The fact the operator needs — the clone path — already exists on the roster row and is dropped in the cockpit.

## The Problem

`apps/agentos/view/fleet/cockpit/LivenessController.mjs:551` `mapRosterRow` maps the cockpit DTO row onto the `FleetAgent` record and omits two facts the Brain already serves: `row.harnessType` and `row.repoStatus.repoPath` (with `repoSlug`), both assembled in `ai/services/fleet/fleetCockpitStatus.mjs:108-160` from `inspectFleetRepos` (the path is derived before any launch). The `FleetAgent` model (`apps/agentos/model/FleetAgent.mjs`) has no field for either, so no card or pane can name the folder a Claude Desktop seat must open, and the first-launch handoff has nothing to point at.

## The Architectural Reality

- Brain row (unchanged): `fleetCockpitStatus.mjs` rows carry `harnessType` (`:111`) and `repoStatus: {configured, repoSlug, repoPath, exists, isCheckout, state, provisioningAction}` (`:160`, from `inspectFleetRepos.mjs:54-66`).
- Institution: `LivenessController.mapRosterRow` (`:551-579`) — tri-state mapping, producer-foreign fields omitted; `FleetAgent` fields (`:34-157`): identity, display, lifecycle, telltale axes, `sources` — no harness, no repo.
- The roster card (`apps/agentos/view/fleet/roster/card/Container.mjs`) renders display state from the record (`applyRecord`, `:550-630`): engine tag, lane line, avatar; text nodes only for remote data (`:560`). Card anatomy is the card contract (`apps/agentos/CARD-CONTRACT.md`); the card-matrix goldens are stale on dev (#287, Ada).
- No first-launch step exists in the cockpit yet (#12 names it; #384 is the wizard's inline recipe), so the card line is the first-launch handoff for a Claude Desktop seat until one does; the guide correction (OwnAgentTeam.md) rides Euclid's Brain PR for #669.

## The Fix

1. `FleetAgent` gains `harnessType`, `repoSlug`, `repoPath` (tri-state, `null` = unknown); `mapRosterRow` carries them from `row.harnessType` and `row.repoStatus`.
2. The roster card shows, for a seat whose row carries a `repoPath`, one quiet line with the path; for a `claude-desktop` seat the line leads with the verb the operator must perform — open that folder in the Code tab — because the Desktop cannot be launched into it. Text node, never `html`; the line is absent when the path is unknown (no placeholder).
3. Unit arms: `mapRosterRow` carries the three facts (red-first), a Claude Desktop row renders the line with the verb, a Codex row renders the path without it, a row without `repoStatus` renders nothing. Card goldens re-captured only for the fixture rows that gain the line.

## Contract Ledger Matrix

| Target surface | Source of authority | Today | Proposed | Fallback | Docs | Evidence |
|---|---|---|---|---|---|---|
| `AgentOS.model.FleetAgent` fields | the roster DTO (`fleetCockpitStatus.mjs` rows) | no `harnessType` / `repoSlug` / `repoPath` | three tri-state fields | `null` = unknown, never guessed | model JSDoc | AC-1 |
| `LivenessController#mapRosterRow` | the DTO → record mapping contract (`:542-550`) | drops `harnessType` and `repoStatus` | carries the three facts | `null` when the row lacks them | method JSDoc | AC-1 |
| roster card (`roster/card/Container.mjs`) | `CARD-CONTRACT.md` | no repo line | a clone-path line; the Code-tab verb for `claude-desktop` seats | no line without a path | the contract's anatomy note | AC-2, AC-3 |
| card-matrix goldens | #11 harness, #287 state | stale on dev | re-captured for the rows that gain the line | — | — | AC-3 |

## Decision Record impact

`none` (consumer presentation of an existing Brain fact; no ADR touched).

## Acceptance Criteria

- [ ] AC-1 Red-first unit arm: `mapRosterRow` carries `harnessType`, `repoSlug`, `repoPath` from a DTO row and `null` for each when the row lacks them; the existing roster-store arms stay green.
- [ ] AC-2 A `claude-desktop` row with a `repoPath` renders the card line with the Code-tab verb and the path; a `codex-desktop` row renders the path without the verb; a row without `repoStatus` renders no line (unit arms on the card).
- [ ] AC-3 Card goldens: only the fixture rows that gain the line change; the stamp is re-written; #287's stale set is not widened.
- [ ] AC-4 Post-merge, installed receipt: Ada's seat card in the installed Fleet Manager names her clone path (the operator's observation; rides the repackage after #669). This ticket closes with PR #393, so the receipt's surviving owner is Brain #571 (the installed arm of Brain #669 AC-5, which names this ticket), recorded on neomjs/neo-agent-institution#12 like #669's other installed arms (reviewer finding on PR #393, 2026-10-01 15:13Z).

## Out of Scope

A first-launch step or dialog (#12 / #384); the OwnAgentTeam.md Claude section (Euclid's Brain PR for #669); a Brain-side `openedProject` observation (not needed — Euclid's read); Codex seats, which the Fleet launches with `--open-project`.

## Related

- Brain #669 (Fix 3 / AC-5, Euclid), Brain #571 (Ada's epic), Institution #12, #245 (the Accounts/card redesign — Ada, informed), #287 (stale card goldens), #384.

Live latest-open sweep: latest 20 open Institution issues at 2026-10-01 14:18Z — #391 #389 #388 #386 #384 #382 #374 #351 #312 #287 #245 … no equivalent; A2A latest 30 (all read states) at 14:18Z: no competing claim (Euclid's first refusal to me is the origin); Memory Core sweep ("seat card shows the clone path; first-launch handoff; Claude Desktop has no folder argument"): Clio's 09-30/10-01 turns name the gap as Brain #669 Fix 3, no Institution ticket; own-assignment sweep: #9 (epic), #382. Structure map: N/A (no new file).

Origin Session ID: 771833a0-8bd1-4520-8e2c-138d82c15a53
Retrieval Hint: "Claude Desktop seat card clone path Code tab mapRosterRow repoPath"


## Timeline

- 2026-10-01T14:22:52Z @neo-fable assigned to @neo-fable
- 2026-10-01T14:22:54Z @neo-fable added the `enhancement` label
- 2026-10-01T14:22:54Z @neo-fable added the `agent-os` label
- 2026-10-01T14:22:54Z @neo-fable added the `ai` label
- 2026-10-01T14:23:30Z @neo-fable added this to the **FM v1** milestone
- 2026-10-01T14:31:51Z @neo-fable cross-referenced by PR #393
- 2026-10-01T14:44:29Z @neo-fable referenced in commit `9a390ec` - "refactor(agentos): the roster row mapping leaves the liveness controller (#392)

The app-file size bar refused LivenessController.mjs at 1003 lines after the clone-path facts joined mapRosterRow. The mapping is its own responsibility — one pure DTO-to-record function the reconcile path calls — so it moves to AgentOS.util.RosterRow (the SourceHealth sibling shape: a Base class with one static), and the controller keeps a one-line delegate so the reconcile path and its witnesses are unchanged. 1003 → 972 lines; the four warn-band files are untouched."
- 2026-10-01T14:50:11Z @neo-fable referenced in commit `2f46d3f` - "chore(visual): re-stamp the baselines for the roster-row seam (#392)

The visual-baseline stamp records the staged blob of every style-owning input under apps/agentos; the seam move (a new util module plus the controller's delegate) moved two inputs past the stamp written for the first commit. Pixels are unchanged — the goldens stayed as they were — only the input identity is re-recorded."
- 2026-10-01T14:52:40Z @neo-gpt cross-referenced by PR #692
- 2026-10-01T15:16:08Z @neo-gpt cross-referenced by #571
- 2026-10-01T15:32:43Z @neo-fable-clio cross-referenced by #669
- 2026-10-01T16:45:23Z @tobiu referenced in commit `5c992c6` - "feat(agentos): a seat card names the clone path a Claude Desktop seat must open (#392) (#393)

* feat(agentos): a seat card names the clone path a Claude Desktop seat must open (#392)

The roster row has carried the facts since the Fleet derived them before any launch —
`harnessType` and `repoStatus.repoPath` — and the cockpit dropped both in `mapRosterRow`, so no
card could tell the operator which folder a Fleet-launched Claude Desktop seat needs opened in
the Code tab (that Desktop cannot be launched into a folder; the seat opens a scratch workspace
until the clone is opened by hand).

FleetAgent gains `harnessType`, `repoSlug` and `repoPath` as tri-state fields, the mapper carries
them (null when the row lacks them), and the card renders one quiet mono line under the lane for
a reported path: a `claude-desktop` seat's line leads with "Open in the Code tab", every other
harness shows the bare path, no path means no line. Inert text nodes, the title carries the whole
path. The card contract gains the row; the stamp is re-written for the skin input.

* refactor(agentos): the roster row mapping leaves the liveness controller (#392)

The app-file size bar refused LivenessController.mjs at 1003 lines after the clone-path facts joined mapRosterRow. The mapping is its own responsibility — one pure DTO-to-record function the reconcile path calls — so it moves to AgentOS.util.RosterRow (the SourceHealth sibling shape: a Base class with one static), and the controller keeps a one-line delegate so the reconcile path and its witnesses are unchanged. 1003 → 972 lines; the four warn-band files are untouched.

* chore(visual): re-stamp the baselines for the roster-row seam (#392)

The visual-baseline stamp records the staged blob of every style-owning input under apps/agentos; the seam move (a new util module plus the controller's delegate) moved two inputs past the stamp written for the first commit. Pixels are unchanged — the goldens stayed as they were — only the input identity is re-recorded."
- 2026-10-01T16:45:23Z @tobiu closed this issue
- 2026-10-01T18:30:26Z @neo-fable-clio cross-referenced by #351

