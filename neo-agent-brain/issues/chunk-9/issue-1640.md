---
id: 1640
title: The Stop hook's night arm keys on the recorded operator presence
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T06:38:14Z'
updatedAt: '2026-11-13T05:30:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1640'
author: neo-opus-ada
commentsCount: 3
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
closedAt: '2026-11-12T17:06:13Z'
---
# The Stop hook's night arm keys on the recorded operator presence

## Context

PR #1639 (#1618) gives the Stop hook's question-box mirror a night arm: in a turn the operator did not prompt, a `wrap_up` call with an `ask` blocks once, and the directive says to take the stated pick or record the question on its ticket. "Did not prompt" is the hook's prompting classifier. Since #1626, #1632 and the Nightshift toggle (`neomjs/neo-agent-institution#929`), the plane records the operator's presence itself. The hook does not read that record.

## The Problem

Replaying every `wrap_up` ask in one Claude seat's local transcripts (83 sessions, 494 asks; the newest ask of each turn, classified with the hook's own `extractLatestHumanUserTextFromJsonl` + `isOperatorDialogueText`) shows the classifier and the operator's presence disagree both ways:

1. **It fires while he is there.** 92 asks fall in turns the classifier calls unprompted. The operator's next message of his own came within 30 minutes after 50 of them, and within 5 minutes after 26. Neither a turn prompted by a background-task notification (34 of the 92) nor one whose human-filtered walk stops at a non-human record (57) says he left.
2. **It misses the night it is for.** A handoff in the operator's own words reads as dialogue unless it matches `AUTONOMOUS_HANDOFF_PATTERNS` (`ai/scripts/lifecycle/stopHookDecision.mjs`): "nightshift", "freely choose … for the next N h", "merge when I get back", "you drive … until". A handoff like "back at <time>, you can freely choose what to focus on" matches none of them, so every turn of the chain after it counts as attended.

## The Architectural Reality

- `decideDeferenceStopHookAction` (`ai/scripts/lifecycle/stopHookDecision.mjs`) takes `operatorInLoop` from the adapter's `classifyPromptingContext`; since #1639 the night arm is `!phrase && !operatorInLoop && ask`.
- `OperatorPresenceService` (`ai/services/memory-core/OperatorPresenceService.mjs`) holds `{mode: 'present'|'away', since, until, expired}`. Only the Fleet's human operator writes it, through `FleetControlBridge#setOperatorPresence` (`ai/services/fleet/`), which sends one `[operator-presence]` notice per change. `who_is_online` carries the read, and the FM's `fleetRoster` hands it to the cockpit. **A plane where nobody ever set it reads `{mode: 'present', since: null}`** ("the fleet's standing assumption"), not a recorded presence.
- The Claude hook runs host-side in each seat's checkout and imports from the Agent OS runtime root (`ai/scripts/lifecycle/hooks/projectSeatHooks.mjs`). It resolves `AiConfig` in `resolveStopHookPolicy` (`hooks/claude/laneStateStopHook.mjs`), but it cannot call the plane's tools.
- **An FM-spawned seat gets only allowlisted env plus the reserved Start injections** (`FleetLifecycleService#start`: `NEO_AGENT_IDENTITY`, `NEO_SEAT_PLANE_BASE` "so the seat's hooks reach that plane as the seat", …). Measured in a running Claude Desktop seat: both reach its process tree, `NEO_HOOK_PROJECTION_PATH` does not. So a leaf the hook reads stays empty in every seat unless the Start fills its env.
- The per-seat carrier built for this, the hook projection (`hookProjectionReader.mjs`, `stopHook.projection`), has no production writer: only specs import `makeHookProjectionWriter`. Its `path` leaf is the consumer shape this ticket reuses: `leaf(null, …)`, "injected by the trusted seat boot boundary".
- `fleet.dataDir` (`ai/configBase.mjs`) is the fleet's own plane-member directory, placed in every profile, so a record beneath it needs no new writer leaf and no Institution change.
- Design authority: `OperatorPresenceService`'s JSDoc, "The mode answers one question: will an operator-only ask (org admin, credentials, merges) get a quick answer?", and the operator's note on #1618, "it gets more tricky in nightshifts, when you act while i am not around."

## The Fix

