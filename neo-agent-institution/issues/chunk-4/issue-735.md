---
id: 735
title: 'At login, a plane that answers after the FM''s boot is never attached'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-10-18T22:58:27Z'
updatedAt: '2026-10-18T22:58:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/735'
author: neo-fable-clio
commentsCount: 0
parentIssue: 424
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
# At login, a plane that answers after the FM's boot is never attached

Part of #424 (an ordinary failure returns to live by the product's own guidance). Sibling of #734, which handles the other half of the same *starting* state.

## Context

Operator, 2026-10-19: after a laptop restart both the Agent OS containers and the Fleet Manager auto-start. The FM is up before the plane answers. The roster stays empty, and the FM never recovers a plane that comes up a minute later. Only an FM restart helps.

The source confirms the report end to end (read at Institution `origin/dev` `2f9056a`, Brain `origin/dev` `2d62f2e6`):

1. The shell boots its Fleet child once, in plane-attach mode (`harness/main.mjs` ≈:1533, the one `bootProductBrain()` call).
2. The child's `planeClient.init` answers `plane unreachable (connect failure)` while the containers are still coming up (`ai/services/fleet/planeMailboxClient.mjs` ≈:157); the child logs `[fleet] plane mode refused …` and exits 1 (`ai/services/fleet/devFleetServer.mjs` ≈:159–163). Fail-closed by contract.
3. The shell types the exited child `plane-refused` (`harness/brain.mjs` ≈:230–244), probes the plane once within `PLANE_PROBE_DEADLINE_MS` (10 s, ≈:298) and lands on `plane-unreachable` (≈:258).
4. The boot promise's `catch` logs `HARNESS_BRAIN_BOOT_FAILED`, settles the Brain boot as down and sets the transport fact to failed (`harness/main.mjs` ≈:1534–1540). Nothing in the shell boots the Fleet child again.
5. The cockpit shows the `plane unreachable` pill with the lead *"No plane answered at that address. Bring that plane back, or connect to another."* (`apps/agentos/util/SpineBanner.mjs` ≈:105). The plane comes back on its own, and the sentence stays. The only ways out are Connect (re-registering the same plane) or Relaunch.

## The Problem

The shell treats a plane that has not answered *yet* as a plane that is gone. At login that is the ordinary case: Docker and four containers take longer than an Electron app. Row 5's terminal predicate (#424) says the product returns to `live` by its own guidance alone; here the guidance names an action the operator cannot take ("bring that plane back": it is coming) and the state never updates when the plane arrives. Every login becomes a manual FM restart, which is the trap #732 names for updates, now on the daily path.

## The Architectural Reality

