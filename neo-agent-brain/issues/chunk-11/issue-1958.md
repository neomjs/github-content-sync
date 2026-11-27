---
id: 1958
title: The integrity repair blocks mc-server's event loop 7–10 s every 70 s
state: OPEN
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-opus-ada
createdAt: '2026-11-27T16:58:20Z'
updatedAt: '2026-11-27T16:58:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1958'
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
---
# The integrity repair blocks mc-server's event loop 7–10 s every 70 s

Owner: Ada (`@neo-opus-ada`), who owns the parent #1843 and the evidence below. This is the named-job follow-up that #1955's Out of Scope reserved for a sub of #1843.

## Context

#1957's job attribution has been live since 15:57:47Z on `46228e03`. In its first 58 minutes, mc-server logged 119 blocks of 1 s or more: 56 of 5 s or more, 10 between 2 and 5 s, and 53 under 2 s. I compared each job's share of the long blocks (5 s or more) with its share of the short ones (under 2 s), the read set out in the #1955 correction:

| In flight | Blocks ≥ 5 s | Blocks < 2 s |
|---|---:|---:|
| `message-graph-integrity-repair` | **93 %** (52 of 56) | 30 % |
| `memory-wal-drain`, `message-embed-drain` | 100 % | 64 % |
| `message-wal-drain` | 84 % | 64 % |
| `heap-observation` | 34 % | 6 % |

- **The drains are bystanders.** They cycle several times a second, so they are in flight across most blocks of any length.
- **`heap-observation` is a timer artifact.** Its 10 s timer comes due inside a 7 s block and fires right after it, inside the window (Julia's depth-floor case on #1957).
- **The repair is the culprit.** It runs about once every 70 s, yet it is in flight across 52 of the 56 long blocks.
- **Before the instrument,** its hourly digest counted 53 passes in 13:15–14:16Z, every one clean with `scanned: 0` and `candidateCount: 0`, against 41 no-call blocks of 5 s or more in the same hour ([#1955 comment](https://github.com/neomjs/neo-agent-brain/issues/1955#issuecomment-7221213475)).
- **Cost:** about 53 blocks of 7–10 s an hour, or 10–15 % of wall time, with every tool call held behind them. Fleet Manager's liveness read times out at 10 s.

## The Problem

`createMessageGraphIntegrityRepairCadence` (`ai/daemons/message/drainCycle.mjs:93`) runs after a message-drain cycle, at least 60 s after its previous run *ended* (`MESSAGE_GRAPH_REPAIR_CADENCE_MS`). It calls `repairMessageGraphIntegrity({box: 'all'})` (`ai/services/memory-core/MailboxService.mjs:4345`). Each pass classifies every projected WAL id through `classifyMailboxGraphProjectionCandidates()` (`:2411`). That includes `getMessageWalGraphProjectionStats`, copies of the routing and cohort maps (`new Map(stats.mailboxRoutingById)`), and SQLite checks for damaged ids, and it finds nothing. At #563's scale this was cheap. The store now holds about 87k messages and grows every minute, so the pass cost grows with it.

## The Architectural Reality

- Design authority: #563 moved the repair off the `list_messages` read path onto the drain host's cadence (`94fa7f5d`), which is where it belongs. It is the per-pass scope that no longer fits.
- `ai/daemons/message/drainCycle.mjs` holds the cadence, the digest, and the `afterCycle` seat, which #1957 wraps as `message-graph-integrity-repair`.
- `ai/services/memory-core/MailboxService.mjs` holds `repairMessageGraphIntegrity`, `getMailboxGraphProjectionRepairCandidates` (`:2639`, which coalesces concurrent scans), `classifyMailboxGraphProjectionCandidates`, and its caches (`messageWalCandidateMetadataCacheById`, `graphProjectionRepairCursorByView`, `graphProjectionRepairFailureById`).

## The Fix

Measure first, because the pass has no duration of its own on record. Time `classifyMailboxGraphProjectionCandidates` and each SQLite phase on the live store, read-only, and post the split.

Then make a pass cost what changed:
- Classify only the ids projected since the last clean pass, using a high-water mark the WAL marker index can supply.
- Keep a full sweep at boot and on a slow cadence as the backstop for older damage.
- Bound each synchronous segment, so a full sweep yields between batches instead of holding the loop.

## Acceptance Criteria

- [ ] The measured split of one full pass on the live store is posted here before the change.
- [ ] A unit spec: a clean pass with no new projected ids does not classify the whole population, and a damaged older record is still found and repaired by the full sweep.
- [ ] Post-merge, a 24 h probe read on a plane cut to the merge commit: `message-graph-integrity-repair` no longer concentrates in blocks of 5 s or more, so its share of them is close to its share of blocks under 2 s. Blocks of 5 s or more fall by at least 80 % from this hour's 56.

## Out of Scope

- The three `mark_read` blocks of about 7 s in the same hour, a separate shape.
- Heap growth and sizing (#1929).
- The drain loops' own cost.

## Avoided Traps

- Raising the cadence alone. Each pass would still take 7–10 s and keep growing with the store; there would only be fewer of them.
- Moving the pass off the main thread before measuring. It reads the graph store that the main thread owns.

## Related

Parent #1843 · instrument #1955 / #1957 · origin #563 · #1929

Live latest-open + closed sweep: I checked the latest 20 open issues, open and closed searches (integrity repair message graph, plus cadence) and my own assignments at 2026-11-27T16:57Z, plus the last hour of A2A claims and a Memory Core query. No equivalent ticket exists; the closed hit is #1955, the instrument. 153 open (red band), filed as a sub of its owning #1843.

Retrieval Hint: "message graph integrity repair event loop block every 70 s full classification"

Origin Session ID: a3e89c2c-c549-46ca-aadf-1a2e17f7d2ec

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-27T16:58:20Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-27T16:58:22Z @neo-opus-ada added the `bug` label
- 2026-11-27T16:58:22Z @neo-opus-ada added the `ai` label
- 2026-11-27T16:58:22Z @neo-opus-ada added the `performance` label
- 2026-11-27T16:58:40Z @neo-opus-ada added parent issue #1843
- 2026-11-27T17:37:06Z @neo-opus-ada cross-referenced by #1959

