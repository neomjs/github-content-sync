---
id: 395
title: A sunset's background tasks and save order break its rotation
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-15T02:02:16Z'
updatedAt: '2026-11-15T03:05:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/395'
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
closedAt: '2026-11-15T03:05:34Z'
---
# A sunset's background tasks and save order break its rotation

## Context

Design authority: the operator's ruling (neomjs/neo-agent-brain#1722): a sunset is terminal, and a second live session of one seat *"violates 'never a copy'"*.

Two specimens from 2026-11-15:

**Grace: background tasks.** Her sunset handover went out at 01:41:58Z while three of her background subagents were still running (neomjs/neo-agent-brain#1739; times from her A2A `6f525ede`).
- They finished at 01:42:28Z, 01:42:37Z and 01:45:55Z, and each finish re-invoked the sunsetted session.
- The plane's fresh session, typed at 01:43:03Z, was already working by then.
- In those re-invoked turns the old session sent two more handover-tagged self-DMs, and at 01:46:35Z it posted a GitHub comment on neo `#20983`.

For about four minutes, the seat had two live sessions.

**Vega: save order.** Her old session planned to swap Steps 8 and 10, and its addendum turn did: `add_memory` at 02:28:29Z, the addendum at 02:28:42Z, and no save after it (her A2A `d81f5211`). The workflow orders Step 10 after Step 8 but never says why. The plane reads the writer's save after the receipt as the turn's close. Without one, it waits on presence alone, for up to an hour (neomjs/neo-agent-brain#1739 Avoided Traps).

## The Problem

1. **Background tasks.** In Claude Code, a finishing background task re-invokes its parent session: an `Agent` subagent, a `run_in_background` shell, a `Monitor`, or a `ScheduleWakeup`. The session-sunset workflow never checks for them, so a sunset can end its turn with work still scheduled to wake it. The plane requested Grace's rotation at 01:42:44Z, 7 s after the second finish had re-invoked the old session. Once a rotation is delivered, the plane's `decideRotation` only waits for a marker, so the only place to prevent the wake is the session that schedules the work.
2. **Save order.** Step 10's save is what closes the receipt's turn, but nothing in the step says so. That leaves the order open to "save first", which delays the rotation.

## The Architectural Reality

- `.agents/skills/session-sunset/references/session-sunset-workflow.md` Step 8 sends the continuity ping, which is the plane's sunset receipt. The plane acts on it once the turn's `add_memory` (Step 10) closes the turn.
- No step in the workflow mentions background tasks, and Step 10 does not name its role as the close. Neither does `SKILL.md` (1197 bytes, the router; it stays untouched).

## The Fix

- **Step 8** gains one precondition before the ping: no background task is in flight. Stop each one or wait for it to finish (Claude Code: `Agent`, `run_in_background`, `Monitor`, `ScheduleWakeup`), and say in the ping that none is left.
- **Step 10** says its save closes the receipt's turn: save after Step 8, and again after any addendum.

Accretion: the Step 8 line retires once the rotation itself ends the old session's background work, through D#19401's session binding or a harness-level close. The PR keeps the corpus inside its growth cap.

## Acceptance Criteria

- [ ] Step 8 carries the precondition and names the Claude Code task kinds that re-invoke a parent session.
- [ ] The ping states, in one clause, that no background task is left.
- [ ] Step 10 says its save closes the receipt's turn, after Step 8 and again after any addendum.
- [ ] The workflow's byte delta is ≤ 0, or the PR cites the retirement trigger.
- [ ] Post-merge (live): the next Claude sunset that ran background work shows no turn from its old session after the handover. *(Deferred to neomjs/neo-agent-brain#1739, which records it with its own live check; Sophie's review of #396.)*

## Out of Scope

- The plane's handling of amended handovers and of writers whose session changed (neomjs/neo-agent-brain#1739).
- Stopping background work from the plane or the receiver.
- Step 1's manual pulls (#289).

## Related

neomjs/neo-agent-brain#1739 · neomjs/neo-agent-brain#1722 · #289 · D#19401

Live latest-open + closed sweep: latest 20 open plus a closed search ("sunset background subagent task") at 2026-11-15T02:0xZ, no equivalent. Backlog 36 open (orange), and no open ticket owns the sunset's background-task gap; #289 covers Step 1 only. A2A: the latest 8 at 01:58Z, no claim. MC: a rationale query on the problem nouns, run at 02:02Z after filing; no prior art. Own assignments: none on this surface. create-skill read for placement: the change stays in the reference payload, not the router.
Labels: not `good first issue` / `help wanted`. It pairs with neomjs/neo-agent-brain#1739, which I own.

Retrieval Hint: "sunset background subagent task notification re-invokes parent session after rotation second live session precondition TaskStop add_memory closes receipt turn save order"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-15T02:02:16Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-15T02:02:18Z @neo-opus-ada added the `bug` label
- 2026-11-15T02:02:18Z @neo-opus-ada added the `ai` label
- 2026-11-15T02:02:18Z @neo-opus-ada added the `agent-os` label
- 2026-11-15T02:06:32Z @neo-opus-ada cross-referenced by #1739
- 2026-11-15T02:21:53Z @neo-opus-ada cross-referenced by PR #396
- 2026-11-15T02:28:46Z @neo-opus-ada referenced in commit `02997f9` - "fix(session-sunset): trim the precondition under the growth cap (#395)"
- 2026-11-15T02:45:08Z @neo-opus-ada referenced in commit `3e4ce8a` - "fix(session-sunset): the Step 10 save closes the receipt's turn (#395)"
- 2026-11-15T02:45:28Z @neo-opus-ada changed title from **A background task finishing after a sunset re-invokes the old session** to **A sunset's background tasks and save order break its rotation**
- 2026-11-15T03:05:34Z @tobiu referenced in commit `14ff7f6` - "fix(session-sunset): no background task outlives the sunset (#395) (#396)

* fix(session-sunset): no background task outlives the sunset (#395)

Step 8 now opens with a precondition: before the continuity ping, the session
stops every background task it started, or lets it finish, and says in the
ping that none is left. A task finishing later re-invokes the sunsetted session
beside the fresh one: Grace's 2026-11-15 sunset ran three subagents past its
handover, and the woken old session sent two handover amendments and a GitHub
comment while its successor worked.

Version 1.9.7 (npm has 1.9.6; #394 also bumps to 1.9.7, so whichever merges
second re-bumps).

* fix(session-sunset): trim the precondition under the growth cap (#395)

* fix(session-sunset): the Step 10 save closes the receipt's turn (#395)"
- 2026-11-15T03:05:35Z @tobiu closed this issue

