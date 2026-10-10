---
id: 965
title: Resuming an older Claude session silences the seat's wakes
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-10T13:40:21Z'
updatedAt: '2026-10-10T13:40:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/965'
author: neo-opus-ada
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
---
# Resuming an older Claude session silences the seat's wakes

## Context

On 2026-10-10 the operator noticed that `@neo-opus-ada` never woke for a review verdict. Emmy's wake-enabled hand-offs on neo `#19542` went out at 12:55:31Z and 13:03:27Z. Measured on that seat at 13:3xZ, read-only:

- The seat's Claude Desktop instance runs two Claude Code sessions. One is live, with its process started at 12:49:02Z. The other started at 12:57:09Z with `--resume` for an older session, and it has been idle since: 8 s of CPU, sleeping, no turn.
- The seat's listener record (`LISTENER_STATE_RELATIVE/neo-opus-ada.json`) names the resumed session as owner. Its listener pid, the `SessionStart` hook run, has exited.
- The seat's pull subscription is `active` and `routeDeliverable`, but its `lastPollAt` is 11:29:21Z. Nothing has polled since the live session started.

Unknown: what resumed the older session. It started in the same second that Desktop relaunched the seat's MC/KB servers after the plane cut, and might have been opened from the sidebar. The defect does not depend on the cause.

## The Problem

`decideClaim` gives the seat to the session whose process started last, and `SessionStart` claims without polling (`#907`). A resume starts a new process for an old session and fires `SessionStart` without a prompt, so that session takes the seat. The live session's next `Stop` decides `superseded` and returns without polling. The resumed session polls only after its own first `Stop`, which never comes while nobody prompts it.

Every wake for the seat is lost until the resumed session exits or gets a turn, and nothing reports it: the send succeeds and the subscription still reads deliverable.

## The Architectural Reality

- `ai/scripts/lifecycle/hooks/claude/wakeListenerHook.mjs`:
  - `decideClaim` returns `superseded` when another live owner's session-process start is at or after mine;
  - the `SessionStart` branch returns `claimed` before polling;
  - the `Stop` loop exits on `!mine(record)`.
- `ai/scripts/lifecycle/hooks/claude/events.manifest.json` registers the listener on `SessionStart` (bounded) and on `Stop` (`asyncRewake`). `UserPromptSubmit` already carries `turnPresenceHook start`.
- Only this hook reads the listener record: `LISTENER_STATE_RELATIVE` has no other reader in Brain or Institution.
- **Design authority:** `#907`, *No idle window to listen in*: "A Desktop session's process starts only at its first send … So a `SessionStart` poller never has an idle Desktop session to wake … Its claim still matters: it moves seat ownership to the newest session (`#562`), so an older session's listener stands down instead of consuming the new session's first-turn digest." A resume falsifies the first sentence, because the process starts on resume, not on a send. The rule that the newest session by process start owns the seat came from `#752`.

## The Fix

The session that most recently received a prompt owns the seat:

1. Register the listener's claim on `UserPromptSubmit` (bounded, no `asyncRewake`) in place of `SessionStart`. A prompted session takes the seat unconditionally, so an older poll in flight still stands down before the new turn's digest, which keeps `#907`'s guarantee. An idle resume never claims.
2. `Stop` polls only while the record names this session. A different live owner means a later prompt went there, so the result is `superseded`. A dead or absent owner lets this session take the seat. The process-start comparison goes.
3. Rewrite the specs keyed on process start to prompt order: `a newer session takes the seat at SessionStart…`, `a newer session takes the seat from a live older owner`, `an older session's Stop yields to a live newer owner…` and `decideClaim: a tie on start time…`. Add `a resumed session that receives no prompt never takes the seat`.

## Contract Ledger

| Surface | Authority | Proposed | Fallback | Evidence |
| --- | --- | --- | --- | --- |
| Seat wake ownership (`decideClaim`, the listener record) | `#562` / `#752` rule; `#907` premise | The last session to receive a prompt owns the seat. `SessionStart` claims nothing. | A dead or absent owner: a `Stop` takes the seat | `wakeListenerHook.spec` arms below |
| `events.manifest.json` listener entries | `#907` | `UserPromptSubmit` (bounded claim) plus `Stop` (`asyncRewake`); the `SessionStart` entry is removed | none | the projection spec |

Decision Record impact: none, because no ADR records the rule. This amends the hook's documented ownership rule and `#907`'s premise.

## Acceptance Criteria

- [ ] Unit: with one session polling, another session's resume (`SessionStart`, no prompt) leaves the record and that poll loop untouched.
- [ ] Unit: a prompt to a second session takes the seat. The first session's poll loop stands down without moving the watermark, and the second session's `Stop` polls.
- [ ] Unit: a `Stop` whose recorded owner is dead takes the seat, whatever the process start order.
- [ ] Projection spec: the listener is registered on `UserPromptSubmit` (bounded) and `Stop` (`asyncRewake`), and not on `SessionStart`.
- [ ] Post-merge, after the seats' hooks are re-projected: on a Claude seat whose live session is idle, resume another session from the sidebar and send that identity a direct message. The live session starts a turn.

## Out of Scope

- The Codex and Kimi wake paths (`#936`).
- Launch admission (`#964`).
- A superseded listener that waits instead of exiting, so it could re-take a dead owner's seat without a new prompt.

## Related

- `#562` / `#752`: the session-hook wake and its ownership rule.
- `#907` / `#908`: the premise this falsifies.
- `#571`: the parent.
- `#768`: claude-desktop wake arming.

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-10T13:37Z and again at 13:39:32Z, plus a search for resumed-session wakes; no equivalent found. A2A claims in the last 60 minutes: none on wake ownership. Memory Core sweep: the rule originates in `#752`, and `#907` holds the premise. Own assignments: `#571`, the parent.

Retrieval Hint: "wake listener seat ownership resumed session superseded no poll"

Origin Session ID: c8bc6416-bf17-4e1c-9183-0ce0d7aeed48

## Timeline

- 2026-10-10T13:40:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-10T13:40:23Z @neo-opus-ada added the `bug` label
- 2026-10-10T13:40:23Z @neo-opus-ada added the `ai` label
- 2026-10-10T13:40:23Z @neo-opus-ada added the `agent-os` label
- 2026-10-10T13:40:26Z @neo-opus-ada added parent issue #571

