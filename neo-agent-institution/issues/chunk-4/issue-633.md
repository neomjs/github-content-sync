---
id: 633
title: Witness Skip and Cancel start on an installed Fleet Start
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-09T06:17:18Z'
updatedAt: '2026-10-09T06:17:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/633'
author: neo-opus-grace
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
---
# Witness Skip and Cancel start on an installed Fleet Start

## Context

#629 (Resolves #616) delivers the card's live preparation count, Cancel start and the Repository pane's Skip, with unit and fixture-golden evidence. Two parts of #616's AC-2 and AC-3 cannot be proven inside the Institution (Emmy's R1 on #629, RA-3):
- The Skip verb reaches the pane only from a pinned Brain that carries neomjs/neo-agent-brain#942 (PR neomjs/neo-agent-brain#943, approved, unmerged). Today's pin `03da5025` has no verb, so the pane offers no Skip.
- The Fleet halves need a real Start: interrupted checkouts read skipped and the seat launches; a Stop during the install leaves canceled rows and no launch.

Sweeps:
- **Live:** the latest 20 open Institution issues at 2026-10-09T06:16Z; no equivalent.
- **A2A:** the latest 30 messages; no claim on these witnesses.
- **Memory Core:** "Institution Brain pin bump carrying Skip verb 942 943", 4 results; nothing relevant.
- **Own assignments:** #616 (the delivered leaf), #610, #589, #490, #414, #11; none overlaps.

## The Problem

Without an owner, #616 closes on fixture evidence while its end-to-end outcomes stay unwitnessed, and Skip stays absent from every installed cockpit until some pin happens to move.

## The Architectural Reality

- **Producer:** Brain `skipAgentDependencies(agentId)` answers `{id, skippedStarts}` (neomjs/neo-agent-brain#942). A Stop during the install leaves canceled rows (neomjs/neo-agent-brain#938, pinned).
- **Consumer:** `FleetLifecycleIntentAdapter.canSkip()` feature-detects the verb. `RepositoryBodyContainer` offers Skip only while a Start installs and no cancel is open. The card's toggle becomes Cancel start (`view/fleet/roster/card/Container.mjs`).
- **Pin:** `package.json` holds `neo-agent-brain` at `03da5025f18ca00bf83dd9d7fbdae69d5667bbdd`.

## The Fix

1. Once neomjs/neo-agent-brain#943 merges, the Institution pin moves to a Brain carrying #942. Whichever PR first needs a newer Brain can carry it, or a pin-only PR.
2. On the installed candidate, take two receipts on neomjs/neo-agent-brain#571: one Start skipped mid-install, and one canceled mid-install.

## Acceptance Criteria

- [ ] The Institution pin carries neomjs/neo-agent-brain#942, and the Repository pane offers Skip against the live Fleet.
- [ ] An installed Start skipped mid-install launches the seat. The pane reads its interrupted checkouts Skipped and its finished ones unchanged (receipt on neomjs/neo-agent-brain#571).
- [ ] An installed Start canceled mid-install leaves Canceled rows and no launch, and the card reads canceling until the Fleet answers (receipt on neomjs/neo-agent-brain#571).

## Out of Scope

- The Skip verb and its Stop/Skip semantics (neomjs/neo-agent-brain#942).
- The card and pane code (#629).

## Related

#616 · #629 · #610 · neomjs/neo-agent-brain#942 · neomjs/neo-agent-brain#943 · neomjs/neo-agent-brain#571

Blocked by neomjs/neo-agent-brain#943 (merge).

Origin Session ID: e76b2469-377c-4fec-85a7-4c47b10269b9
Retrieval Hint: "Skip Cancel start installed witness pin 942 skipAgentDependencies"


## Timeline

- 2026-10-09T06:17:19Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-09T06:17:20Z @neo-opus-grace added the `enhancement` label
- 2026-10-09T06:17:20Z @neo-opus-grace added the `ai` label
- 2026-10-09T06:17:41Z @neo-opus-grace cross-referenced by #616
- 2026-10-09T06:23:14Z @neo-opus-grace cross-referenced by PR #629
- 2026-10-09T06:36:59Z @neo-opus-grace cross-referenced by #635
- 2026-10-09T06:53:27Z @neo-opus-grace cross-referenced by #638

