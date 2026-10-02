---
id: 435
title: 'The Agent Detail''s Repository pane reads the roster row, and every pane names its missing producer'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-02T11:39:01Z'
updatedAt: '2026-10-02T12:47:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/435'
author: neo-fable
commentsCount: 0
parentIssue: 391
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-02T12:44:47Z'
---
# The Agent Detail's Repository pane reads the roster row, and every pane names its missing producer

## Context

The delivered slice of #391, split out at its PR's review (Euclid's RA-1 on PR #419, 2026-10-02): #391's body still asks for three live panes (thought stream, current lane, pull requests) whose producers are leaves of their own (#418 for the lane, neomjs/neo-agent-brain#741 for the peer-turn read, neomjs/neo#19122's projection for the pull requests), so a PR that resolves #391 would close undelivered work. This leaf is what the PR delivers and nothing more; #391 stays open as the tracker for the three panes.

## The Problem

On the installed Fleet Manager every Status pane of a Fleet-booted seat reads `not observed — source not wired`, the Repository pane included — although the header's `repository` axis three lines above renders the roster row's `sources.repoStatus` fact as `wired · observed`. One record, two truths; and no pane says which producer it waits for.

## The Architectural Reality

- `apps/agentos/view/fleet/detail/Container.mjs` classifies each pane from `paneLedgers` (`classifyPaneFreshness` → `describePaneFreshness`); nothing in `apps/` writes a ledger.
- The header's `renderStateLedger` reads `SourceHealth.normalizeFleetSources(record.sources).repoStatus`; the roster row carries `repoSlug` and `repoPath` (`util/RosterRow.mjs`).
- The liveness owner admits every roster through `FleetAdmission.admitRoster` and re-seats the selection there; the cockpit pushes the live pane through `afterSetDetailRecord` and seeds a pane projected later (a vessel, a dock rematerialization) from owner-held state.
- ADR 0041's anti-anchor: an observation with its age, never a stored status.

## The Fix

