---
id: 1931
title: Wake dispatch records pile up forever and slow every who_is_online call
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-25T17:15:45Z'
updatedAt: '2026-11-25T19:02:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1931'
author: neo-opus-ada
commentsCount: 0
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
closedAt: '2026-11-25T19:02:00Z'
---
# Wake dispatch records pile up forever and slow every who_is_online call

## Context

Found while attributing #1843's `who_is_online` floor. The full measurement table is in https://github.com/neomjs/neo-agent-brain/issues/1843#issuecomment-7169115725. Measured 2026-11-25 around 17:10Z, read-only:

- **The listing.** In `mc-server`, `fs.readdir('/app/wake-receiver-records')` returns 21,790 entries in 850–880 ms (3 runs). The directory is a macOS host bind mount. On the host's native filesystem the same listing takes 11 ms.
- **The floor.** `who_is_online`'s 24 h minimum is 860 ms. `readWakeDelivery` lists this directory on every call that does not join a read already in flight.
- **The growth.** There are 21,785 records, the oldest from 2026-08-16, and nothing prunes them (about 216 a day). The reader holds the 21,781 terminal ones, about 57 MB of V8 heap (measured on the host).
- **The trend.** At 9,562 entries the same in-container listing took 0.19–0.21 s (#787, 2026-10-03).

## The Problem

#787 made a repeat read cost one `readdir` plus the held records. It deferred retention with this sentence: "Retention earns its own leaf when that measures slow." It does now: the listing costs about 4× as much for 2.3× the entries. The listing time and the held heap both grow with every wake ever dispatched, and `healthcheck`'s full check reads the same directory (#787).

## The Architectural Reality

- **The file is the duplicate check.** `WakeReceiverState` (`ai/daemons/wake/receiverState.mjs`) keeps one file per stable key. `getWakeRecordKey` builds it from the subscription plus either the event id or the sorted source event ids. `accept()` links the file into `records/` and answers `duplicate` on `EEXIST`. A digest re-derived over the same unread source set therefore stays a duplicate today, however late it is re-sent.
- **Deleting re-opens a key.** The signed envelope carries `emittedAt`, but the `/wake` handler in `receiver.mjs` checks no freshness window.
- **The sender degrades on a refusal.** `WebhookDeliveryService` (`ai/services/memory-core/WebhookDeliveryService.mjs`) marks a subscription degraded on any 4xx except the unknown-subscription 404, and never attempts a degraded route again.
- **Locking.** The receiver's drain pass runs under `withStateMutex`; `accept()` does not.
- **The roster needs no reader change; message verdicts do.** The reader drops from `held` any name the directory stops listing (#787 AC-3), so a smaller directory also shrinks the heap. A message's verdict is first-delivery evidence that the Fleet asks for with no maximum age (`deliveriesOf` in `wireFleetOpenWorkWakes.mjs`; holder silence in `openWorkWakes.mjs` needs `delivered`), so it must outlive its record. The container sees only `records/`.
- **Not every file is a record.** The receiver's `receiver.liveness` file sits in the same directory and is not a `.json` record.
- Design authority: #787's Out of Scope, quoted above, and `receiverState.mjs`'s module summary: "One file per stable event/source identity keeps acceptance atomic and retry-safe".

## The Fix

1. **Retire records instead of deleting them.** `WakeReceiverState` gains a prune pass. It moves each terminal `.json` record whose `acceptedAt` is older than `RECORD_RETENTION_MS` (a module constant of 7 days, not an `AiConfig` leaf) from `records/` to a sibling `archive/<first two hex of the key>/<key>.json`. The sibling lies outside the bind-mounted `records/` path, so no container reader lists it. Non-terminal records stay at any age, and so do non-record files.
2. **The duplicate check covers both places.** A key in the archive is never accepted or dispatched again. `accept()` answers `duplicate` for an archived key before staging anything. The archive check, the staging and the link run under a key lock that every retirement step also takes, so a retry cannot slip in between them. A retirement links every expired record into the archive and syncs every directory on that path (shard, `archive/`, the state directory) strictly before any record leaves `records/`; a failed or unsupported sync fails the pass with every live file in place.
3. **When the prune runs.** At receiver startup, and from the drain cadence at most once an hour, under `withStateMutex`.
4. **The reader keeps its contract.** Moved names leave `held` by #787's rule. A subscription whose newest record moved projects like a route with no dispatch yet. Each retired record that carried source events leaves its fate (`state`, `sourceEventIds`, attempt times) as one line of `records/retired.ndjson`, written atomically before the live file goes. `readWakeDelivery`'s message-ID path reads it, so a delivered message keeps its first delivery after retirement; roster and health reads never open it.

Effect: the live directory holds about one week, roughly 1,500 records at today's rate, plus `retired.ndjson` (about 4.5 MB for the 100 days retired at the first pass).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `WakeReceiverState#accept` duplicate answer | `receiverState.mjs` | `duplicate` for a key in `records/` or in the archive; an archived key stages nothing | an unreadable archive fails the accept as today's I/O errors do, never as `accepted` | JSDoc | unit |
| Contents of `records/` | `receiverState.mjs` | non-terminal records, plus terminal records inside the window, plus `retired.ndjson` | — | JSDoc | unit + PMV |
| `readWakeDelivery` `messages` for a retired record's message | `wakeDeliveryReader.mjs` | the same verdict and first-delivery time as before retirement | an index that exists and cannot be read leaves every message `unknown` | JSDoc | unit |

## Acceptance Criteria

- [ ] AC-1 (unit): the prune pass moves a terminal record older than the window into the archive. It keeps a terminal record inside the window, a `pending` or `dispatching` record of any age, and `receiver.liveness`.
- [ ] AC-2 (unit): re-accepting an archived key answers `duplicate`, writes nothing to `records/` and dispatches nothing. This holds for an event-id key and for a source-set key.
- [ ] AC-3 (unit): an `accept()` racing a retirement still answers `duplicate` and leaves no pending record. A pass interrupted after the archive link finishes the move on its next run, so a key is always in at least one of the two places. Every directory on the archive path is synced before the live file goes, and a failed or unsupported sync keeps the live file.
- [ ] AC-4 (unit): after the prune pass shrinks the directory, `readWakeDelivery` drops the moved names from `held`, and its four outcomes are unchanged. A delivered message read through the message-ID path keeps its first delivery after its record retires.
- [ ] PMV-1 (post-merge, after the host receiver restarts on the merged code): the in-container `readdir` of `/app/wake-receiver-records` lists about a week of records, and `who_is_online`'s 24 h minimum in `get_memory_core_tool_metrics` falls well below 860 ms.

## Out of Scope

- **Pruning the archive and `retired.ndjson`** (about 85 MB and 4.5 MB per 100 days of host disk; the reader holds the parsed index). It needs a freshness window on `emittedAt`, with a refusal that `WebhookDeliveryService` treats as success. Not filed.
- **#1843's other hot paths** (`_readReviewLifecycleLoad`, `list_messages`, `query_recent_turns`) and the event-loop block probe.

## Avoided Traps

- **Deleting old records.** The file is the duplicate check. A digest re-emitted over the same unread source set would wake the seat again once per window.
- **Refusing expired envelopes with a 4xx.** The sender degrades the route, so one stale envelope would cost a seat its wake route.
- **A TTL cache in the reader.** #787's Avoided Traps already rule it out: it serves stale verdicts for its whole window.
- **Keeping retired message facts only in the reader's memory.** An `mc-server` restart would turn every retired delivery into `undelivered`.

## Decision Record impact

None. This is an in-place change to `ai/daemons/wake/receiverState.mjs`, `receiver.mjs` and `ai/services/memory-core/wakeDeliveryReader.mjs`, owning folders `ai/daemons/wake` and `ai/services/memory-core` (structure map read). It adds no new module and no config leaf.

## Related

Parent: #1843 · Predecessor: #787 (PR #791) · #1929 (headroom, sized after #1843)

Live latest-open + closed sweep:
- Ran at 2026-11-25T17:13Z: the latest 20 open, plus open and closed searches for "wake receiver records". #787's Out of Scope and Avoided Traps were read.
- 156 open issues, which is the red band. #1843 owns the `who_is_online` hot path, but it spans three tools plus a probe, and #787 already ruled retention its own leaf. So this is filed as a sub, not folded in.
- A2A sweep: the last 30 messages, all read states. No claim on this scope.
- MC sweep: "wake receiver dispatch records directory retention prune old records readdir slow" and "who_is_online slow latency floor wake delivery reader healthcheck", 12 results. No prior decision found.
- Own-assignment sweep: 7 open. #1843 is the parent, and none other is on this surface.

Retrieval Hint: "wake receiver records retention archive duplicate key who_is_online readdir floor retired delivery index"
Origin Session ID: cc55a843-2d87-4222-b61d-8efb38ee837d

## Timeline

- 2026-11-25T17:15:45Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-25T17:15:48Z @neo-opus-ada added the `bug` label
- 2026-11-25T17:15:48Z @neo-opus-ada added the `ai` label
- 2026-11-25T17:15:48Z @neo-opus-ada added the `performance` label
- 2026-11-25T17:15:49Z @neo-opus-ada added the `agent-os` label
- 2026-11-25T17:15:51Z @neo-opus-ada added parent issue #1843
- 2026-11-25T17:22:25Z @neo-gemini-julia cross-referenced by #1843
- 2026-11-25T17:29:26Z @neo-opus-ada cross-referenced by PR #1932
- 2026-11-25T17:42:35Z @neo-opus-ada cross-referenced by #1933
- 2026-11-25T18:16:10Z @neo-opus-ada referenced in commit `2da8f12` - "feat(wake): retired delivery index and strict retirement syncs (#1931)

A replayed retired key answers duplicate before anything is staged: the archive check, the staging and the link run under the key lock. Retirement syncs every directory on the archive path strictly and writes each retired record's fate to records/retired.ndjson before the live file goes. The reader's message-ID path reads that index, so a delivered message keeps its first delivery after its record retires; roster and health reads never open it."
- 2026-11-25T18:42:24Z @neo-opus-ada referenced in commit `7e844fe` - "fix(wake): a message read never joins an older index read (#1931)

A call whose listing no longer shows a just-retired record could join a retired-index read that began before the retirement and answer undelivered. Each listing now carries a number, and a message call joins an index read only if it started after that listing; the receiver writes a fate before it unlinks the record, so such a read holds every record the listing misses."
- 2026-11-25T19:02:00Z @tobiu referenced in commit `0afb757` - "feat(wake): retire week-old terminal wake records to an archive (#1931) (#1932)

* feat(wake): retire week-old terminal wake records to an archive (#1931)

The receiver never pruned its dispatch records: 21,785 files since
2026-08-16. Every reader pays per file. In mc-server, who_is_online's
per-call readdir of the bind-mounted records directory took 850-880 ms,
which is the tool's 860 ms floor. The reader also held ~57 MB of
terminal records, and the receiver's own drain parsed all of them on
every pass.

Terminal records accepted more than RECORD_RETENTION_MS (7 days) ago
move to a sibling archive/<kk>/<key>.json outside the bind mount. The
record file is the acceptance dedupe, so it is moved rather than
deleted, and accept() answers `duplicate` for an archived key. The
archive check and the link hold a key lock that every retirement step
takes. A pass links all expired records and syncs the archive before
any of them leaves records/. The receiver prunes at startup, before
the first drain, and then hourly under its state mutex.

* feat(wake): retired delivery index and strict retirement syncs (#1931)

A replayed retired key answers duplicate before anything is staged: the archive check, the staging and the link run under the key lock. Retirement syncs every directory on the archive path strictly and writes each retired record's fate to records/retired.ndjson before the live file goes. The reader's message-ID path reads that index, so a delivered message keeps its first delivery after its record retires; roster and health reads never open it.

* fix(wake): a message read never joins an older index read (#1931)

A call whose listing no longer shows a just-retired record could join a retired-index read that began before the retirement and answer undelivered. Each listing now carries a number, and a message call joins an index read only if it started after that listing; the receiver writes a fate before it unlinks the record, so such a read holds every record the listing misses."
- 2026-11-25T19:02:02Z @tobiu closed this issue
- 2026-11-25T19:04:33Z @neo-gemini-julia cross-referenced by PR #1937
- 2026-11-25T19:09:06Z @neo-opus-ada cross-referenced by #1939
- 2026-11-25T19:09:13Z @neo-opus-ada cross-referenced by #1940

