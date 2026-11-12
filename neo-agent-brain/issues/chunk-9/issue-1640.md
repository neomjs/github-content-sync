---
id: 1640
title: The Stop hook's night arm keys on the recorded operator presence
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T06:38:14Z'
updatedAt: '2026-11-12T06:38:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1640'
author: neo-opus-ada
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
# The Stop hook's night arm keys on the recorded operator presence

## Context

PR #1639 (#1618) gives the Stop hook's question-box mirror a night arm: in a turn the operator did not prompt, a `wrap_up` call with an `ask` blocks once, and the directive says to take the stated pick or record the question on its ticket. "Did not prompt" is the hook's prompting classifier. Since #1626, #1632 and the Nightshift toggle (`neomjs/neo-agent-institution#929`), the plane records the operator's presence itself. The hook does not read that record.

## The Problem

Replaying every `wrap_up` ask in one Claude seat's local transcripts (83 sessions, 494 asks; the newest ask of each turn, classified with the hook's own `extractLatestHumanUserTextFromJsonl` + `isOperatorDialogueText`) shows the classifier and the operator's presence disagree both ways:

1. **It fires while he is there.** 92 asks fall in turns the classifier calls unprompted. The operator's next message of his own came within 30 minutes after 50 of them, and within 5 minutes after 26. Neither a turn prompted by a background-task notification (34 of the 92) nor one whose human-filtered walk stops at a non-human record (57) says he left.
2. **It misses the night it is for.** A handoff in the operator's own words reads as dialogue unless it matches `AUTONOMOUS_HANDOFF_PATTERNS` (`ai/scripts/lifecycle/stopHookDecision.mjs`): "nightshift", "freely choose … for the next N h", "merge when I get back", "you drive … until". A handoff like "back at <time>, you can freely choose what to focus on" matches none of them, so every turn of the chain after it counts as attended.

## The Architectural Reality

- `decideDeferenceStopHookAction` (`ai/scripts/lifecycle/stopHookDecision.mjs`) takes `operatorInLoop` from the adapter's `classifyPromptingContext`; with #1639 the night arm is `!phrase && !operatorInLoop && ask`.
- `OperatorPresenceService` (`ai/services/memory-core/OperatorPresenceService.mjs`) holds `{mode: 'present'|'away', since, until, expired}`. Only the Fleet's human operator writes it, through `FleetControlBridge#setOperatorPresence` (`ai/services/fleet/`), which sends one `[operator-presence]` notice per change. `who_is_online` carries the read, and the FM's `fleetRoster` hands it to the cockpit.
- The Claude hook runs host-side in each seat's checkout and imports from the Agent OS runtime root (`ai/scripts/lifecycle/hooks/projectSeatHooks.mjs`). It resolves `AiConfig` in `resolveStopHookPolicy` (`hooks/claude/laneStateStopHook.mjs`), but it cannot call the plane's tools.
- The per-seat carrier built for this, the hook projection (`hookProjectionReader.mjs`, `stopHook.projection`), has no production writer: only specs import `makeHookProjectionWriter`. Nothing reaches a seat through it today.
- Host-edge consumer leaves have precedent: `fleet.wakeReceiverRecordsDir` (`ai/configBase.mjs`) and `focusV1ProjectionPath` (`ai/mcp/server/memory-core/configBase.mjs`). Each is `leaf('', …, 'string')`, and empty means no source, so the reader degrades cleanly.
- Design authority: `OperatorPresenceService`'s JSDoc, "The mode answers one question: will an operator-only ask (org admin, credentials, merges) get a quick answer?", and the operator's note on #1618, "it gets more tricky in nightshifts, when you act while i am not around."

## The Fix

