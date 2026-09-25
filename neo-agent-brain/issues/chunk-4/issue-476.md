---
id: 476
title: 'The graph database''s WAL file never shrinks: 9 GiB for 3.3 MB of live frames'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-24T20:53:40Z'
updatedAt: '2026-09-25T10:07:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/476'
author: neo-opus-vega
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
closedAt: '2026-09-25T10:07:39Z'
---
# The graph database's WAL file never shrinks: 9 GiB for 3.3 MB of live frames

## Context

The graph database's write-ahead log never shrinks. Measured 2026-09-24 20:51Z on `neo-local-canonical`, in the shared `sqlite` volume that kb-server, mc-server and the orchestrator all mount:

| file | size |
|---|---|
| `memory-core-graph.sqlite` | 3,239,145,472 bytes (3.02 GiB) |
| `memory-core-graph.sqlite-wal` | 9,710,802,952 bytes (9.04 GiB) |

@neo-opus-ada measured the log's content: 793 live frames, about 3.3 MB. The rest of the file is a stale high-water mark (defect-note `a9e6cda42a658d28`, promoted on that independent measurement). No other WAL on the plane is above 4 KiB.

## The Problem

In WAL mode SQLite checkpoints and then reuses the log file from its start. It does not truncate the file unless `journal_size_limit` is set. So a single burst that once grew the log, perhaps a reader that held a snapshot while writes continued, leaves every later reader and backup carrying the high-water size. The plane's disk holds about 9 GB that no query can reach.

This is disk hygiene, not a performance or stability fix. The live log is small, and nothing here ties the file's size to latency.

## The Architectural Reality

- `ai/graph/storage/SQLite.mjs:70` opens the graph with `journal_mode = WAL`, `busy_timeout = 5000` and `foreign_keys = ON`. It sets no `journal_size_limit`, and neither does anything else in the Brain (`git grep journal_size_limit` at `dev@19be7e8`: 0 hits).
- `ai/scripts/maintenance/compactGraphLog.mjs:374` runs `wal_checkpoint(TRUNCATE)`, but only as an explicit maintenance script.
- `journal_size_limit` is a per-connection setting. Every process that opens the graph goes through this class, so setting it here covers all three services.
- **Probe (better-sqlite3, the Brain's own binding).** A reader held a snapshot while a writer added about 50 MB, so the WAL grew to 50.9 MiB. The probe then ran a PASSIVE checkpoint and one more write. Without a limit, the WAL stays at 50.85 MiB. With `journal_size_limit = 8 MiB`, it is truncated to exactly 8.00 MiB.

## The Fix

In `SQLite.mjs`, beside `journal_mode = WAL`, set `journal_size_limit` to 64 MiB (`67108864`). After a checkpoint resets the log, SQLite then truncates the file to that size.
- **Why 64 MiB:** the live working set is about 3.3 MB, and the default autocheckpoint fires every 1,000 pages. 64 MiB absorbs bursts without churning the file.
- **Why a constant, not a leaf:** no deployment has a reason to tune it (ADR-0019 A7, "likely YAGNI").

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `memory-core-graph.sqlite-wal` size (`ai/graph/storage/SQLite.mjs:70`) | this ticket | truncated to at most 64 MiB whenever a checkpoint resets the log | none; the pragma is per connection | the pragma's JSDoc line | AC-2 |

Decision Record impact: none.

## Acceptance Criteria

- [ ] **AC-1** `SQLite.mjs` sets `journal_size_limit = 67108864` on every graph connection, and a spec reads the pragma back as 67108864.
- [ ] **AC-2** A spec opens a store through the real `initAsync`, grows its WAL past 64 MiB, and runs a checkpoint. The WAL file then measures at most 64 MiB. With the pragma removed, the same spec fails.

## Out of Scope

- **Why the log grew to 9 GB.** A long-held reader snapshot is the likely shape, but nothing here has measured it. The limit bounds the file whatever the cause.
- **The other WAL-mode databases** (`KBRecorderService`, `MemoryCoreRecorderService`, `SourceRegistryService` and others). Their logs are 4 KiB or smaller on the plane.

## Avoided Traps

- **A periodic `wal_checkpoint(TRUNCATE)` task.** It needs a scheduler slot and a lease, and it truncates only when it runs. The pragma truncates on every reset, at no cost.
- **Deleting the WAL file by hand.** Frames that have not been checkpointed live in it, so deleting it can lose committed data.

## Related

#464 (the drain) · #466 · `compactGraphLog.mjs`

Sweeps at 20:53Z:
- **Live latest-open:** the latest 20 open `neomjs/neo-agent-brain` issues. None is equivalent.
- **Exact search** (`journal_size_limit`, `sqlite-wal`, `WAL file`, `graph WAL`; org-wide): no match. #61 concerns a host-edge process holding an orphaned host graph, not this log.
- **MC sweep:** a 2026-08-11 analysis already named the unset `journal_size_limit` a minor hygiene defect, not an instability cause, and filed nothing. This ticket keeps that label.
- **A2A in-flight:** no WAL claim in the last hour.
- **Own-assignment sweep:** 7 open; none overlapping.
- **Structure map:** executed for #471 today; the owner is `ai/graph/storage/`.

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("graph sqlite WAL file 9.7 GB never shrinks journal_size_limit high-water mark checkpoint")`

Authored by Vega (Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-24T20:53:41Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-24T20:53:41Z @neo-opus-vega added the `bug` label
- 2026-09-24T20:53:41Z @neo-opus-vega added the `ai` label
- 2026-09-24T20:53:42Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T20:56:23Z @neo-opus-vega cross-referenced by PR #477
- 2026-09-25T10:07:39Z @tobiu referenced in commit `0d81f04` - "Merge pull request #477 from neomjs/vega/476-graph-wal-journal-size-limit

fix(graph): the graph WAL is truncated to 64 MiB after a checkpoint resets it (#476)"
- 2026-09-25T10:07:39Z @tobiu closed this issue

