---
id: 1046
title: The roster's lane line reads claims from a page that never holds them
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-15T11:09:23Z'
updatedAt: '2026-10-15T11:16:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1046'
author: neo-opus-vega
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
blocking:
  - '[ ] 1047 A GitLab seat''s lane line reads its GitLab issue claims'
---
# The roster's lane line reads claims from a page that never holds them

## Context

The operator reported it on 2026-10-15 from the installed Fleet Manager: every roster card reads "no lane claimed" while peers hold live, broadcast lanes. Four were claimed that morning: Stage #39 at 09:15Z, Stage #59 at 09:18Z, #1041 at 09:47Z and #1042 at 10:10Z.

**Observed live** through the Neural Link on the installed FM's AgentOS window, at 11:06Z:
- All 12 roster records carry `sources.lane = {source: 'memory-core:mailbox', state: 'wired', reason: null}` and `laneLine: null`.
- The cockpit's `FleetActivityEvents` store holds 74 events: 65 PR, 4 issue, 2 work-stall and 3 A2A. All three A2A rows are addressed to `@tobiu`, and none is an `AGENT:*` broadcast.

## The Problem

The lane line is folded from the A2A page the Fleet holds for its viewer: the lane record in `fleetActivityComposer`, shipped by #824 on 2026-10-04. The held page's rows show that the viewer is the operator, a human principal.

Every lane claim is an `AGENT:*` broadcast, and a broadcast's cohort is agents only. A human is reached only on a direct route (`MailboxService.mjs:228–230`, stated by #860 on 2026-10-05). So the viewer's page never holds a claim, the fold stays empty, and the card renders a wired source with no line, which reads "no lane claimed" for every seat.

That is a wired source asserting an absence it cannot observe. It is indistinguishable from "nobody claimed anything". Whether a build before #860 ever delivered broadcasts to a human viewer is not established; #824 may have been verified with an agent viewer.

**Operating modes (operator, 2026-10-15).** The roster must work for teams that do not run team sharing. The D#19440 observer read admits other agents' broadcasts only under the `team` policy. Under `private` or `legacy` it admits the viewer's own messages plus the inboxes the viewer holds `CAN_READ_INBOX_OF` grants for (`MailboxService` observer policy, around `:3970`). So A2A alone cannot be the lane source for every team.

## The Architectural Reality

- **The held page.** `ai/services/fleet/fleetActivityComposer.mjs` holds A2A only for reads with no explicit observer (`holdsA2A`, `:493`; "explicit observers never enter ordinary mailbox retention"). The lane record's `fold(viewerIdentity, …)` and `claimsFor(viewerIdentity)` are scoped to that page's viewer.
- **Classification is correct.** `ai/services/fleet/fleetA2AActivityAdapter.mjs` classifies with `collisionPreventionTag` over the raw subject and normalizes `from` to the bare login. `ai/services/fleet/fleetCockpitStatus.mjs` (`:84–102`) joins claims onto rows by that login.
- **The broadcast-visible read already exists.** D#19440 (graduated 2026-10-07) chose a bounded observer view, "All" or "involves me". Its canonical read contract shipped as #921 via #952 (2026-10-09, "observe A2A history without changing receipts"). neomjs/neo-agent-institution#596 consumes it for the Activity pane only.
- **A GitHub lane source already exists.** `ai/services/fleet/fleetPrLaneActivityAdapter.mjs` emits `lane-claim` events from issue comments (source `commentLane`) and assignee facts (`issue-activity`, `agentId = assignees[0]`). Every sharing mode admits this public forge data. The roster fold in `fleetCockpitStatus.mjs` skips any claim whose source is not the A2A source, so neither ever reaches a card.
- **The card.** Institution renders `state === 'wired' && !laneLine` as "no lane claimed" (`apps/agentos/view/fleet/roster/card/Container.mjs:527`).

## The Fix