1. **A host record of the presence.** The FM's fleet child writes `{mode, since, until, writtenAt}` to `operator-presence.json` under `AiConfig.fleet.dataDir` whenever `setOperatorPresence` answers `changed`, and whenever `fleetRoster` reads a plane presence that differs from the last one written (a write from elsewhere, an FM restart). Temp file + rename; a failed write never fails the verb or the read.
2. **The Start carries the path.** `FleetLifecycleService#start` injects the record's absolute path under a new reserved slot, `NEO_OPERATOR_PRESENCE_PATH`, beside `NEO_SEAT_PLANE_BASE`; `launch.env` and the seat's `.env` cannot preload it. The hook resolves it through one consumer leaf, `seat.operatorPresencePath` (`leaf('', 'NEO_OPERATOR_PRESENCE_PATH', 'string')`, empty = no record), beside `seat.planeBase`, the subtree that holds what a seat's hooks read as the seat; the grace is `stopHook.operatorPresenceGraceMs` (`leaf(600000, null, 'number')`, no env knob).
3. **The hook reads it.** The Claude adapter reads the file (one `readFileSync`; a failure means no record), and the shared decision derives `operatorAway`: `true` when `mode === 'away'`, `until` is not past, and the transcript's latest operator message is not later than `since` plus the grace (so a "good night" typed right after the switch still counts as the handoff); `false` for a `present` record, or an `away` ended by `until` or by a later operator message; `null` for no record, including `since: null`, so a fleet that never toggles keeps #1639's arm.
4. **The night arm keys on it:** `!phrase && (operatorAway ?? !operatorInLoop) && ask`.
5. **The reminder names the record** when it fired on it, e.g. "The operator switched Nightshift on at <since>".

## Acceptance Criteria

