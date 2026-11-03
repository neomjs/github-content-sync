---
id: 1448
title: 'The plane never embeds A2A messages: the message loop runs only in the standalone daemon'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-03T00:04:52Z'
updatedAt: '2026-11-03T00:58:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1448'
author: neo-opus-vega
commentsCount: 1
parentIssue: 35
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-03T00:47:00Z'
---
# The plane never embeds A2A messages: the message loop runs only in the standalone daemon

## Context

#1439's post-merge validation failed on the plane ([6591683330](https://github.com/neomjs/neo-agent-brain/issues/1439#issuecomment-6591683330)). After the cut to `dev` `41d360ab`, no message is embedded.

## The Problem

The plane hosts its embed drain in-process. The mc-server env sets `NEO_MEMORY_WAL_IN_PROCESS_DRAIN=true` and `NEO_MESSAGE_WAL_IN_PROCESS_DRAIN=true` (`ai/mcp/server/memory-core/Server.mjs:343`), and neither container runs `ai/daemons/embed/daemon.mjs`. #1443 started the message embed loop only in that daemon.

## The Architectural Reality

`drainCycle.mjs`'s loop host serves two embed hosts (its JSDoc, l.605): the embed daemon, and Memory Core's in-process drain under the memory-WAL lock. Each host wired its loops separately, so one host could embed memories while skipping messages.

## The Fix

`startEmbedDrainLoops` (`drainCycle.mjs`) starts both loops, the memory WAL and `MESSAGE_WAL_STORE`. Both hosts call it: `daemon.mjs`, and `Server.mjs`'s in-process branch under its memory-WAL lock. A host can no longer start one loop without the other.

Decision Record impact: `none`.

## Acceptance Criteria

- [ ] AC-1: `startEmbedDrainLoops` starts both loops, and each WAL lands only in its own collection. Covered by a spec.
- [ ] AC-2: Both embed hosts start their drains through it.
- [ ] AC-3 (post-merge, plane): after a re-cut, a message sent after the cut appears in `neo-agent-messages` with its metadata. This is #1439's AC-5, re-observed.

## Related

- Parent: #35. Defect in #1439 (PR #1443).

*Sweep attestations:*
- *Live latest-open + closed sweep:* latest open at 2026-11-03T00:0xZ; no equivalent. This defect was found minutes ago on my own merged PR.
- *A2A in-flight sweep:* no claim.
- *Own-assignment sweep:* the overlap is the #35 chain.

Ownership: self-assigned (Vega).

Origin Session ID: afcce2e4-5eb9-4fa6-8895-4730263a042d

Retrieval Hint: "in-process embed drain message loop startEmbedDrainLoops plane inProcessDrain"

## Timeline

- 2026-11-03T00:04:52Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-03T00:04:53Z @neo-opus-vega added the `bug` label
- 2026-11-03T00:04:54Z @neo-opus-vega added the `ai` label
- 2026-11-03T00:04:54Z @neo-opus-vega added the `agent-os` label
- 2026-11-03T00:05:20Z @neo-opus-vega cross-referenced by PR #1449
- 2026-11-03T00:47:00Z @tobiu referenced in commit `cf17e6c` - "fix(memory-core): every embed host starts the message loop beside the memory loop (#1448) (#1449)

The plane hosts its embed drain in-process (memoryWal.inProcessDrain), where #1443 never started the message loop, so no message was embedded there. startEmbedDrainLoops starts both loops and both hosts — the embed daemon and Memory Core's in-process drain — call it."
- 2026-11-03T00:47:01Z @tobiu closed this issue
- 2026-11-03T00:58:09Z @neo-opus-vega cross-referenced by #1439
### @neo-opus-vega - 2026-11-03T00:58:13Z

**Post-merge validation: AC-3 observed on the plane** at `cf17e6c1`. Memory Core's in-process drain (`NEO_MEMORY_WAL_IN_PROCESS_DRAIN=true`) now embeds messages too: a message sent after the cut was embedded within 20 s, with its `from`, `to` and `kind` metadata and its WAL embed marker. The receipt is on #1439: https://github.com/neomjs/neo-agent-brain/issues/1439#issuecomment-6592522360

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-03T00:58:36Z @neo-opus-vega cross-referenced by PR #1443

