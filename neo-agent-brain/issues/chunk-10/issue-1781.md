---
id: 1781
title: A wake receiver skip records no reason; its log lines carry no time
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-16T02:54:57Z'
updatedAt: '2026-11-16T08:16:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1781'
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
closedAt: '2026-11-16T08:16:45Z'
---
# A wake receiver skip records no reason; its log lines carry no time

## Context

On 2026-11-15 the plane requested `@neo-opus-grace`'s rotation for receipt `a244dd95`, sent at 23:41:38Z. The host receiver's record for event `session-rotate:MESSAGE:a244dd95-5805-4755-abaa-0240f9e6f54c` (route `WAKE_SUB:a4f6a0c9…`), read on 2026-11-16:

| Field | Value |
|---|---|
| `state` | `skipped` |
| `acceptedAt` | 23:42:48.327Z |
| `dispatchStartedAt` → `dispatchFinishedAt` | 23:43:02.386Z → 23:43:02.502Z (116 ms) |
| `outcomeReason` | absent |

No session opened. The operator started Grace by hand, and her successor marker followed at 23:46:44Z. The deploy at the time was `d6e65d75`; the same return paths exist on `dev` `1f884fed`.

**Observed:** the record names no cause. The receiver's `launchd.err.log` lines carry no timestamp, so no line can be tied to this dispatch, and a skipped rotation writes no line of its own.

**Inferred from the code and one timing:** the dialog probe spawns `osascript` to query System Events, and such a query took 206–328 ms on this host (three runs, 2026-11-16). A 116 ms dispatch therefore ended before the probe, at one of the early bare `'skipped'` returns below. Of those, only the instance-pid lookup can fail on one attempt and pass on another, and its `catch` drops the resolver's error.

## The Problem

A skip is the one receiver outcome that tells nobody why: there is no reason on the record and no line in the log. For a rotation, that leaves a seat dark at night with nothing to diagnose. The operator ruled that out on 2026-11-16: post-sunset restarts have to work on night shifts while he is away. #1083 left "Recording a skip's reason on the receiver record" out of scope, and no ticket picked it up.

## The Architectural Reality

- `ai/daemons/wake/localWakeAdapters.mjs` returns a bare `'skipped'` at eight sites:
  - `performDispatch`: Codex app-server unconfigured, tmux session missing, unknown adapter;
  - `deliverOsascript`: not darwin, unsupported app, Codex focus seed missing, the `resolveGuiInstancePid` catch, and an ambiguous or unprobed default target.
- `ai/daemons/wake/instanceResolver.mjs` `resolveGuiInstancePid` throws named errors: not local, invalid pid, `No running <app> instance found for userDataDir=…`, unsupported address type.
- `ai/daemons/wake/receiver.mjs` already copies an object result's `outcomeReason` onto the record, so `{outcome: 'skipped', outcomeReason}` needs no receiver change. Its entrypoint hands `console` to `startWakeReceiver`, so the launchd log has no time. A terminal rotation writes a line only on some failure paths.
- Readers: `ai/services/memory-core/wakeDeliveryProjection.mjs` keeps skips transparent and takes `lastOutcomeReason` from failure states only, so a skip's reason changes no delivery verdict (#1083's Avoided Trap holds). No code parses the receiver's log lines.

Design authority: `spawnOsascriptOnce` in the same file: "Reporting the id alone leaves an operator with a confident line and no cause, which is harder to notice than silence. Carried on the record too: a receiver under launchd writes stdout where nobody reads it."

## The Fix

1. Each of the eight skip sites returns `{outcome: 'skipped', outcomeReason: '<named cause>'}`. The pid-lookup catch carries the resolver's message, bounded.
2. The receiver entrypoint's logger prefixes every line with an ISO time.
3. A terminal rotation record writes one receiver line with its state, route and reason.

## Acceptance Criteria

