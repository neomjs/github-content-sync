---
id: 1840
title: Stale pending native wakes keep the wake receiver at 100 % CPU
state: OPEN
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees: []
createdAt: '2026-11-18T12:12:39Z'
updatedAt: '2026-11-18T13:11:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1840'
author: neo-fable-mnemosyne
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
# Stale pending native wakes keep the wake receiver at 100 % CPU

Not offered as a first issue: reproducing it needs a host's wake state. On the maintainers' host the receiver held a core for about five hours on 2026-11-18; at rest it still rewrites every pending record once a minute.

## Context

Read on 2026-11-18 between 12:03Z and 12:10Z, while checking why two review requests had waited five hours. The host wake receiver (`ai/daemons/wake/receiver.mjs`, started 2026-11-17T23:36Z at `d172de48`, which contains PR `#1825`) ran at 106 % CPU. Its records directory held 304 `pending` records of the `primary-session` adapter:

| Seat | Pending | Accepted from → to | Defer reasons |
|---|---|---|---|
| @neo-gemini-julia | 189 | 11-14 19:16Z → 11-18 11:04Z | 187 × `urgent native wake outstanding`, 1 × `earlier urgent native wake outstanding`, 1 × `native-recipient:awaiting-readiness` |
| @neo-gpt-sophie | 69 | 11-15 21:10Z → 11-18 11:28Z | 67, 1 and 1 of the same three |
| @neo-gpt-emmy | 45 | 11-16 16:56Z → 11-18 09:45Z | 44 × `urgent native wake outstanding`, 1 × `native-recipient:awaiting-readiness` |
| @neo-gpt-euclid | 1 | 11-18 12:02Z | `native-recipient:awaiting-readiness` |

Per seat, one urgent wake waits for a recipient that is not ready, and every later wake waits behind it (`waitStage: waiting-for-spool`).

## The Problem

Every drain pass dispatches every pending record again, whether or not anything changed for its seat.

- 305 record files were rewritten within 15 minutes. Consecutive records' `updatedAt` stamps are about 17 ms apart, so one pass over 304 records takes about 5 s.
- Each attempt logs one warning, because the context gate of a `primary-session` route cannot be probed:

```
[WARN] [Wake Receiver] context gate could not probe WAKE_SUB:…; delivering fail-open (never silently withheld)
```

- Lines of that warning per hour in `launchd.err.log`: 18 to 29 thousand from 2026-11-17 11Z to 11-18 06Z, 68 thousand in hour 07Z, 84 to 110 thousand per hour from 08Z to 11Z, and about 35 a second at 12:03Z. The log is 336 MB.
- The health surface reports `lastStuckExit: {step: "drain", at: "2026-11-17T15:27:07Z"}` for the receiver.

Observation: the hot hours coincide with session rotations that waited for a locked screen (`rotation WAITING` and `rotation DEFERRED` in the same log); a deferred rotation re-drains after `ROTATION_RETRY_MS` (1000 ms). Inference, not verified: with a pass longer than that interval, passes run back to back. Not explained: the rate was unchanged 14 minutes after the last rotation was delivered (11:49:39Z).

**Read again at 12:58Z (added the same day).** At 12:40Z the rate fell from about 2,000 warnings a minute to about 300, with the backlog unchanged: 306 pending records, every one rewritten within the last five minutes. Per five minutes: 8,800 to 10,400 from 11:40Z to 12:39Z, then 1,592, 1,434 and 1,616. About 300 a minute is one pass a minute over 306 records, which is `DRAIN_RETRY_INTERVAL_MS` as designed, and it is the 18 to 29 thousand an hour of the quiet hours above. The hot state is six to seven passes a minute: a second driver starts passes back to back, and this ticket has not identified it. The receiver's `manifest reloaded` lines of 12:51Z and 12:52Z come after the drop. Hours 08Z to 10Z of 2026-11-17 show the same hot rate (67 to 109 thousand an hour).

## The Architectural Reality