1. **Fold claims from every source the active mode admits**, instead of the viewer's own inbox page:
   - (a) the D#19440 / #952 observer read, where `team` policy or `CAN_READ_INBOX_OF` grants admit broadcasts. It must keep the no-receipt property: observing changes no read state.
   - (b) The seat's own forge, in every mode: lane-claim comments and self-assignment. The ticket gate already makes every lane a self-assigned ticket. The forge comes from the agent definition's `forge` / `forgeHost`, which the Fleet already carries (`FleetControlBridge`, `FleetLifecycleService` branches on `gitlab`). GitHub seats read through the existing `fleetPrLaneActivityAdapter`. The lane-fact contract (login, forge, issue reference, claimedAt, line, source) stays forge-neutral, so a GitLab reader plugs in without changing the fold.
   
   Keep claims keyed by login, let the newest claim win across sources, and have each row name the source that answered.
2. **Make a blind source say so.** The capability reports which sources the active mode admitted, and `degraded` with a reason when the A2A source is clamped away. It never reports `wired` with an empty fold that it could not have filled.

## Acceptance Criteria

- [ ] AC-1 (`team` policy): With a human viewer, a `[lane-claim]` broadcast from a seat becomes that seat's lane line. Unit: a fake mailbox whose inbox excludes broadcasts but whose team observer view includes them.
- [ ] AC-1b (`private` / `legacy` policy, no grants): A seat's GitHub lane claim (a lane-claim comment or a self-assignment) becomes its lane line, so the roster works without team sharing. The capability states that the A2A source is clamped.
- [ ] AC-1c: A seat whose forge has no lane reader yet (today, `gitlab`) shows its forge lane source as `not-wired` with that reason, never an empty `wired` line. A2A claims still fill its line where the mode admits them, since that path is forge-independent.
- [ ] AC-2: The lane capability names the sources the active mode admitted. It reports `degraded` with a reason whenever the A2A source is clamped, never `wired` with an empty fold it could not have filled.
- [ ] AC-3: A `claim-corrected` release still clears a line, and a newer claim still replaces an older one, under the new source.
- [ ] AC-4: Feeding the lane record stamps no read, seen or receipt state on any message. Tested.
- [ ] AC-5 (post-install): on the installed FM, the roster shows the lane claims broadcast in the last hour.

## Out of Scope

- **The GitLab lane reader itself:** that is #1047, blocked by this ticket. GitLab open work and activity are #1048. Both are low priority under #684.

- The Activity pane's All / involves-me view: that is neomjs/neo-agent-institution#596.
- Changing who receives broadcasts. The agents-only cohort is the mailbox's rule, and the observer read exists so the Fleet does not need to change it.

## Avoided Traps

- **Adding the operator to the `AGENT:*` cohort.** That turns status broadcasts into the human's inbox and its unread counts.
- **Folding claims from an ordinary inbox read with a larger page.** A bigger window over the wrong population still holds no broadcasts.

## Related

- #824: the lane line.
- #860: the human-recipient rule.
- #921 and #952: the observer contract.
- D#19440.
- neomjs/neo-agent-institution#418 and #596.

**Decision Record impact:** none. This aligns with the D#19440 outcome.

**Sweeps (2026-10-15T11:08Z):**
- Live latest-open: the latest 20 open neo-agent-brain issues and the latest 20 Institution issues contain no equivalent.
- Search: four org-wide queries on lane, roster and broadcast terms returned nothing open.
- A2A: no in-flight claim on this scope.
- MC: prior art is #740 / #418 / #822 (the shipped lane line), with no diagnosis of this failure.
- Own assignments: none.

unowned-rationale: filed by Vega with about 6 % weekly budget left, which is reserved for reviews today. This is a Brain fleet lane, offered to the GPT seats; @neo-gpt holds #596 and its observer wiring.

Retrieval Hint: "roster lane line no lane claimed AGENT broadcast human viewer observer"

Origin Session ID: 2efbd8f3-1462-47da-af2e-5ba6501d5d09

Vega (Opus 5.5, Claude Code)





## Timeline

- 2026-10-15T11:09:24Z @neo-opus-vega added the `bug` label
- 2026-10-15T11:09:25Z @neo-opus-vega added the `ai` label
- 2026-10-15T11:09:25Z @neo-opus-vega added the `agent-os` label
- 2026-10-15T11:16:21Z @neo-opus-vega cross-referenced by #1047
- 2026-10-15T11:16:23Z @neo-opus-vega cross-referenced by #1048
- 2026-10-15T11:16:37Z @neo-opus-vega marked this issue as blocking #1047

