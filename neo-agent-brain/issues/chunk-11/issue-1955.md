---
id: 1955
title: Most long mc-server event-loop blocks name no tool call
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
assignees:
  - neo-opus-ada
createdAt: '2026-11-27T11:50:37Z'
updatedAt: '2026-11-27T16:00:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1955'
author: neo-opus-ada
commentsCount: 2
parentIssue: 1843
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-27T15:54:14Z'
---
# Most long mc-server event-loop blocks name no tool call

Unowned: a sub of #1843 that #1940's profile named. Ready for any maintainer.

## Context

#1940's 40 h probe receipt ([comment](https://github.com/neomjs/neo-agent-brain/issues/1940#issuecomment-7217517914)) found 93 event-loop blocks of 10 s or more. **57 of them had no tool call in flight**, and 32 of those fell on 11-27 between 09:00 and 11:59Z, while seats came online. The probe cannot say what ran, so the largest share of the plane's blocked time has no name.

## The Problem

`createEventLoopBlockProbe` (`ai/services/memory-core/helpers/eventLoopBlockProbe.mjs:48`) attributes a block only to entries in its `calls` map, filtered at `:67`. Only MCP tool calls register there. In-process background work registers nothing, so its blocks report `callIds: []`: WAL drain and re-embedding, REM pipeline stages, graph and wake maintenance, scheduled sweeps.

## The Fix

Register each scheduled or background job with the probe while it runs: a pseudo-call `job:<name>` with `startedAt`/`endedAt` through the same map. `eventLoopBlocks` then names it like a tool call. Find the job entry points by reading which timers and loops mc-server starts. The inventory is part of the deliverable.

## Acceptance Criteria

- [ ] A unit spec: a block inside a registered job reports that job's id, and a block inside an unregistered timer still reports `[]`.
- [ ] Every interval and loop mc-server starts at boot is either registered or listed in the class JSDoc with why it cannot block.
- [ ] Post-merge: a 24 h probe read where blocks of 10 s or more with empty `callIds` are fewer than 10 % of all blocks of 10 s or more.

## Out of Scope

- Fixing whatever the new names reveal: each named job gets its own sub of #1843.
- The mailbox read blocks: the sibling sub.

## Related

Parent #1843 · Profile #1940 · Instrument #1933 / PR #1937

Live latest-open + closed sweep: as for the sibling mailbox sub at 2026-11-27T11:52Z. #1933 shipped the tool-call attribution this extends. Structure map: `ai/services/memory-core/helpers/eventLoopBlockProbe.mjs` and its spec.

Retrieval Hint: "event loop block probe background job attribution empty callIds"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-27T11:50:38Z @neo-opus-ada added the `enhancement` label
- 2026-11-27T11:50:38Z @neo-opus-ada added the `ai` label
- 2026-11-27T11:50:39Z @neo-opus-ada added the `performance` label
- 2026-11-27T11:50:54Z @neo-opus-ada added parent issue #1843
- 2026-11-27T11:51:32Z @neo-opus-ada cross-referenced by #1940
- 2026-11-27T14:38:04Z @neo-opus-ada cross-referenced by #1843
- 2026-11-27T14:42:02Z @neo-opus-ada assigned to @neo-opus-ada
### @neo-opus-ada - 2026-11-27T14:46:37Z

**Intake and a candidate name, before the instrument exists.** I'm taking this, branch `ada/1955-job-block-attribution`. I filed the ticket before a compaction, so I ran the drift probe: only `MailboxService.mjs` moved since filing (#1956), so the surface is unchanged. Prescription checked: `ai/services/memory-core/helpers/eventLoopBlockProbe.mjs` owns the concern. Its `calls` map takes any id, and no central scheduler exists that could own it instead, because mc-server's timers are spread across modules.

**Leading candidate: the message-graph integrity repair cadence.** `createMessageGraphIntegrityRepairCadence` (`ai/daemons/message/drainCycle.mjs:93`) runs `repairMessageGraphIntegrity({box: 'all'})` after a message-drain cycle, at least 60 s after its previous run *ended*. Each pass classifies every projected WAL id (`classifyMailboxGraphProjectionCandidates`, `MailboxService.mjs:2411`). The evidence so far is circumstantial:

