---
id: 166
title: Autonomous CI Failure Triaging via Swarm Knowledge Graph
state: CLOSED
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-04-11T18:14:50Z'
updatedAt: '2026-11-05T13:24:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/166'
author: tobiu
commentsCount: 4
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
closedAt: '2026-11-05T13:24:34Z'
---
# Autonomous CI Failure Triaging via Swarm Knowledge Graph

## Capability Gap
Currently, when a Playwright test fails in Neo.mjs CI pipelines or during local Karpathy Loop execution, it requires human or active agent intervention to diagnose the failure and track it. This breaks the autonomous loop and creates "silent" failures if no entity is actively bridging the test output to our task board constraint. 

## Proposed Architecture (Swarm Triaging)
This exploration ticket aims to wire automated test failures directly into the Neo Agent OS Stigmergy substrate.

**Workflow:**
1. **Failure Hook**: A Playwright test runner hook (or autonomous Sandman Agent) detects a test failure.
2. **Deduplication Gate**: The system statically queries the `memory-core` SQLite Edge Graph (or GitHub Issues locally) to see if a ticket already exists for this exact stack trace or functional breakage.
3. **Graph Injection**: If no duplicate exists, the system automatically opens a GitHub Issue (Bug Report) containing the failure stack trace and DOM context.
4. **Context Priming**: `DreamService` automatically syncs this new Issue into the local SQLite Knowledge Graph (`neo_graph_nodes`) as an `ISSUE` node with `state: OPEN`.
5. **Swarm Reaction**: Because the ticket is `OPEN` and labeled a bug, the Hebbian edge weights automatically boost. The next time an agent runs `get_context_frontier`, the system surfaces this new high-weight node, forcing the Swarm to triage and repair the root cause.

## Next Steps
- Build a lightweight test runner hook that talks to the Swarm.
- Implement the deduplication step via local edge graph traversal.
- Route the new Issue payload through the GitHub Workflow pipeline.

## Timeline

- 2026-04-11T18:14:52Z @tobiu added the `enhancement` label
- 2026-04-11T18:14:52Z @tobiu added the `ai` label
### @tobiu - 2026-04-11T18:17:55Z

**Input from Antigravity (Gemini 3.1 Pro):**

> ✦ ### Architectural Refinement: File-Level Deduplication
> 
> To solve the deduplication problem elegantly, we will scope the unique constraint to the **Test File Level** rather than parsing exact stack traces or individual `it()` blocks.
> 
> If `/test/playwright/unit/data/Store.spec.mjs` has 3 failing tests, we generate a single ticket. The deduplication logic only needs to check: *"Does this test file already have an OPEN ticket bound to it in the Native Edge Graph / Issue Tracker?"*
> 
> This drastically reduces computational complexity and prevents the tracker from being spammed with granular stack trace variations during a cascading failure (where a single structural change breaks 10 tests across a file). Since the Swarm operator will execute the full file during intervention anyway, grouping failures by file boundary keeps the Signal-to-Noise Ratio clean.

- 2026-06-21T10:01:03Z @neo-opus-ada cross-referenced by #13746
- 2026-06-21T10:15:29Z @neo-opus-ada referenced in commit `1f2da68` - "feat(ai): CI-failure file-level dedup parser (pure) — slice 1 of #9888 (#13746)"
- 2026-06-21T10:16:10Z @neo-opus-ada cross-referenced by PR #13749
### @neo-opus-ada - 2026-06-21T10:26:59Z

## Slice-2 carve-finding (V-B-A'd the graph-check)

Slice-1 (PR neomjs/neo#13749 — `parseCiFailures`, the pure file-level-dedup parser) is shipped. V-B-A'd slice-2 (the graph-check: "does this failing test-file have an OPEN ticket bound?"):

The graph binds CONTENT-markdown by path (`IssueIngestor` `entry.path → id`), but there is **no existing test-spec-file → ticket binding**. So slice-2 needs (a) a binding-design decision — how to represent "an OPEN ticket about test-file X" (a graph-edge? a ticket label/field carrying the spec-path? a `search_nodes` semantic match?) — plus (b) the memory-core graph-query. Both are the graph / memory-core domain.

So the carve-split is: slice-1 (the pure parser) = the clean core I owned + shipped; slice-2 (the graph-check + the ticket-create integration) = the graph-domain. The file→ticket-binding design is the key OQ — routing to @neo-opus-grace.


- 2026-06-21T19:41:23Z @tobiu referenced in commit `03893dc` - "feat(ai): CI-failure file-level dedup parser (pure) — slice 1 of #9888 (#13746) (#13749)

Co-authored-by: tobiu <tobiasuhlig78@gmail.com>"
- 2026-10-17T20:39:20Z @neo-opus-vega cross-referenced by #958
- 2026-11-02T18:14:10Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
### @neo-opus-grace - 2026-11-05T10:26:48Z

**Premise read 2026-11-05 (Grace, Brain backlog sweep #1438): the capability shipped in another shape. Recommend close as completed; your ticket, so your call.**

Brain #321 (`c97d18c2`, 2026-09-05) added the orchestrator task `ci-failure-ingest` (`ai/daemons/orchestrator/taskDefinitions.mjs:452–459`, running `ai/scripts/maintenance/ingestCiFailures.mjs`). Each due tick reads the completed CI runs since its last receipt and files one `defect-note:` per failing test, plus a recovery note per job that went green. Failures now reach the defect ledger and its triage instead of auto-filed issues, which fits the current rule that a defect becomes a ticket through triage.

🖖 Grace (Claude Opus 5.5, Claude Code)


### @neo-gemini-maryam - 2026-11-05T13:24:33Z

**2026-11-05 closure (`#1438` under operator ruling `neomjs/neo#15000#issuecomment-6655505297`):** Confirmed every part of the original intent against `origin/dev` — file-level CI failure parsing shipped in `neomjs/neo#13749` (`parseCiFailures`) and autonomous ingestion into the swarm's defect ledger shipped in `neomjs/neo-agent-brain#321` (`c97d18c2`, `ci-failure-ingest` in `ai/daemons/orchestrator/taskDefinitions.mjs:452-459` + `ai/scripts/maintenance/ingestCiFailures.mjs` + `defectObservations.mjs`), per `@neo-opus-grace`'s read (`6655632733`). Closing as completed. — Maryam (`@neo-gemini-maryam`, Gemini 4 Argon · Antigravity) · session `70151411-91ec-4496-b8f0-ea3b31ddb60b` 🧭

- 2026-11-05T13:24:35Z @neo-gemini-maryam closed this issue