1. The pane derives the Repository pane's pill from the roster row's `repoStatus` fact — the same descriptor the header renders — and ages it from the roster admission instant; a non-observing fact shows `<state> — <reason>` in the roster's words; a row that carried no repository fact says so; the body shows the slug and the clone path.
2. The liveness owner stamps `rosterObservedAt` at every admission, before the reconcile, so a re-seat it causes carries the record and the clock in one pane write; an unchanged record gets the clock alone; a pane projected later is seeded with the owner's instant.
3. The three panes without a producer keep the honest unobserved pill and name the producer they wait for on the pill's title.
4. A long reason elides at rail width with the full words on the title (the head's `min-width: 0`).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `AgentDetail.rosterObservedAt_` (new reactive config) | `FleetAdmission.admitRoster` (the one admission path) | The reader's clock (epoch ms) at the last roster admission; the repository pane's observation instant. Written with the record in one `set()` on a re-seat, alone on an unchanged record, seeded into a projected pane from the owner. | `null` (a bare mount, no roster landed) → the pane states `unobserved — no roster read has landed in this mount`. | JSDoc on the config and the hook | `detail/container.spec.mjs`, `rosterStore.spec.mjs` (the paired write) |
| the Repository pane's pill + body | `renderStateLedger`'s `sources.repoStatus` fact | `wired` + a stamp → `updated <age>`; any other state → `<state> — <reason>` with the roster's reason, or `the roster row carried no repository fact` when the row carried none; body = `repoSlug` (or `no repository declared`) + `repoPath`. | An explicit `paneLedgers.repo` entry outranks the derived source (a feed stamping the pane directly). | JSDoc | `detail/container.spec.mjs`, `FleetCockpitDrillNL` |
| the other three panes' pills | #391's producer table | `not observed — source not wired` with the awaiting producer on the title (the policy-aware peer-turn read · the roster row's lane stamp · the per-seat open-work projection). | — | JSDoc (`AWAITING`) | `detail/container.spec.mjs`, `FleetCockpitDrillNL` |

## Acceptance Criteria

- [x] AC-1 On the fixture landing, a landed roster carrying the resident's repository fact dates the Repository pill and shows the slug and path, while the header's `repository` axis reads the same fact; the sample roster without the fact reads `not wired — the roster row carried no repository fact` (e2e `FleetCockpitDrillNL`).
- [x] AC-2 The first admission that re-seats the selection writes the record and the clock in one pane `set()`; an unchanged record receives the clock alone; a projected pane is seeded with the owner's instant (unit).
- [x] AC-3 Every producer-less pane names its producer on the pill's title; no pane shows the bare label without one (unit + e2e).
- [x] AC-4 The freshness contract is unchanged: fresh → stale at the TTL → lost at four TTLs; an explicit ledger outranks the derived source (unit).
- [x] AC-5 `pane-agent-detail.png` and `cockpit-review-1280.png` re-captured; the visual stamp re-written; the full visual run green.

## Out of Scope

The three live panes and their producers (#391 tracks them: #418, neomjs/neo-agent-brain#741, neomjs/neo#19122).

## Related

#391 (the tracker this slice came from) · #9 (parent) · #418 · neomjs/neo-agent-brain#741 · neomjs/neo#19122 · PR #419 (delivers this leaf)

Decision Record impact: aligned-with ADR 0041's anti-anchor.

Sweeps: live latest-open sweep, the latest 20 open Institution issues at 2026-10-02T11:38Z (newest #431), no equivalent (#391 is the tracker, not the slice) · A2A in-flight sweep at 11:35Z, no claim on the detail panes besides mine · own-assignment sweep: #391, #384, #418.

Origin Session ID: 774647be-7f3e-4a83-a197-0f7d1f7cef1a
Retrieval Hint: "Agent Detail repository pane roster row rosterObservedAt paired write awaiting producer title"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 774647be-7f3e-4a83-a197-0f7d1f7cef1a


## Timeline

- 2026-10-02T11:39:03Z @neo-fable added the `enhancement` label
- 2026-10-02T11:39:03Z @neo-fable added the `agent-os` label
- 2026-10-02T11:39:04Z @neo-fable added the `ai` label
- 2026-10-02T11:44:05Z @neo-fable assigned to @neo-fable
- 2026-10-02T11:44:09Z @neo-fable added parent issue #391
- 2026-10-02T11:44:24Z @neo-fable referenced in commit `7104948` - "fix(agentos): the admission writes the record and the roster clock in one pane set (#435)

The liveness owner stamps rosterObservedAt before the reconcile, so a re-seat it causes carries the record and the clock in one pane write; an unchanged record receives the clock alone; the projected pane's seed is covered. The PR's delivered slice of #391 is #435."
- 2026-10-02T11:44:48Z @neo-fable cross-referenced by PR #419
- 2026-10-02T11:44:53Z @neo-fable cross-referenced by #391
- 2026-10-02T11:50:52Z @neo-fable referenced in commit `4727987` - "chore(visual): re-stamp the baseline inputs after the paired-write commit (#435)

FleetCockpitVisual 26/26 against the unchanged goldens; the stamp follows detail/Container.mjs's JSDoc change."
- 2026-10-02T12:44:48Z @tobiu closed this issue
- 2026-10-02T12:44:48Z @tobiu referenced in commit `7cd284b` - "feat(agentos): the Agent Detail panes state their sources, the Repository pane live from the roster row (#435) (#419)

* feat(agentos): the Agent Detail panes state their sources, the Repository pane live from the roster row (#391)

The four Status panes no longer share one bare 'not observed — source not wired' pill. The Repository pane reads the roster row's repoStatus fact (the same descriptor the header's repository axis renders) and ages from the roster admission instant, which the liveness owner stamps on every admitted roster and seeds into a pane projected later; a roster fact that did not observe shows its state and reason in the roster's words. The three panes without a producer on this plane keep the honest pill and name the producer they wait for on its title. A long reason elides at rail width with the full words on the title (the head's min-width breaks the min-content chain). Goldens re-captured and stamped.

* fix(agentos): the admission writes the record and the roster clock in one pane set (#435)

The liveness owner stamps rosterObservedAt before the reconcile, so a re-seat it causes carries the record and the clock in one pane write; an unchanged record receives the clock alone; the projected pane's seed is covered. The PR's delivered slice of #391 is #435.

* chore(visual): re-stamp the baseline inputs after the paired-write commit (#435)

FleetCockpitVisual 26/26 against the unchanged goldens; the stamp follows detail/Container.mjs's JSDoc change."

