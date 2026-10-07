---
id: 907
title: The Claude wake listener blocks a Claude Desktop session's first prompt
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-06T21:13:11Z'
updatedAt: '2026-10-06T22:12:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/907'
author: neo-opus-grace
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-06T22:12:29Z'
---
# The Claude wake listener blocks a Claude Desktop session's first prompt

## Context

This came up in Ada's FM pilot on 2026-10-06, with the operator present. An FM-launched Claude Desktop seat never runs its first prompt. The listener that `#752` registers on `SessionStart` holds the session's start until it exits, which only happens on a wake digest or after its 86400 s timeout. Defect-notes `60b4c79f` and `6a4d5da7` carry the same witness.

## The Problem

Witness: session `local_59201fdb`, Claude Code 2.1.288 under Desktop. The running argv is `<profile>/claude-code/2.1.288/…/claude`. Desktop's `main.log` times are local:
- **22:47:32** — the first prompt arrives. Claude Code goes idle (0 % CPU) and writes no transcript. The projected `wakeListenerHook` has been running since session start. A second prompt only queues, and the operator cancels it.
- **22:57:04** — Desktop logs `initializationResult read failed` and clears the session. **22:57:26:** the cold restart reaches the same state, with a new listener (pid 33864).
- **22:58:37** — only that listener's pid is killed. **22:58:38:** `Mapping internal session … to CLI session 6179858d…`, and the transcript starts. Desktop's start-timing reads `init=71908ms first_assistant=6712ms`.
- **Control:** after that turn ended, the `Stop`-registered listener (pid 39859) ran in the background as designed. Only the `SessionStart` registration blocks.

## The Architectural Reality

**Design authority:** the `events.manifest.json` `$comment` says *"wakeListenerHook runs with asyncRewake: it polls in the background and exit 2 wakes the session with the digest."* At `SessionStart` under Desktop it does not run in the background, so the behavior contradicts its own record.

- **The harness contract.** Per Claude Code's hooks reference (`#sessionstart`), the first response waits for `SessionStart` hooks, and those hooks should stay fast. In the bundled 2.1.288, an `asyncRewake` hook is backgrounded only when a runtime condition holds: `(!isInteractive() → hasStreamingInput()) && !<unresolved flag>`. During Desktop's SDK initialization it evidently ran in the foreground, as the timing above shows.
- **No idle window to listen in.** A Desktop session's process starts only at its first send (`sendMessage on uninitialized session; cold-starting via startSession`). So a `SessionStart` poller never has an idle Desktop session to wake; it can only delay the prompt that started it. Its claim still matters: it moves seat ownership to the newest session (`#562`), so an older session's listener stands down instead of consuming the new session's first-turn digest.
- **The watermark carries over on the same plane.** The per-seat listener record keeps `watermark` across sessions; `writeRecord` resets it only when `record?.source !== source`. So when a watermark exists, the first `Stop` poll continues from where the previous session stopped. A seat's first listener, or a changed plane, records a baseline at that poll instead of replaying earlier events.
- **Same class as #757.** That was the turn-presence hook blocking every prompt, closed by `#758`, which moved it off the blocking path.
- **Surfaces:**
  - `ai/scripts/lifecycle/hooks/claude/events.manifest.json` (`SessionStart`, `Stop`, `$comment`)
  - `ai/scripts/lifecycle/hooks/claude/wakeListenerHook.mjs` (the `SessionStart`-only head-start `sleep`, plus its docs)
  - the projection spec for `#562` AC-4

## The Fix

1. In `events.manifest.json`, make `SessionStart`'s `wakeListenerHook` entry bounded: no `asyncRewake`, timeout 10. Keep `Stop`'s `asyncRewake` entry, and rewrite the `$comment` to say why.
2. On `SessionStart` the listener runs only its existing claim and exits (`claimed`). Delete the head-start `sleep`.
3. Specs: the projection spec checks `Stop` with `asyncRewake` and SessionStart once, with no background flag and a bounded timeout. A listener overlap control: an older poll is in flight when a newer session claims; the older one then stands down without moving the watermark, and the newer session's first `Stop` gets the digest.

One engine line is added (the claim-only return); the sleep and its comment go.

*Corrected 2026-10-06 after review RA-1 on #908: the first version removed the `SessionStart` entry entirely, which dropped the ownership claim.*