- [ ] Unit, red on `dev` first: a rotation whose instance lookup throws ends `skipped` with an `outcomeReason` carrying the resolver's message.
- [ ] Unit: each other skip site yields a named `outcomeReason`.
- [ ] Unit: the entrypoint logger prefixes an ISO timestamp.
- [ ] Unit: a terminal rotation record (delivered, skipped or failed) writes one line with its reason.
- [ ] Unit: `wakeDeliveryProjection` verdicts are unchanged for a skip that now carries a reason.
- [ ] Post-merge, after the cut: the receiver's next rotation lines carry timestamps.

## Out of Scope

- Why Grace's instance was not found at 23:43Z. The next occurrence names it.
- The plane's silent waits and repeat-sunset attribution (#1777).
- `ai/daemons/wake/daemon.mjs`'s own skip paths, which already log at the branch.

## Avoided Traps

- **Turning a skip into a failure to give it a reason.** It would move `consecutiveFailures` and alarm healthy seats (#1083).
- **Reading reasons back out of the log.** The record is the receiver's account; the log line is for people.

## Related

#1083 (its Out of Scope) · #503 (delivery projection) · #1770 / PR #1771 (the receiver's rotation wait) · #1777 · defect note `ebb63037` (A2A, 2026-11-16), promoted here by triage under the operator's night-shift bar

Live latest-open + closed sweep: latest 20 open, plus open searches (`wake receiver skipped`, `receiver log timestamp`, `skipped reason receiver`) and closed searches (`wake receiver skipped reason`, `outcomeReason adapter`), 02:46Z, re-checked 02:54Z. Out of Scope and Avoided Traps read on #598 and #1083; #1083 defers exactly this. No open equivalent: #503 owns the plane-side projection, not the receiver's write. Backlog band: 160 open (red).
A2A in-flight sweep: latest 30 messages in all read states, re-checked at 02:54Z. No receiver claim; Maryam's #1738 / PR #1780 touches fleet setup files only.
MC sweep: "wake receiver rotation ended skipped with no outcome reason…": 5 results, no prior decision.
Own-assignment sweep: 4 open (#1531, #571, #133, #1508), none overlapping.
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): `ai/daemons/wake` owns the change; no new file.

Ownership: `@neo-opus-ada`, self-assigned at creation as part of the night-shift rotation lane.

Origin Session ID: f19f77db-e206-4ada-af1b-1a46b677a056

Retrieval Hint: "wake receiver skipped rotation no outcome reason" · "receiver launchd log no timestamps"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-16T02:54:59Z @neo-opus-ada added the `bug` label
- 2026-11-16T02:54:59Z @neo-opus-ada added the `ai` label
- 2026-11-16T02:54:59Z @neo-opus-ada added the `agent-os` label
- 2026-11-16T02:55:05Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-16T03:09:03Z @neo-opus-ada referenced in commit `53fc389` - "test(wake): a failed rotation writes its line too (#1781)"
- 2026-11-16T03:11:38Z @neo-opus-ada cross-referenced by PR #1784
- 2026-11-16T06:16:11Z @neo-opus-ada cross-referenced by #1777
- 2026-11-16T08:16:44Z @tobiu referenced in commit `179e0ff` - "feat(wake): skips carry their reason, receiver lines a time (#1781) (#1784)

* feat(wake): skips carry their reason, receiver lines a time (#1781)

Grace's rotation of 2026-11-15 23:42Z ended `skipped` after 116 ms, with no
reason on its record and no timed line in the receiver's log: the adapter's
instance-lookup catch dropped the resolver's error, and the receiver logged
through a bare console under launchd.

- localWakeAdapters: every skip returns {outcome: 'skipped', outcomeReason};
  the lookup skip keeps the resolver's sentence, bounded to 240 chars.
- receiver: the entrypoint logger stamps [ISO] [PID:n] [LEVEL] like the other
  daemons, the adapters log through it, and each terminal rotation writes one
  line with its reason.
- The delivery projection still reads skips as transparent (#1083's trap).

* test(wake): a failed rotation writes its line too (#1781)"
- 2026-11-16T08:16:46Z @tobiu closed this issue
- 2026-11-16T08:16:49Z @neo-opus-ada cross-referenced by #1794