- [ ] With an `away` record and no later operator message, a `wrap_up` ask blocks once in any turn, including a chain the operator's own handoff prompted. Unit: `stopHookDecision.spec.mjs` and the adapter seam in `laneStateStopHook.spec.mjs`.
- [ ] With a `present` record, an unprompted turn's ask does not block; with no record, or a record with `since: null`, the behavior of #1639 holds.
- [ ] An operator message after `since` (past the grace) ends the `away` reading for that turn; a past `until` ends it.
- [ ] The FM writes the record on a `changed` write and on a roster read that disagrees with it, and leaves it alone on a roster read that agrees or did not answer (unit, in the writer's spec).
- [ ] `FleetLifecycleService#start` injects `NEO_OPERATOR_PRESENCE_PATH`, and a `launch.env` or seat `.env` naming it is refused (unit).
- [ ] The leaves pass `lint-config-template-ssot` and `check-aiconfig-antipatterns`.
- [x] Post-merge: on the installed FM, switching Nightshift on writes the record, and a Claude turn ending on a box ask after the switch writes one `BLOCK … the wrap-up ask` audit line. Witnessed on an operator-prompted turn, the case only the record can block: [receipt](https://github.com/neomjs/neo-agent-brain/issues/1640#issuecomment-6845043451).

## Out of Scope

- Widening `AUTONOMOUS_HANDOFF_PATTERNS`. It also decides lane continuation and voluntary stops, so a broader handoff reading changes far more than the box.
- Giving the hook projection its production writer. If that lands first, the presence can ride it instead of its own file.
- The deference arm in dialogue (#1618) and the night's turn supply (#95).

## Avoided Traps

- **Inferring absence from the prompt.** The replay above is the evidence: half of the "unprompted" asks were answered within half an hour.
- **Calling the plane from the hook.** A Stop hook that waits on the network can hang a turn-end; one local file read cannot.
- **A second resolver for the path.** The writer derives it from `fleet.dataDir` in one place; the Start serializes that value into the seat's env, and the hook reads its own leaf (ADR-0019 B1, C1; §10.7's Start serialization).
- **Reading the never-set plane as `present`.** It would silence #1639's night arm on every seat whose operator never uses the toggle.

## Related

#1618 and PR #1639 (the night arm this narrows) · #1626, #1632 (the plane record and its FM verb) · `neomjs/neo-agent-institution#929` (the toggle) · #95 (presence-aware wakes, the night's turn supply)

Decision Record impact: aligned-with `adr:0019-aiconfig-reactive-provider-ssot` (two consumer leaves, `seat.operatorPresencePath` and `stopHook.operatorPresenceGraceMs`; the writer reads `fleet.dataDir`).

Not a good first issue: it changes what ends a turn on every Claude seat, and it spans the FM writer, the seat Start, two config leaves and the hook.

Live latest-open + closed sweep: latest 20 open (156 open, red band; no ticket owns the box's night behavior, and #95 owns wake policy, not turn ends) + closed search ("operator presence stop hook", "night question box", "unattended turn wrap-up", "nightshift hook"; hits read by title, none on this surface) at 2026-11-12T06:37Z; no equivalent or prior exclusion found. A2A in-flight claim sweep: latest 30 messages, all read states, at 06:38Z; no claim on this surface. MC sweep: "question box waits unanswered while the operator is away at night; operator replied within minutes to an unprompted turn's ask", 6 results, no prior decision. Own-assignment sweep: 5 open; #1618 is this ticket's predecessor, none overlapping. Structure map: `--files --loc` run; `ai/scripts/lifecycle`, `ai/services/fleet` and `OperatorPresenceService.mjs` exist. Implementation placed one module, `ai/services/fleet/operatorPresenceRecord.mjs` (the record's path, format and atomic write, shared by the bridge and the Start), beside the sibling single-concern modules there.

Intake (2026-11-12, drift probe fired: #1639 moved `stopHookDecision.mjs`): Prescription checked: `ai/services/fleet/FleetLifecycleService.mjs#start` — the leaf alone never reaches the hook; the Start owns the carrier. `ai/scripts/lifecycle/stopHookDecision.mjs#decideDeferenceStopHookAction` owns the arm; `ai/services/fleet/FleetControlBridge.mjs` owns the writes.

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
- 2026-11-12T13:48:46Z @neo-opus-ada cross-referenced by PR #1645
- 2026-11-12T15:20:48Z @neo-opus-ada referenced in commit `85679bc` - "fix(fleet): an overlapped roster read never undoes a switch (#1640)

fleetRoster() published the presence it awaited even when a setOperatorPresence write completed while that read was out. The plane may have answered the read before the write, so an older reading could replace the newer record and silence the night arm while the plane read away. The bridge now counts its completed presence writes, and a roster read during which the count moved publishes nothing; the next read publishes what it missed, so newer external changes and failed-write retries still land."
- 2026-11-12T15:20:48Z @neo-opus-ada referenced in commit `5d9e4be` - "fix(lifecycle): a return ends the away whatever it says (#1640)

The Claude adapter passed the prompting message's timestamp to decideOperatorAway only when operatorInLoop held, and that dialogue gate also refuses handoff wording. A genuine return past the grace ("Nightshift is over; I am back") therefore kept the away reading and blocked a human-only ask that the same return in plain words allows. isOperatorAuthoredText now answers who wrote the text (not a [WAKE], not synthetic, an attested chain), isOperatorDialogueText adds the handoff question on top, and classifyPromptingContext reports operatorAuthored, which decideTurnOperatorAway feeds into the timing decision. The handoff classifier keeps its scope."
### @neo-opus-ada - 2026-11-12T17:02:59Z

**Handover (session refresh; the owner stays me):** PR #1645 is at `5d9e4be4`, green 17/17. Euclid's Round-1 actions are both addressed (author response on the PR: RA-1, an overlapped roster read never undoes a switch; RA-2, an operator-authored return ends the away whatever it says). Euclid is re-requested for the disposition-only Round 2.

Pickup: on Euclid's R2, an approval goes to the merge handoff; a still-open row gets its fix on the same branch. After the merge, AC-7 needs an FM build at a Brain head that contains #1645, followed by the installed receipt, both owned under #1508. The plane and FM run Brain `528105e` today, which does not include #1645.

Origin Session ID: 2d49a7be-ad12-4e49-8c10-30d4d09d0b5a

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-12T17:06:13Z @tobiu referenced in commit `f25fb9b` - "feat(lifecycle): the night arm keys on the recorded presence (#1640) (#1645)

* feat(lifecycle): the night arm keys on the recorded presence (#1640)

The Fleet mirrors the plane's operator presence into a host record beneath fleet.dataDir and hands every seat its path at Start (NEO_OPERATOR_PRESENCE_PATH, read as seat.operatorPresencePath). The Claude Stop hook reads it with one file read: an away record blocks a box ask in any turn, a present one silences the unattended arm, and no record (including the plane's never-set reading) keeps the prompting classifier.

* fix(fleet): an overlapped roster read never undoes a switch (#1640)

fleetRoster() published the presence it awaited even when a setOperatorPresence write completed while that read was out. The plane may have answered the read before the write, so an older reading could replace the newer record and silence the night arm while the plane read away. The bridge now counts its completed presence writes, and a roster read during which the count moved publishes nothing; the next read publishes what it missed, so newer external changes and failed-write retries still land.

* fix(lifecycle): a return ends the away whatever it says (#1640)

The Claude adapter passed the prompting message's timestamp to decideOperatorAway only when operatorInLoop held, and that dialogue gate also refuses handoff wording. A genuine return past the grace ("Nightshift is over; I am back") therefore kept the away reading and blocked a human-only ask that the same return in plain words allows. isOperatorAuthoredText now answers who wrote the text (not a [WAKE], not synthetic, an attested chain), isOperatorDialogueText adds the handoff question on top, and classifyPromptingContext reports operatorAuthored, which decideTurnOperatorAway feeds into the timing decision. The handoff classifier keeps its scope."
- 2026-11-12T17:06:14Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-12T17:25:17Z

**AC-7 receipt, part 1 — installed FM carries #1645.**

- **Installed** at 17:23Z: Brain `f25fb9b9` · Engine `5ecce11` · Institution `821376a`, runtime generation `0d6198e4`. Gates: packaged rehearsal ok on a fresh root; cdhash `469fa943` and Electron 43.8.1 equal the previous install; dry run clean; custody digest unchanged. Rollback is the `528105e` install.
- **Handoff:** sealed in 2878 / 8000 ms with 6 survivors. Recovery read `previous issuer handed-off`, and the plane attach adopted the stored credential.
- **The writer, restart path:** the first roster read after boot wrote `fleet/operator-presence.json` at 17:23:45Z with the plane's current reading, `{"mode":"present","since":null,"until":null}`. The plane holds no presence record yet (`since: null`), so the hook reads it as no record (AC-2) and #1639's arm holds.

**Still open for AC-7:**
1. The first Nightshift switch must write `mode: "away"` with a `since`.
2. A seat Started by this FM must end an unprompted turn on a box ask and write one `BLOCK … the wrap-up ask` line. Seats that were already running keep the previous generation's hooks, and their env has no `NEO_OPERATOR_PRESENCE_PATH` until their next Start.

I check both at the next Nightshift switch and seat Start, then tick AC-7.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-13T05:29:46Z

**AC-7 receipt, part 2: the night arm fires on the operator's first Nightshift switch.**

- **The record.** The operator pressed Nightshift at 05:17:13Z. The FM wrote my seat's presence record (`NEO_OPERATOR_PRESENCE_PATH`): `{"mode":"away","since":"2026-11-13T05:17:13.049Z","until":null}`, written 05:17:13.053Z. The plane record and the single `[operator-presence] away` broadcast agree (Grace's receipt on `neomjs/neo-agent-institution#929`).
- **The block.** My turn ended at 05:26:54Z on a wrap-up ask. The hook's audit log holds one `BLOCK (…, identity=neo-opus-ada, operatorInLoop=true, operatorAway=true, …): the wrap-up ask "Should I start the Institution #938 review next?" at turn-terminal`, and the reminder named the record: "The operator switched Nightshift on at 2026-11-13 05:17 UTC". The turn went on with its stated pick.
- **Why this turn discriminates.** The operator prompted it before `since`, so the prompting classifier reads it as attended (`operatorInLoop=true`) and #1639's arm alone would have let it end. Only the presence record blocks it. An unprompted turn would block under #1639 too, so it would not test this change.

Out of scope, seat-local: my checkout's `.claude/settings.local.json` registers this hook a second time without `NEO_LANE_STATE_ENFORCE`, so each of my turn ends writes two lines (here a paired `WOULD-BLOCK`); the other seats write one.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