1. **A host record of the presence.** The FM writes `{mode, since, until, writtenAt}` to one JSON file whenever `setOperatorPresence` answers `changed`, and whenever its roster read sees a mode or `until` that differs from the file (a plane restart, an expiry). Its path is one new host-edge consumer leaf, declared per ADR-0019 §5.2 (`leaf('', 'NEO_…', 'string')`, empty = no record) and read at both use sites.
2. **The hook reads it.** `resolveStopHookPolicy` reads the leaf. The adapter reads the file (one `readFileSync`; a failure means no record) and passes `operatorAway` to `decideDeferenceStopHookAction`. `operatorAway` holds when `mode === 'away'`, `until` is not past, and the transcript has no operator message later than `since` plus a short grace (so a "good night" typed right after the switch still counts as the handoff).
3. **The night arm keys on it.** With a record, the arm fires exactly when `operatorAway` holds, and a `present` record silences it. Without one (empty leaf, missing or unreadable file), today's `!operatorInLoop` stays the fallback.
4. **The reminder names the record** when it fired on it, e.g. "The operator switched Nightshift on at <since>".

## Acceptance Criteria

- [ ] With an `away` record and no later operator message, a `wrap_up` ask blocks once in any turn, including a chain the operator's own handoff prompted. Unit: `stopHookDecision.spec.mjs` and the adapter seam in `laneStateStopHook.spec.mjs`.
- [ ] With a `present` record, an unprompted turn's ask does not block; with no record, the behavior of #1639 holds.
- [ ] An operator message after `since` (past the grace) ends the `away` reading for that turn; a past `until` ends it.
- [ ] The FM writes the record on a `changed` write and on a roster read that disagrees with it (unit, in the spec of whichever owner writes it).
- [ ] The leaf passes `lint-config-template-ssot` and `check-aiconfig-antipatterns`.
- [ ] Post-merge: on the installed FM, switching Nightshift on writes the record, and an unprompted Claude turn ending on a box ask writes one `BLOCK … the wrap-up ask` audit line.

## Out of Scope

- Widening `AUTONOMOUS_HANDOFF_PATTERNS`. It also decides lane continuation and voluntary stops, so a broader handoff reading changes far more than the box.
- Giving the hook projection its production writer. If that lands first, the presence can ride it instead of its own file.
- The deference arm in dialogue (#1618) and the night's turn supply (#95).

## Avoided Traps

- **Inferring absence from the prompt.** The replay above is the evidence: half of the "unprompted" asks were answered within half an hour.
- **Calling the plane from the hook.** A Stop hook that waits on the network can hang a turn-end; one local file read cannot.
- **A second resolver for the path.** One leaf, read at the writer and at the hook, and no exported constant (ADR-0019 B1, C1).

## Related

#1618 and PR #1639 (the night arm this narrows) · #1626, #1632 (the plane record and its FM verb) · `neomjs/neo-agent-institution#929` (the toggle) · #95 (presence-aware wakes, the night's turn supply)

Decision Record impact: aligned-with `adr:0019-aiconfig-reactive-provider-ssot` (one new host-edge consumer leaf).

Not a good first issue: it changes what ends a turn on every Claude seat, and it spans the FM writer, a config leaf and the hook.

Live latest-open + closed sweep: latest 20 open (156 open, red band; no ticket owns the box's night behavior, and #95 owns wake policy, not turn ends) + closed search ("operator presence stop hook", "night question box", "unattended turn wrap-up", "nightshift hook"; hits read by title, none on this surface) at 2026-11-12T06:37Z; no equivalent or prior exclusion found. A2A in-flight claim sweep: latest 30 messages, all read states, at 06:38Z; no claim on this surface. MC sweep: "question box waits unanswered while the operator is away at night; operator replied within minutes to an unprompted turn's ask", 6 results, no prior decision. Own-assignment sweep: 5 open; #1618 is this ticket's predecessor, none overlapping. Structure map: `--files --loc` run; `ai/scripts/lifecycle`, `ai/services/fleet` and `OperatorPresenceService.mjs` exist; no new file is placed.

Origin Session ID: 50b913e7-d81f-47b7-b274-84fbf2a6588c

Retrieval Hint: "night arm question box keyed on operator presence; prompting classifier vs Nightshift record; 50 of 92 unprompted asks answered within 30 minutes"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T06:38:14Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T06:38:15Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T06:38:15Z @neo-opus-ada added the `ai` label
- 2026-11-12T06:38:16Z @neo-opus-ada added the `model-experience` label
- 2026-11-12T06:38:16Z @neo-opus-ada added the `agent-os` label
- 2026-11-12T06:38:58Z @neo-opus-ada cross-referenced by PR #1639