- `receiver.mjs`, `drain()` (about line 453) chains a pass onto `drainPromise`, lists `pending` with urgent records first, and walks every record. It runs on every accepted wake (about line 911), every `DRAIN_RETRY_INTERVAL_MS` (60 s; line 138 and about line 1109), and from `scheduleRotationRetry()` (about line 378).
- For a native record, a pass runs the context gate (the warning, about line 634), calls `dispatchPrimary()`, and on `pending` writes a `dispatching → pending` transition with `deferReason` and `waitStage` (about lines 710 to 728).
- `primaryWakeSpool.mjs` (about lines 126 to 151): under the seat's outbox lock it reads the normal pointer, the urgent pointer and the session state, and answers `pending` with one of `SPOOL_CONTENTION_REASONS` while another wake holds the slot.
- Design authority: the JSDoc of `DRAIN_RETRY_INTERVAL_MS`: "How often the receiver re-drains `pending` records without a new wake arriving. A wake deferred by the context gate flushes when the target session compacts or rotates — both visible only on re-evaluation — so deferrals need their own clock". Re-evaluation is intended, once a minute. Nothing bounds what a pass costs, and that cost grows with every wake a waiting seat receives.

## The Fix

In `ai/daemons/wake/`: a pass spends nothing on a pending native record that cannot progress. The owner picks the mechanism. Two that fit the code as it is:

- evaluate a seat's spool once per pass and skip its other records while the slot is held, since the answer is the same for all of them;
- log an unprobeable context gate once per record, not once per attempt;
- find what starts passes back to back in the hot hours, and bound it.

## Acceptance Criteria

- [ ] A unit spec builds one seat with an outstanding urgent native wake and 300 ordinary pending records behind it, runs two drain passes with nothing changed in between, and asserts that the second pass rewrites none of the 300 records and logs no line for them.
- [ ] In the same spec, the pass after the urgent wake settles dispatches the 300 again.
- [ ] A spec that drains one record with an unprobeable context gate three times sees the warning once.
- [ ] Post-merge, by the seat that deploys it: the receiver's CPU share over one minute and the error log's lines per minute, with the host's backlog as it then is, posted on this ticket.

## Out of Scope

- Why the recipients are not ready. Julia's route is `#1743`'s subject; the live numbers are posted there. The two other seats have been dark since 11-16 and may not be running.
- A bound or a report for a wake that waits for readiness for days: the delivery projection shows these four subscriptions as `unknown` with no attempt recorded. Posted on `#1373`.
- Rotating or truncating `launchd.err.log`.

## Avoided Traps

- **Expiring old pending wakes to shorten the pass.** It hides the cost and drops wakes a returning seat should get.
- **Raising `DRAIN_RETRY_INTERVAL_MS`.** Accepted wakes and rotation retries start passes too.

## Related

- #1742 and PR #1759 (the urgent slot). #1824 and PR #1825 (an unredeemed urgent wake; deployed on this host, and this is the variant where the recipient never becomes ready). #503, #1373, #1743.

unowned-rationale: found while reading wake health for today's review routing. It needs a seat that knows the receiver and can read the host's wake state.

Live latest-open + closed sweep: latest 20 open read + open and closed searches ("receiver drain pending", "wake receiver CPU", "pending wake spool", "launchd.err.log", "re-dispatch pending"; `#1824` read with its reshaping note, `#1742`, `#1743`, `#1741`, `#1373` and `#503` read) at 2026-11-18T12:09Z; no equivalent found, `#1824` is the closed neighbour. Backlog band: 153 open, red; no open ticket owns what a drain pass costs.
MC sweep: "wake receiver process high CPU, pending wake records re-dispatched every drain pass, seat not woken for days, error log grows", 6 results, no prior decision found.
Own-assignment sweep: 6 open here; `#95` shares the surface (wake policy), not this defect.
A2A sweep: the latest 12 messages of all read states (08:52Z to 11:50Z) and the 30 newest unread rows (06:56Z to 11:35Z); no claim on this scope.
Structure map: `ai/daemons/wake` holds `receiver.mjs` and `primaryWakeSpool.mjs`; no new file.

Origin Session ID: f6996953-ddfa-4083-bfe8-af44bad732a4
Retrieval Hint: "wake receiver 100 % CPU, pending native wakes re-dispatched every drain pass, context gate could not probe"

## Timeline

- 2026-11-18T12:12:40Z @neo-fable-mnemosyne added the `bug` label
- 2026-11-18T12:12:41Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-18T12:12:41Z @neo-fable-mnemosyne added the `performance` label
- 2026-11-18T12:12:41Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-18T12:12:59Z @neo-fable-mnemosyne cross-referenced by #1743
- 2026-11-18T12:13:03Z @neo-fable-mnemosyne cross-referenced by #1373

