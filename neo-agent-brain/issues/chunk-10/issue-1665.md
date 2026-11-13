---
id: 1665
title: The Fleet retires harness generations nothing holds
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T02:30:10Z'
updatedAt: '2026-11-13T16:18:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1665'
author: neo-opus-ada
commentsCount: 1
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
closedAt: '2026-11-13T15:49:02Z'
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

- [x] With five admitted generations, no seat pinned and no lease, a run removes the three oldest copies, stamps their ledger entries, and keeps the selected one and its rollback (unit).
- [x] A generation a seat record or a lease on disk names is kept, whatever its age (unit).
- [x] A lease that cannot be read keeps every generation, and the result names the seat (unit).
- [x] A retired build's version is never acquired again, and the selection never changes because of a retirement (unit).
- [x] The acquisition job runs retirement after each family's run and shows the retired ids in its status (unit).
- [x] **Post-merge** `[L4-deferred — operator handoff needed]` (an FM build carrying this): after the first pass, the Antigravity store keeps 2.31.2 and 2.31.1 plus any build a running seat is pinned to, and 2.31.0's copy is gone with `retiredAt` on its ledger entry. Receipt: https://github.com/neomjs/neo-agent-brain/issues/1665#issuecomment-6858562229

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
- 2026-11-13T13:40:02Z @neo-opus-ada referenced in commit `7c7b266` - "chore(fleet): merge the #1662 head that carries dev (#1665)"
- 2026-11-13T14:29:20Z @neo-opus-ada referenced in commit `a79802c` - "chore(fleet): merge dev after #1664 landed (#1665)"
- 2026-11-13T15:23:30Z @neo-opus-ada referenced in commit `c31995e` - "fix(fleet): retirement reads holds when it removes, fails closed (#1665)

The store calls `held` on the family's admission chain, in the synchronous step that removes, so a pin published while a retirement waits keeps its copy. A pending Start (it selects its build before its record pins it), an unreadable lease, a lease that is not the seat's own, or a pin that is not a generation id keeps every copy and names the seat. `isHarnessGenerationId` is the one id-shape check, shared by the ledger and the lease read."
- 2026-11-13T15:49:02Z @tobiu referenced in commit `7b3076c` - "feat(fleet): retire harness generations nothing holds (#1665) (#1666)

* feat(fleet): Claude Desktop seats run the newest admitted build (#1658)

Claude's own updater is off by policy, and the Fleet had no Claude source,
so every Claude seat stayed on the build it was installed with. Homebrew's
public cask record now points at Anthropic's versioned release zip; the
record only decides whether to download (the vendor's URL shape, its own
version and commit, the SHA-256 checked before extraction), and admission
still proves the bundle.

The download half every acquisition shares lives once in
acquireHarnessArchive.mjs: per-family serialization, the same-admitter
rejection memo, a capped streaming digest check, ditto extraction, the
single .app and admission.

fleet.harnessGenerations.claudeDesktop defaults to true, and the host's
updater policy decides which updater owns the family instead of refusing
a Start: with the policy, seats start from the selected generation and the
Fleet acquires; without it, seats start from the installed bundle, whose
own updater keeps it current. This amends #1584's refusal.

* feat(fleet): Codex Desktop seats run the newest admitted build (#1662)

Every Codex seat's Start keeps its own Sparkle updater off, and
fleet.harnessGenerations.codexDesktop was off, so nothing brought a new
Codex build. The census on the ticket found no app-server reader that
drives a Codex seat with the installed CLI, and every Codex CLI reader in
the Fleet already follows the seat's pin, so the leaf now defaults to true.

The seat journey pins the leaf off for its stand-in bundle, as it pins the
Codex binary, so a default never decides which app it starts.

* feat(fleet): retire harness generations nothing holds (#1665)

Every GUI family now acquires its own builds several times a week, and no
copy was ever removed. After each family's acquisition, the job now retires
every admitted generation except the selected one, its rollback, and the
ones a seat holds: every seat record's pin and every lease on disk. A lease
that cannot be read keeps everything and says which. A retired copy's ledger
entry stays with a retiredAt stamp, so the build is never selected or
fetched again, and admitting it again clears the stamp. Retirement runs on
the family's admission chain, so no admission lands between the check and
a removal.

* test(fleet): the journey child admits into its own store (#1662)

The fleet child inherits the host's HOME, and its harness-generation root defaulted to ~/.neo-ai/harnesses. Before the Codex pin, an opted-in run bootstrapped the Codex stand-in there and left a bundle-metadata-invalid rejection in the live codex-desktop ledger (02:20:26Z). The child now gets NEO_FLEET_HARNESS_GENERATIONS_ROOT under its run root, so isolation no longer depends on every family being pinned.

* fix(fleet): retirement reads holds when it removes, fails closed (#1665)

The store calls `held` on the family's admission chain, in the synchronous step that removes, so a pin published while a retirement waits keeps its copy. A pending Start (it selects its build before its record pins it), an unreadable lease, a lease that is not the seat's own, or a pin that is not a generation id keeps every copy and names the seat. `isHarnessGenerationId` is the one id-shape check, shared by the ledger and the lease read."
- 2026-11-13T15:49:03Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-13T16:18:06Z

**Post-merge receipt (AC-6), 2026-11-13.** FM installed at Brain `7b3076c8` + Institution `8e9ec437` (16:14Z). Its first acquisition pass logged:

```
16:16:33.446Z [fleet] harness acquisition antigravity: checked up-to-date (candidate 2.31.2, selected 2.31.2); retired 1
```

The live `~/.neo-ai/harnesses/antigravity/ledger.json` afterwards:

| id | version | copy | `retiredAt` |
|---|---|---|---|
| `b43c5434a13d` | 2.31.0 | gone | 2026-11-13T16:16:33.445Z |
| `9155a2ef6f51` | 2.31.1 | present | — |
| `566e218c3c40` | 2.31.2 | present | — |

No seat lease pinned a build before the pass. The store went from 1.4 GB to 982 MB, and all three ledger entries stay. AC-6 is met.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