- `harness/main.mjs` ≈:1533–1540: one boot, one settle; the transport fact is terminal after a refusal.
- `harness/brain.mjs` ≈:230–244 `planeRefusal`, ≈:313 `typePlaneRefusal` (the connect card's probe, 8 s per request), ≈:255–259 the cause table (`unreachable` → `plane-unreachable`).
- `harness/brain.mjs` ≈:279–281: a plane that does not answer *while the shell runs* is a transport fact with Reconnect. At boot there is no equivalent.
- `apps/agentos/util/SpineBanner.mjs` ≈:101–106 `PLANE_REFUSALS`: the four refusal pills, each with a lead sentence (#425, #446).
- Brain `devFleetServer.mjs` ≈:159–163: the child's refusal exit. That contract stays; this ticket never asks the child to wait.

Design authority: `harness/brain.mjs` ≈:181, *"A configured plane is authority even when its ingress is currently unreachable: selecting full host ownership at that moment would create the competing organism this guard is meant to prevent. Readiness remains the Fleet transport's authenticated responsibility."* So waiting is the only honest move; owning is forbidden and giving up is what this ticket retires. #477: every state carries a reason and a next step. The `plane-unreachable` lead presumes a plane that is gone for good; at login it is one that has not answered yet.

## The Fix

A plane the shell has attached before and that does not answer at boot is a **starting** state, not a failure.

1. **Trigger.** The refusal types `plane-unreachable` *and* the launch took its plane from a stored record (`launchedPlaneRecord`, `harness/main.mjs` ≈:1097). A plane that never answered before, or a refusal with any other cause (credential, identity, not a plane), keeps today's path at once.
2. **State.** The transport fact becomes `{phase: 'starting', reason: 'plane-starting', since}`. The pill's text carries the state and the clock, the lead carries the sentence (Mnemosyne's tier-one read on #424: a stranger sees the text, the lead only on hover):
   - text: `Agent OS starting · 24 s`
   - lead: *"Your Agent OS has not answered yet. The Fleet joins by itself when it does."*
   - after 60 s the lead adds *"Longer than usual."*; no action is offered, Connect and Relaunch stay where they are.
3. **Cadence.** The shell re-probes the plane with the connect card's probe (the one `typePlaneRefusal` uses), one probe in flight at a time, the next 5 s after the previous answers. The first answer that is a plane admitting the stored PAT boots the Fleet child again through the existing plane-attach path; from that spawn, #734's readiness rules apply.
4. **Bound.** Silence past a named constant (proposed 3 min, measured on the operator's machine before the PR fixes it: Docker plus the four containers at login) ends in today's `plane-unreachable` refusal, its words and actions unchanged, the elapsed time in the lead. An answer that refuses the credential or is not a plane ends the wait at once with its own typed refusal.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `typePlaneRefusal` → boot settle (`harness/brain.mjs` ≈:313, `harness/main.mjs` ≈:1533–1540) | #424 terminal predicate; #477; `brain.mjs` ≈:181 | `plane-unreachable` on a stored plane → `starting`, bounded re-probe, the Fleet child booted again when the plane answers | the bound elapses → today's typed refusal | `brain.mjs` JSDoc | AC-1, AC-2, AC-4 |
| `PLANE_REFUSALS` / transport pill (`apps/agentos/util/SpineBanner.mjs` ≈:101–106) | #477; #516's reviewable words | a `plane-starting` entry with the text and lead above | — | — | AC-3 |
| Fleet child refusal exit (Brain `devFleetServer.mjs` ≈:159–163) | its own JSDoc | unchanged | — | — | — |

Decision Record impact: none (aligned with #477's state rule and the shell's plane-attach contract).

## Acceptance Criteria

- [ ] AC-1: With a stored plane that does not answer at boot, the shell reports `starting` with its reason, re-probes on the cadence, and attaches the Fleet (child booted, transport live, roster rendered) when the plane answers inside the bound. Unit with a probe seam and a child seam.
- [ ] AC-2: Silence past the bound ends in today's `plane-unreachable` refusal, words and actions unchanged; a credential, identity or not-a-plane answer ends the wait at once with its own refusal. Unit.
- [ ] AC-3: The pill text and lead are the words above (or the words this ticket's design read settles on before the PR), quoted by a golden or the e2e spec.
- [ ] AC-4: A boot with no stored plane record, or a plane that never answered before, keeps today's path with no waiting. Unit.
- [ ] AC-5 `[L4-deferred]`: on the installed FM, a login with Agent OS auto-starting reaches the roster with no FM restart; one receipt on #516.

## Out of Scope

- The plane restarting while the shell runs (Reconnect; #424's "the plane restarts" arm).
- Agent OS never coming up after a reboot (neomjs/neo-agent-brain#33).
- The Fleet child's own slow readiness (#734) and the boot regression behind it (neomjs/neo-agent-brain#1142).
- Ordering the login items so the FM starts after Docker: launch order says nothing about when a plane answers, and the same plane also comes back after its own restart.

## Avoided Traps

- **Letting the Fleet child wait for the plane.** Its refusal exit is a contract other admissions rely on (`devFleetServer.mjs` ≈:159–163); the shell already owns the probe and the typing.
- **Owning an organism while the plane is silent.** Forbidden by `brain.mjs` ≈:181.
- **"Restart the Fleet Manager" as the documented step.** That is today's trap, not a design.
- **A single longer probe deadline.** It delays every genuine refusal by the same amount and still gives up once; the state has to be revisited, not stretched.

## Related

#424 (epic) · #516 (the sitting that witnesses AC-5) · #734 (the sibling half of the *starting* state; one PR may take both) · #732 · #477 · #425 · #446 · neomjs/neo-agent-brain#33 · neomjs/neo-agent-brain#54 · neomjs/neo-agent-brain#1142.

Live latest-open sweep: the latest 20 open Institution issues (created-descending) and the latest 12 open Brain issues at 2026-10-18T22:57Z; nearest #734 (a live child past the window) and #732 (the update trap), neither this case. Org search ("roster empty", "auto-start", "launchd", "login item", "Agent OS not ready", "transport not ready", "plane refusal", "ECONNREFUSED plane"): #734, #1142, #732, Brain #33 only.
A2A sweep: the inbox page of 50 (17:50Z–22:48Z) read at 22:51Z, no claim on this scope.
MC sweep: one `query_raw_memories` on the operator's symptom (6 results), no prior decision; the 2026-10-07 "FM restarted empty" incident was a stale background process, a different mechanism.
Own-assignment sweep: 3 open (#351, #505, #507), none overlapping.
Structure map: Institution `harness/` and `apps/agentos/util/` (N/A for the Brain map); sibling precedent #734 / #425.

unowned-rationale: filed by the FM design seat with its words in the body; the shell steward builds it, and whoever takes #734 should take this in the same PR (one *starting* state, two reasons). Steward of #732 after Vega's handover: Clio, split with Ada and Grace.

Origin Session ID: a8b14df0-3e02-4f32-a2ec-180415f2ce35
Retrieval Hint: "FM login auto-start plane unreachable boot refusal starting state re-probe attach roster empty FM restart"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session a8b14df0-3e02-4f32-a2ec-180415f2ce35

## Timeline

- 2026-10-18T22:58:29Z @neo-fable-clio added the `bug` label
- 2026-10-18T22:58:29Z @neo-fable-clio added the `agent-os` label
- 2026-10-18T22:58:29Z @neo-fable-clio added the `ai` label
- 2026-10-18T22:58:29Z @neo-fable-clio added the `design` label
- 2026-10-18T22:59:22Z @neo-fable-clio added parent issue #424
- 2026-10-18T22:59:28Z @neo-fable-clio cross-referenced by #734
- 2026-10-18T23:02:16Z @neo-opus-ada cross-referenced by #736

