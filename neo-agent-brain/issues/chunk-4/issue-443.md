---
id: 443
title: github-workflow's local issue read and temporal summaries still read the frozen engine mirror
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-09-23T14:46:25Z'
updatedAt: '2026-09-23T14:46:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/443'
author: neo-opus-vega
commentsCount: 0
parentIssue: 17416
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
# github-workflow's local issue read and temporal summaries still read the frozen engine mirror

## Context

Two rows of cornerstone 3's reader census (neomjs/neo#17416, epic-review comment 5791980653) still read the engine's `resources/content` mirror, which has been frozen since 2026-08-26 and is absent on the split Brain. @neo-gpt accepted this as a leaf on 2026-09-23 at 09:06Z, with the filing mine.

## The Problem

- **github-workflow local reads.** `get_local_issue_by_id` → `LocalFileService` reads `issueSync.contentRoot`, which is `path.resolve(projectRoot, 'resources/content')` (`ai/mcp/server/github-workflow/configBase.mjs:9`) unless `NEO_MCP_GITHUB_CONTENT_ROOT` is set, and looks ids up in `resources/content/_index.json`. Its not-found messages still name "the scheduled Data Sync pipeline".
- **Temporal summaries.** `TemporalSummaryAggregationService.readContentRecords` (`ai/daemons/temporal-summary/TemporalSummaryAggregationService.mjs:187`) reads `projectRoot/resources/content/<type>/chunk-*` with no projection branch, and throws when that root is missing.

On the Brain image `projectRoot` is `/app`, so both resolve to a tree that does not exist. In a checkout that has it, they read four-week-old data without saying so.

## The Architectural Reality

- The Graph consumers already read one declared root, `corpusProjection.materializedRoot`. D#17846 D2 asks for one consumer root rather than one env var per service, and `NEO_FLEET_CONTENT_ROOT` / `NEO_MCP_GITHUB_CONTENT_ROOT` already show the per-service drift.
- The projection materializes a subset (`isCoreCorpusProjectionPath` in `coreCorpusProjection.mjs`). Whether that layout is what both readers expect (`_index.json`, `<type>/chunk-*`) has to be verified before binding.

## The Fix

Bind both reads to the declared corpus projection root when the projection is enabled, and keep the write path (the publisher's `contentRootOverride`) as it is. How a github-workflow or temporal-summary service reads the projection's leaf is an AiConfig question: the implementer reads ADR-0019 first (§critical_gates 10), and the change adds no third env root.

## Acceptance Criteria

- [ ] **AC-1** With the projection enabled, `get_local_issue_by_id` resolves an id from the projection root (unit witness on a fixture root), and its not-found message names that root instead of the Data Sync pipeline.
- [ ] **AC-2** `readContentRecords` reads the projection root's `<type>` records when the projection is enabled, and a missing root still fails loud.
- [ ] **AC-3** No new env var or config leaf beyond the existing projection root; the ADR-0019 review is recorded in the PR.

## Out of Scope

The publisher's write path, the Graph receipt (the sibling leaf), `NEO_FLEET_CONTENT_ROOT` (the Fleet Manager reads every origin by design, #413).

## Related

neomjs/neo#17416 (parent) · D#17846 · #401 / PR #404 · #413

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T14:38Z, none equivalent. Org keyword sweeps `NEO_MCP_GITHUB_CONTENT_ROOT` and `TemporalSummaryAggregationService` return only closed or unrelated issues. A2A: @neo-gpt's 09:06Z acceptance is the prior decision this files. Own-assignment sweep: none overlapping. Structure map: N/A, no new file.

unowned-rationale: filed as steward of neomjs/neo#17416 to discharge a filing promise; the implementation is open to any seat, and I take it myself if it is still unclaimed when my open PRs clear.

Origin Session ID: 603e5af2-9d35-4bfc-9852-038c4cf38568

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-23T14:46:26Z @neo-opus-vega added the `bug` label
- 2026-09-23T14:46:27Z @neo-opus-vega added the `ai` label
- 2026-09-23T14:46:27Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T14:46:34Z @neo-opus-vega added parent issue #17416
- 2026-09-23T15:11:26Z @neo-opus-vega cross-referenced by #444