| Target surface | Source of authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| `events.manifest.json` `SessionStart` | this ticket; `#562` | a claim-only `wakeListenerHook` entry: no `asyncRewake`, timeout 10; it takes the seat for the newest session and exits | a seat not yet re-projected keeps the old wiring: kill that session's listener pid | AC-1, AC-2 |
| `events.manifest.json` `Stop` | `#562` | unchanged: `asyncRewake`, timeout 86400 | — | AC-2 |

**Decision Record impact:** none. Aligned with ADR 0002, as `#752` was.

## Acceptance Criteria

- [ ] AC-1 (unit): the projected seat settings carry `wakeListenerHook` on `Stop` with `asyncRewake`, and on `SessionStart` once, with no background flag and a bounded timeout.
- [ ] AC-2 (unit): the listener's existing `Stop` behavior specs stay green. On `SessionStart` it claims and exits without connecting or polling, and an older poll in flight then stands down without advancing the watermark.
- [ ] AC-3 (post-merge, L4, operator present): a re-projected FM Claude Desktop seat runs its first prompt without intervention, and after that turn a background listener is running. Residual-Owner: `#571`.

## Out of Scope

- The Desktop-profile MCP carrier (D19437).
- The Engine-owned `rgReplaceGuardHook` command path (ADR 0040 §2.7, the Engine template).
- The MCP transport session id (defect-note `29cb8474`).
- A CLI-only `SessionStart` listener: the roster has no `claude-code` CLI seats.

## Avoided Traps

- **A `SessionStart` baseline poll.** Emmy raised the bounded-arm candidate. The claim half is adopted; the poll half is not. A poll keeps a plane call in the first-response path and races `wakeArmingHook`'s subscribe, which is why the `sleep` existed. Its only gain is the first turn of a seat's first listener on a plane.
- **Hand-editing a seat's generated `.claude/settings.json`.** The projector owns it, so the pilot mitigation is a targeted pid kill until the seat is re-projected.

## Related

`#562` · PR `#752` · `#757`/`#758` (same class) · `#571` (parent) · `#768` · D19437

Live latest-open sweep: checked the latest 20 open Brain issues at 21:10:53Z and re-ran at 21:12:42Z; no equivalent. A2A claim sweep: no claim on this scope. MC rationale sweep: no prior decision. Own assignments: `#904`, `#684`, `#18`, `#54`, none overlapping. Structure: no new file; the owning folder is `ai/scripts/lifecycle/hooks/claude/`.

Origin Session ID: c1461533-f31d-4846-8e11-cc7500b5e6e9
Retrieval Hint: `query_raw_memories("SessionStart wakeListenerHook asyncRewake Claude Desktop initialization blocked first prompt")`


## Timeline

- 2026-10-06T21:13:12Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-06T21:13:13Z @neo-opus-grace added the `bug` label
- 2026-10-06T21:13:14Z @neo-opus-grace added the `ai` label
- 2026-10-06T21:13:14Z @neo-opus-grace added the `agent-os` label
- 2026-10-06T21:13:21Z @neo-opus-grace added parent issue #571
- 2026-10-06T21:19:12Z @neo-opus-grace cross-referenced by PR #908
- 2026-10-06T21:50:32Z @neo-opus-grace referenced in commit `8d6562a` - "fix(hooks): a new session still takes the seat at SessionStart, without polling there (#907)

The SessionStart registration also moved seat ownership to the newest
session, so an older session's listener stood down instead of consuming the
next digest during the new session's first turn. SessionStart now runs the
listener's claim only and exits, bounded and without asyncRewake; Stop keeps
the background poll."
- 2026-10-06T22:12:29Z @tobiu referenced in commit `f5ee2bc` - "fix(hooks): the Claude wake listener polls only on Stop, so a Desktop session's first prompt runs (#907) (#908)

* fix(hooks): the Claude wake listener rides Stop only, so a Desktop session's first prompt runs (#907)

SessionStart holds the first response until its hooks finish. Under Claude
Desktop the asyncRewake listener registered there was not backgrounded during
SDK initialization, so a seat's first prompt waited until a wake arrived or the
86400 s timeout passed (Ada's pilot: init=71908ms until the listener was
killed). A Desktop session starts at its first send, so SessionStart has no
idle window to listen in, and the per-seat watermark already carries across
sessions. Stop keeps the listener; the SessionStart head-start sleep is gone.

* fix(hooks): a new session still takes the seat at SessionStart, without polling there (#907)

The SessionStart registration also moved seat ownership to the newest
session, so an older session's listener stood down instead of consuming the
next digest during the new session's first turn. SessionStart now runs the
listener's claim only and exits, bounded and without asyncRewake; Stop keeps
the background poll."
- 2026-10-06T22:12:29Z @tobiu closed this issue

