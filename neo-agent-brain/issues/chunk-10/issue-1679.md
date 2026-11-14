---
id: 1679
title: No process runs the Task expiry sweep on the plane
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-13T09:30:34Z'
updatedAt: '2026-11-14T02:20:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1679'
author: neo-opus-vega
commentsCount: 5
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1689 The Fleet''s questions read counts the asks that expired within the span'
blocking: []
closedAt: '2026-11-13T21:04:39Z'
---
# No process runs the Task expiry sweep on the plane

## Context

Found while building #1677, at the step that asks which process would send its expiry notice. On the canonical local plane a Task's `expiresAt` is never enforced: the sweep's one scheduled caller sits in a role that cannot open the graph the sweep updates.

The plane's own orchestrator log, read through `inspect_deployment` on 2026-11-13:

```text
[2026-11-12T20:01:29.760Z] [PID:7] [INFO] [Orchestrator] Started. authorityProfile=container-plane authorityReceipt=/app/.neo-ai-data/orchestrator-daemon/orchestrator-authority.json …
[2026-11-12T20:01:29.760Z] [PID:7] [WARN] [Orchestrator] Not running 2 lane(s) this role does not own — host-edge owns primary-dev-sync, swarm-heartbeat. This process does not verify that the owning role is live.
```

## The Problem

- **Nothing expires.** A probe Task sent to myself with `expiresAt` 2026-11-13T09:23:59Z still read `Submitted` at 09:29Z. That alone proves little, because the heartbeat's cadence is 20 minutes; I re-read it after a full cadence and add the reading here. My own inbox and outbox hold no `Expired` Task at all (`list_messages` with `taskStates: ['Expired']`, `includeArchived: true`: 0).
- **What depends on it:**
  - ADR 0042 §1.6 says of a `requestPublish` Task that "the Task envelope's `expiresAt` applies". On the plane it does not, so a queued publication request is never refused as expired.
  - #922's AC-4 (an expired ask leaves the open list and its count) holds in the unit suite only.
  - #1677 stamps a default deadline on every ask to a human. Without a sweep the stamp is written and never acted on, and neomjs/neo-agent-institution#951's count stays where it is.

## The Architectural Reality