- **Spacing:** on `055b2614`, the no-call blocks of 6.7–10.9 s ended at 14:37:02, 14:38:10, 14:40:35, 14:41:45, 14:42:56 and 14:44:03Z, which is 67–71 s apart. That fits "60 s after the run ends, gated on the 5 s drain poll". It does not fit the two fixed `setInterval`s at 60 s (`graphProjectionDrainIntervalMs`, `embeddingWriteCanaryCadenceMs`).
- **Boot:** the first repair pass after the cut logged at 14:33:30.460Z, and a no-call block of 8,027 ms ended at 14:33:30.
- **Rate:** the repair's hourly digest for 13:15:28–14:15:56Z counts 53 passes, all clean, each with `scanned: 0` and `candidateCount: 0`. In the same hour the file log holds 41 no-call blocks of 5 s or more, plus 22 blocks of 5 s or more that had calls in flight.

That is a correlation, not an attribution. The instrument this ticket ships is what settles it: the first post-cut read either names `job:message-graph-integrity-repair` behind these blocks or it does not. If it does, the fix gets its own sub of #1843, per Out of Scope.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-27T15:03:33Z @neo-opus-ada referenced in commit `d54e8d9` - "feat(memory-core): name the deferred message projection job (#1955)

add_message answers before its graph projection runs, so a block inside that setTimeout(0) pass had no call in flight. It runs as message-graph-projection now, listed beside the other jobs in the recorder's trackJob JSDoc."
- 2026-11-27T15:05:06Z @neo-opus-ada cross-referenced by PR #1957
- 2026-11-27T15:39:47Z @neo-opus-ada referenced in commit `c1c2f5a` - "feat(memory-core): track heap writes, document job callIds (#1955)

The heap observation reporter's boot interval reads V8 heap spaces and writes a file synchronously, so it runs as heap-observation through BaseServer#getBackgroundJobTracker, which Memory Core's Server returns and reuses for its other wirings. The callIds docs in openapi.yaml and recordEventLoopBlock name job:<name>:<run> beside tool call ids."
- 2026-11-27T15:54:14Z @tobiu referenced in commit `46228e0` - "feat(memory-core): name background jobs behind event-loop blocks (#1955) (#1957)

* feat(memory-core): name background jobs behind event-loop blocks (#1955)

The block probe attributes a block only to calls in flight, and only MCP tool calls registered, so background work reported callIds []. trackJob(name, fn) holds job:<name>:<run> across one pass. mc-server's in-process passes now run as jobs: both embed drains, the message drain and its integrity repair, the graph projection drain and deferred projection, the embedding write canary, and timer-fired wake digest flushes. The recorder's trackJob JSDoc lists them and the timers that cannot block.

* feat(memory-core): name the deferred message projection job (#1955)

add_message answers before its graph projection runs, so a block inside that setTimeout(0) pass had no call in flight. It runs as message-graph-projection now, listed beside the other jobs in the recorder's trackJob JSDoc.

* feat(memory-core): track heap writes, document job callIds (#1955)

The heap observation reporter's boot interval reads V8 heap spaces and writes a file synchronously, so it runs as heap-observation through BaseServer#getBackgroundJobTracker, which Memory Core's Server returns and reuses for its other wirings. The callIds docs in openapi.yaml and recordEventLoopBlock name job:<name>:<run> beside tool call ids."
- 2026-11-27T15:54:16Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-27T16:00:15Z

**Correction to AC-3: as written, it passes without naming anything.** #1957 is live since 15:57:47Z (`46228e03`). The first three blocks after the cut, 8.0 s at boot, 1.1 s and 1.7 s, each list six or seven job runs in flight: `memory-wal-drain`, `message-embed-drain`, `message-wal-drain`, plus `message-graph-integrity-repair`. The drain loops cycle several times a second, so some pass of theirs is in flight across almost every block. "Blocks of 10 s or more with empty `callIds` are fewer than 10 %" therefore holds by construction. The 250 ms culprit rule cannot narrow the list either, because jobs, unlike tool calls, persist no end time.

**The read AC-3 needs instead.** For each job name, compare its share of blocks of 10 s or more with its share of blocks between 1 and 2 s. A bystander that is always in flight scores about the same in both. A culprit concentrates in the long blocks. So the 24 h read (due 2026-11-28 15:58Z) names a job only if its share of the long blocks clearly exceeds its share of the short ones. If that leaves the long blocks ambiguous, the missing piece is a per-block record of which in-flight runs ended inside the window. The probe holds that in memory at report time, but nothing persists it. It would then get its own sub of #1843.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-27T16:58:21Z @neo-opus-ada cross-referenced by #1958

