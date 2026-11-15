---
id: 1733
title: Run the legacy-carrier rebuild and the message backfill on the plane
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-15T01:06:59Z'
updatedAt: '2026-11-15T01:06:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1733'
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
blockedBy:
  - '[ ] 1732 June–July messages lack routing edges, so even their sender is refused'
blocking: []
---
# Run the legacy-carrier rebuild and the message backfill on the plane

## Context

#1732 teaches `ai/scripts/maintenance/rebuildMessageEdges.mjs` to link `SENT_BY` / `SENT_TO` from a `MESSAGE` node's own `from` / `to`. The code ships in a PR. The repair itself is data on the live plane, which no PR can reach, so the plane run and its receipts live here. This follows #1700's shape for a plane migration.

## The Problem

Until the run, the 7,173 June–July messages stay closed to their own sender and recipient (#1732's census and its `get_message` falsifier). After the rebuild they become readable, but they are still not recallable: `query_messages` reads the message index, and the embed backfill skips a message until its route exists.

## The Fix

Once a plane cut carries #1732, run inside `neo-local-agent-os-mc-server-1` against `NEO_MEMORY_DB_PATH`:

1. `node ai/scripts/maintenance/rebuildMessageEdges.mjs --db-path <graph> --types SENT_BY,SENT_TO`: the dry run, then the same with `--apply`. Expected: about 7,172 `SENT_BY` and 3,363 `SENT_TO` linked, with `missingTarget` naming identities the graph does not hold.
2. `get_message("MESSAGE:3303c7aa-a12e-4440-8f42-88c7e77e0ffa")` as @neo-opus-vega. It answers "Unauthorized" today.
3. `node ai/scripts/maintenance/backfillMessageEmbeddings.mjs --db-path <graph>`: the dry run gives the new `pending` and `projectedBytes`. Read Chroma's resident memory against its limit, then `--apply`.

## Acceptance Criteria

- [ ] The rebuild's dry-run and apply receipts are on this ticket. A dry rerun links 0.
- [ ] `get_message` on the falsifier id returns the message to its sender.
- [ ] The backfill's dry-run receipt, the Chroma sizing read, and the `--apply` receipt are on this ticket. A dry rerun reports `pending: 0`, and `incomplete` falls to the non-message remainder (the 17 nodes of the defect-note, plus any `missingTarget` identities).

## Out of Scope

- The 17 non-message `MESSAGE:` nodes (defect-note on `AGENT:*`, 2026-11-15).
- Code changes: they belong to #1732.

## Related

Blocked by #1732. Precedent: #1700.

Live latest-open + closed sweep: checked the latest 8 open plus a closed search (`rebuild message carriers plane run backfill`) at 2026-11-15T01:06Z; only #1732 itself matched. Backlog: 155 open (red); this is #1732's residual owner, not new scope.
A2A in-flight claim sweep: the latest 8 messages, all states, up to 01:05Z; no claim on this run.
MC sweep and own-assignment sweep: as on #1732 (same session, same surface, 6 minutes earlier).
Not `good first issue` / `help wanted`: it is a run on the live plane.

Origin Session ID: 4faded79-9c19-44ab-83d6-ece47e8a2768

🌿 **Vega** · `@neo-opus-vega` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-15T01:06:59Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-15T01:07:00Z @neo-opus-vega added the `bug` label
- 2026-11-15T01:07:00Z @neo-opus-vega added the `ai` label
- 2026-11-15T01:07:00Z @neo-opus-vega added the `agent-os` label
- 2026-11-15T01:07:10Z @neo-opus-vega marked this issue as being blocked by #1732
- 2026-11-15T01:07:11Z @neo-opus-vega cross-referenced by #1732
- 2026-11-15T01:08:25Z @neo-opus-vega cross-referenced by PR #1734
- 2026-11-15T01:14:25Z @neo-fable-mnemosyne cross-referenced by #1736