- `MailboxService.sweepExpiredTasks` has one scheduled caller: step 2 of `SwarmHeartbeatService#pulse` (`ai/daemons/orchestrator/services/SwarmHeartbeatService.mjs:384`). The other caller is the manual CLI `ai/scripts/lifecycle/sweepExpiredTasks.mjs`.
- `'swarm-heartbeat'` is a `hostEdge` lane (`ai/daemons/orchestrator/taskAuthority.mjs:115`). `Orchestrator.mjs:1771` starts it only where the role owns it and `swarmHeartbeatEnabled` is on; that leaf defaults to `false`.
- Host-edge is graphless by contract (`ai/daemons/orchestrator/daemon.mjs:346`: "Host-edge roles must remain graphless"), and the sweep answers `{success: true, sweptCount: 0}` when it has no storage handle (`if (!db?.storage?.db)`). So even an enabled heartbeat on the host sweeps nothing, and reports success.
- The heartbeat's other steps (sunset and idle-out detection, harness resume) are host concerns. The sweep is plane maintenance that rode along and lost its graph at the hard cut (#84).
- Precedent for a small plane-side lane: `dated-items-overdue` (#1576): `ai/daemons/orchestrator/scheduling/datedItemsOverdue.mjs`, one row each in `taskAuthority.mjs` and `taskDefinitions.mjs`, class `containerPlane`.

## The Fix

Give the sweep an owner that has the graph:

1. A recurring `containerPlane` lane in the Orchestrator that calls `MailboxService.sweepExpiredTasks()` on its own cadence leaf, with the lane's row in `taskAuthority.mjs` and `taskDefinitions.mjs`.
2. `SwarmHeartbeatService#pulse` no longer sweeps, so one role owns the step.
3. The lane awaits the graph and throws when there is none, so the run is recorded as failed. `MailboxService.sweepExpiredTasks` alone answers `{success: true, sweptCount: 0}` without a graph, which is the silent success above.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Orchestrator lane for the Task expiry sweep (new) | ADR 0019 §10.8: one explicit owner per lane | Class `containerPlane`; runs `MailboxService.sweepExpiredTasks()` on its cadence | A failed run is logged and isolated; the next run sweeps what is due | JSDoc, the lane's config leaf | Unit |
| Cadence leaf for that lane (new) | ADR 0019 | One Tier-1 leaf under `orchestrator.intervals` | None | Config JSDoc | Unit |
| `Orchestrator#sweepExpiredTasks` (new) | This ticket | Awaits the graph, requires it, runs `MailboxService.sweepExpiredTasks()` | No graph: throws, and the lane records a failed run | JSDoc | Unit |
| `SwarmHeartbeatService#pulse` | This ticket | No longer calls the sweep | None | JSDoc | Unit |
| `orchestrator-authority.json` receipt | ADR 0019 §10.8 | Names the new lane with its class and owner | A role that does not own it logs it among the lanes it does not run | None | Unit: the topology census |

## Decision Record impact

`aligned-with` ADR 0019 §10.8: it adds one classified lane with exactly one owner. No ADR changes.

## Acceptance Criteria

- [ ] AC-1: the container-plane role runs the sweep on its cadence and the host-edge role does not (unit: authority map and scheduling).
- [ ] AC-2: after one lane run a Task past its `expiresAt` is `Expired`; a throwing sweep does not stop the Orchestrator's other lanes (unit).
- [ ] AC-3: the heartbeat pulse no longer calls the sweep (unit).
- [ ] AC-4: the topology audit and the authority receipt name the lane with its class (unit: the census spec).
- [ ] AC-5 `[post-merge]`: on the plane after the cut, a probe Task with a near deadline reads `Expired` within one cadence. The receipt goes on this ticket.

## Go-live precondition

The runner reaches the plane only after Home counts what expired (the design seat's consumer leaf in `neomjs/neo-agent-institution`), or after the operator declines that count in words ([comment](https://github.com/neomjs/neo-agent-brain/issues/1679#issuecomment-6861690712)). The PR can be reviewed before that; its merge waits for it.

## Out of Scope

- The heartbeat's other steps on a graphless host: they read the graph too (`GraphService.ready()`, `MailboxService.listMessages`). Their owner is a separate question.
- Which Tasks get a deadline: #1677.

## Avoided Traps

- Turning the heartbeat on in the container: it would pull host-only steps (harness resume, harness-state directories) into the plane.
- Expiring at read time: #859 rules that the read invents no deadline and that expiry stays the sweep's transition and event.
- A cron or CLI outside the Orchestrator: no owner in the authority receipt, which is the gap this ticket is about.

## Related

Blocks the post-merge AC of #1677 and through it neomjs/neo-agent-institution#951. History: #84, #859, #922, #1576.

Live latest-open + closed sweep: the latest 20 open of 160 (red band; no open ticket or epic owns Task expiry, and #84's hard cut is the origin, not an owner of this gap) and searches over open and closed ("sweepExpiredTasks", "expiry sweep host-edge", "swarm-heartbeat graphless", "Task never Expired plane TTL") at 2026-11-13T09:29Z. No equivalent. A2A sweep (12 newest rows, all read states): no claim on this surface; Grace's defect-note on `checkSunsetted.mjs` concerns another heartbeat step. Memory Core: the #1677 queries of this session, no prior decision. Own assignments: #1677 only on this surface.

Structure map: the owner is `ai/daemons/orchestrator` (`taskAuthority.mjs`, `taskDefinitions.mjs`, `scheduling/`); the cadence sits inline in `scheduling/registry.mjs`, as `defect-ledger-digest` and `ci-failure-ingest` do, so no new module.

Origin Session ID: d31ea882-d395-44a7-8af4-6d6be04e17c3
Retrieval Hint: "Task expiry sweep never runs on the plane · swarm-heartbeat host-edge graphless · sweepExpiredTasks owner · container-plane lane"

## Timeline

- 2026-11-13T09:30:34Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-13T09:30:35Z @neo-opus-vega added the `bug` label
- 2026-11-13T09:30:36Z @neo-opus-vega added the `ai` label
- 2026-11-13T09:30:36Z @neo-opus-vega added the `agent-os` label
- 2026-11-13T09:30:56Z @neo-opus-vega cross-referenced by #1677
- 2026-11-13T09:31:13Z @neo-opus-vega cross-referenced by #951
### @neo-opus-vega - 2026-11-13T13:28:07Z

**The probe's reading, as the body promises.**

Task `MESSAGE:3865717e-7ec6-4051-8c97-46e98ff57fdd` (to myself, `expiresAt` 2026-11-13T09:23:59.188Z) read through `get_message` at 2026-11-13T13:27:46Z:

```text
task: {state: "Submitted", expiresAt: "2026-11-13T09:23:59.188Z", assignee: "@neo-opus-vega"}
```

That is 4 h 03 min past its deadline, twelve of the heartbeat's 20-minute cadences. The Task never left `Submitted`, so the gap is not one of cadence: no process on the plane runs the sweep.

I leave the probe Task as it is. It is AC-5's subject: after the cut that carries this ticket's lane, the same Task has to read `Expired` within one cadence.

— Vega (Fable 5.2, Claude Code) 🌿

### @neo-opus-vega - 2026-11-13T13:36:14Z

**What the first run would do on our plane, measured before any build. It changes what the operator is shown, so the build waits for his word.**

Read-only counts in the orchestrator container, with the sweep's own predicate (the `selectCandidates` statement of `MailboxService.sweepExpiredTasks`), at 2026-11-13T13:31:41Z:

| Reading | Count |
| :--- | ---: |
| MESSAGE nodes | 69,365 |
| Non-terminal Tasks | 1,585 (1,541 without `expiresAt`, 44 with) |
| Past their `expiresAt`, so candidates of the first sweep | 44 (31 `Working`, 11 `InputRequired`, 2 `Submitted`) |
| Tasks already `Expired` | 5; the newest changed on 2026-07-22T00:03:09Z |

So the sweep last moved a Task here on July 22, and every Task that carries a deadline today is past it.

**The 44 by kind:**

- **31 `Working` broadcasts to `AGENT:*`:** lane claims, lane intents and night-shift leases with a stated end, sent between July 26 and November 6 (Emmy 16, Euclid 10, Sophie 5).
- **11 `InputRequired` asks to @tobiu**, each with a deadline its sender stated, one to two days after it was sent: one authorization from August 25 and ten from October 4 and 5 (eight merge handoffs, two questions; five from Grace, five from me, one from Emmy). The handoffs name nine pull requests I could read from the subjects, and all nine were merged on October 4 or 5; the August one names a pull request that was closed and replaced the same day. The asks were answered by the merge, and the Tasks stayed open.
- **2 `Submitted`:** one from July 24 to a benched seat, and my probe above.

**Wakes:** none. The plane holds 15 active wake subscriptions and all are `SENT_TO_ME` (read at 2026-11-13T13:35:27Z); a `task_state_changed` row reaches a seat only through a `TASK_STATE_CHANGED` subscription. The first run is silent, and so is every later expiry.

**What that means for the list the operator reads.** Home counts every non-terminal Task addressed to him (#922). After the first run those 11 are `Expired`: they leave the waiting count and stay readable in the Mailbox as expired. That is the contract #922's AC-4 states, and it is also the same kind of change that held #1677 this morning: something leaves the operator's view without his act. I wrote on #1677 that this ticket does not depend on his answer there. For 33 of the 44 that is true. For these 11 it is not.

**So, before I build:** @tobiu, question 1 on #1677 decides this ticket's first run too. May an ask to you that states a deadline leave Home's waiting list when the deadline has passed?

- **Yes:** I build this ticket as filed. The first run after the cut expires the 44 above. Clio's read on #1677 proposes a second number on Home for expired asks, so they move and do not vanish; whether that leaf lands before this ticket's cut is then the one ordering question.
- **No:** the sweep gets an exclusion for asks to a human principal, publication requests keep ADR 0042's rule, and this ticket's first run expires the other 33.

Until then this ticket is not built, and #1677's branch stays local.

— Vega (Fable 5.2, Claude Code) 🌿

- 2026-11-13T16:35:03Z @neo-opus-vega cross-referenced by #1383
- 2026-11-13T17:39:50Z @neo-opus-vega cross-referenced by PR #1684
### @neo-opus-vega - 2026-11-13T17:49:57Z

**The operator's word, 2026-11-13: yes. This ticket is no longer held and is built as filed (Vega, owner).**

- The rule itself is in review: #1684 (Resolves #1677), green at `a8c1e7cc3c`.
- This ticket is the runner that rule needs on the plane: a `containerPlane` lane beside `scheduling/datedItemsOverdue.mjs`, one row each in `taskAuthority.mjs` and `taskDefinitions.mjs`, a cadence leaf, and the step leaves `SwarmHeartbeatService#pulse`.
- The same structural cause surfaced a second time today in #1383: `swarm-heartbeat` is a host-edge lane, and what it needs to read is in the container.

Not started. It is my next Brain lane, from a fresh session.

— Vega (Fable 5.2, Claude Code) 🌿

### @neo-opus-vega - 2026-11-13T18:35:27Z

**Go-live precondition, added by the owner on 2026-11-13: the runner does not reach the plane before Home counts what expired.**

The operator's yes covered question 1 of #1677 (an ask to him may expire). Question 2 of the same comment, whether Home also counts what expired, has the design seat's yes and no answer from him ([receipt](https://github.com/neomjs/neo-agent-brain/issues/1677#issuecomment-6861670460)). The design seat's read names that count as the answer to the concern that held #1677's push: an expired ask moves from "waiting" to "expired unanswered" and stays in his view.

This ticket is the point where that concern becomes real. #1684 only defines the rule; the first run of the runner built here ends 11 of his open asks that state a deadline ([the reading](https://github.com/neomjs/neo-agent-brain/issues/1679#issuecomment-6854510486)), and with #1684 merged also every open ask to him that is older than the default span. So:

- The build of this ticket is not blocked.
- Its deployment is: the Home count (the consumer leaf in `neomjs/neo-agent-institution`, the design seat's) is installed first, or the operator declines the count in words.

I fold this into the body as a criterion when I start the build.

— Vega (Fable 5.2, Claude Code) 🌿


- 2026-11-13T20:29:17Z @neo-fable-clio cross-referenced by #959
- 2026-11-13T20:34:55Z @neo-opus-vega cross-referenced by PR #1687
- 2026-11-13T20:47:47Z @neo-opus-vega cross-referenced by #1689
- 2026-11-13T20:47:56Z @neo-opus-vega marked this issue as being blocked by #1689
- 2026-11-13T21:04:39Z @tobiu referenced in commit `60f011e` - "feat(orchestrator): the container plane runs the Task expiry sweep (#1679) (#1687)

The sweep's one scheduled caller was step 2 of the swarm heartbeat, a host-edge
lane that is graphless by contract, so no process expired a Task on the plane.
A `task-expiry-sweep` lane now runs it in the container-plane role on its own
cadence leaf (`orchestrator.intervals.taskExpirySweepMs`, ten minutes), without
the heavy-maintenance lease. `Orchestrator#sweepExpiredTasks` awaits the graph
and throws without one, where the sweep alone reports a successful sweep of
nothing. The heartbeat pulse no longer sweeps."
- 2026-11-13T21:04:41Z @tobiu closed this issue
- 2026-11-13T21:21:23Z @neo-opus-vega cross-referenced by PR #1691
### @neo-opus-vega - 2026-11-14T02:20:52Z

**AC-5 receipt (post-merge), plane `0a1f4cf9`, after Ada's cut (live 02:00:54Z):**

- **Probe:** a Task to myself, `MESSAGE:ad520fb4`, task id `probe:brain-1679-ac5-2026-11-14`, sent 2026-11-14T02:09:47.673Z with `expiresAt` 02:13:30Z.
- **Read at 02:16:07Z:** `Submitted` (the deadline had passed, the next sweep had not run).
- **Read after the next sweep:** `state: 'Expired'`, `expiredAt: 2026-11-14T02:18:06.826Z`, 4 min 37 s after the deadline. The cadence leaf is `orchestrator.intervals.taskExpirySweepMs` (10 min); the container sweep's first run was at 01:58:00Z, so this was its second.

The backlog cleared in that first run too: my ask `MESSAGE:36e0dff7` (deadline 2026-10-05) reads `Expired` with `expiredAt` 01:57:53Z, and its sender's notice (five asks, each with its message id) reached me at 01:58:10Z.

— Vega (Claude Opus 5.5, Claude Code) 🌿


