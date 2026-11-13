---
id: 1665
title: The Fleet retires harness generations nothing holds
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T02:30:10Z'
updatedAt: '2026-11-13T02:30:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1665'
author: neo-opus-ada
commentsCount: 0
parentIssue: 1531
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
# The Fleet retires harness generations nothing holds

## Context

The operator, 2026-11-13: *"since updates arrive multiple times a week, we need a permanent solution to handle it."* With #1658 and #1662 every GUI family acquires its own builds, and nothing ever removes one. Measured on the operator's Mac, 2026-11-13 ~02:28Z: the Antigravity store already holds three admitted builds of 491 MB each (2.31.0, 2.31.1, 2.31.2, admitted within 27 hours), and the data volume is 93% full with 134 GiB free. A Codex `ChatGPT.app` is about 1.9 GB and a Claude build several hundred MB, each shipping more than once a week, so the store grows by several GB a week until the disk is full.

## The Problem

#1531 point 3 says the Fleet "retires only a generation that no pin or rollback target holds", but no code retires anything: `harnessGeneration.mjs` admits, lists and selects, and a copy stays forever. A permanent update loop with no retirement is a disk-full incident on a schedule.

## The Architectural Reality

- The store: `<root>/<family>/ledger.json` plus one copy per admitted build (`harnessGeneration.mjs`). `listHarnessGenerations` lists admitted entries whose copy is present, newest first; `selectHarnessGeneration` takes the first. The ledger keeps extra fields on its entries.
- A seat's pin: a Start launches the selected generation and records `harnessGeneration` on the seat's record and in its lease (`FleetLifecycleService#writeSeatLease`, `harnessGenerationId`). A fresh Fleet server re-adopts a seat from its lease (`adoptLeasedSeat`); a lease naming a generation whose copy is gone is void, so deleting a held copy would orphan a running seat.
- ADR 0034 §2.5 item 5 (Institution) sets the rule for runtime generations, and #1531 extends it to vendor harness builds: the installed package and the rollback target keep theirs, a surviving seat's pin keeps its own, unknown ownership or liveness keeps the generation and names the uncertainty, retirement must exclude a concurrent admission or Start taking a new reference, and the seat lifecycle owns the decision rather than a second counter.
- The acquisition job (`wireFleetHarnessAcquisition.mjs`) already runs once per family at a time, so a retirement run inside it never overlaps that family's admission.

## The Fix

1. **Store.** `retireHarnessGenerations({root, family, held, now})` in `harnessGeneration.mjs`: it keeps the newest two present generations (the selected one and its rollback) and every id in `held`, removes every other copy, and stamps `retiredAt` on those ledger entries, so the ledger keeps the history and a retired build is never fetched again.
2. **Pins.** `FleetLifecycleService#retireHarnessGenerations(harnessType)` collects the held ids from the seat lifecycle: every seat record's `harnessGeneration` and every lease of the family's seats on disk. A lease that cannot be read (anything but absent) makes the run keep everything and name why.
3. **Trigger.** The acquisition job retires after each family's run, inside the same per-family turn, and records the result in that family's status.

## Acceptance Criteria

- [ ] With five admitted generations, no seat pinned and no lease, a run removes the three oldest copies, stamps their ledger entries, and keeps the selected one and its rollback (unit).
- [ ] A generation a seat record or a lease on disk names is kept, whatever its age (unit).
- [ ] A lease that cannot be read keeps every generation, and the result names the seat (unit).
- [ ] A retired build's version is never acquired again, and the selection never changes because of a retirement (unit).
- [ ] The acquisition job runs retirement after each family's run and shows the retired ids in its status (unit).
- [ ] **Post-merge** `[L4-deferred — operator handoff needed]` (an FM build carrying this): after the first pass, the Antigravity store keeps 2.31.2 and 2.31.1 plus any build a running seat is pinned to, and 2.31.0's copy is gone with `retiredAt` on its ledger entry.

## Out of Scope

- A retention setting. Two plus the held ones is the rule until a measured need says otherwise.
- The cockpit surface (#1531 point 7).
- Fleet runtime generations (ADR 0034 §2.5), which have their own lifecycle.

## Avoided Traps

- **A reference counter.** The pins already live in seat records and leases; a second count drifts (ADR 0034 §2.5 item 5).
- **Retiring by age alone.** A seat that has run since an older build keeps it until its next Start.
- **Removing the ledger entry.** Then the acquisition would fetch a retired build again as "not admitted".

## Related

#1531 (parent, point 3) · #1658 and #1662 (the families that now acquire) · #1543 (the pin and the lease) · #1590 (Antigravity's opt-in) · neomjs/neo-agent-institution#adr:0034-electron-shell-architecture §2.5

Decision Record impact: aligned-with `neomjs/neo-agent-institution#adr:0034-electron-shell-architecture` §2.5 item 5.

Not a `good first issue`: it is owned, and its post-merge receipt needs the operator's Mac.

Live latest-open + closed sweep: latest open, plus a search for `retire generation OR prune harness OR generation store disk` across open and closed at 2026-11-13T02:29Z; no equivalent found (#1392 is about a bridge response during runtime-generation retirement). Backlog band: 157 open (red), so this folds under #1531. A2A claim sweep: latest 30 messages, all read states; no claim on harness retirement. MC sweep: "harness generation store grows forever; old vendor app copies never deleted; disk fills", 5 results (earlier disk-hygiene reports, Oct 22 and Nov 1); no prior decision on harness generations. Own-assignment sweep: #1531 (the parent), #571, #133 and #1508; none overlapping.

Origin Session ID: 776d12fd-202a-486b-88c0-41605d050556

Retrieval Hint: "harness generation store grows forever; retire copies no seat pin or rollback holds"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T02:30:10Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T02:30:11Z @neo-opus-ada added the `enhancement` label
- 2026-11-13T02:30:11Z @neo-opus-ada added the `ai` label
- 2026-11-13T02:30:12Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T02:30:17Z @neo-opus-ada added parent issue #1531
- 2026-11-13T02:35:36Z @neo-opus-ada cross-referenced by PR #1666
- 2026-11-13T02:44:06Z @neo-opus-ada cross-referenced by #1531

