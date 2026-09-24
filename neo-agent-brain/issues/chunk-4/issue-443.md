---
id: 443
title: github-workflow's local issue read and temporal summaries still read the frozen engine mirror
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T14:46:25Z'
updatedAt: '2026-09-24T13:03:46Z'
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
closedAt: '2026-09-24T13:03:46Z'
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
- **The two readers run on different planes (verified 2026-09-24).** `temporal-summary` is container-plane and cloud-only (`taskAuthority.mjs:105`; `Orchestrator#temporalSummaryEnabled` resolves `resolveCloudOnlyEnabled`), where the projection root exists. Its materialized layout (`<type>/chunk-*/*.md`) is exactly what `readContentRecords` reads. The github-workflow server runs in each seat's harness on the host (`projectRoot = process.cwd()`), where the projection root, inside the plane's Docker volume, does not exist. Its lookup is already origin-qualified (`findContentIndexEntry(index, {repoSlug: aiConfig.repo, …})`). Pointed at a `github-content-sync` checkout through `NEO_MCP_GITHUB_CONTENT_ROOT`, it reads current data with no code change. Keeping such a checkout current on a seat is D#17846's D4 distribution question.

## The Fix

The temporal summary reads the corpus projection root, refuses when the projection is disabled, and never falls back to a checkout's frozen `resources/content`. The github-workflow reader's not-found and stale-index messages name the root they read, instead of claiming that "the scheduled Data Sync pipeline regenerates resources/content/_index.json" (nothing regenerates that tree now). The write path (the publisher's `contentRootOverride`) is untouched. Both reads use existing leaves at the use site (ADR-0019), with no third env root.

## Acceptance Criteria

- [ ] **AC-1** `get_local_issue_by_id`'s not-found and stale-index messages name the root they read instead of the Data Sync pipeline. Unit witness. Reading a current corpus on a seat's host is configuration plus D#17846 D4, not code here; see the Architectural Reality.
- [ ] **AC-2** `readContentRecords` reads the projection root's `<type>` records when the projection is enabled, refuses a disabled projection, and a missing root still fails loud.
- [ ] **AC-3** No new env var or config leaf beyond the existing projection root; the ADR-0019 review is recorded in the PR.

## Out of Scope

The publisher's write path, the Graph receipt (the sibling leaf), `NEO_FLEET_CONTENT_ROOT` (the Fleet Manager reads every origin by design, #413).

## Related

neomjs/neo#17416 (parent) · D#17846 · #401 / PR #404 · #413

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T14:38Z, none equivalent. Org keyword sweeps `NEO_MCP_GITHUB_CONTENT_ROOT` and `TemporalSummaryAggregationService` return only closed or unrelated issues. A2A: @neo-gpt's 09:06Z acceptance is the prior decision this files. Own-assignment sweep: none overlapping. Structure map: N/A, no new file.

Origin Session ID: 603e5af2-9d35-4bfc-9852-038c4cf38568

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿



## Timeline

- 2026-09-23T14:46:26Z @neo-opus-vega added the `bug` label
- 2026-09-23T14:46:27Z @neo-opus-vega added the `ai` label
- 2026-09-23T14:46:27Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T14:46:34Z @neo-opus-vega added parent issue #17416
- 2026-09-23T15:11:26Z @neo-opus-vega cross-referenced by #444
- 2026-09-24T12:21:54Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-24T12:33:01Z @neo-opus-vega cross-referenced by PR #454
- 2026-09-24T12:52:02Z @neo-opus-vega cross-referenced by #19158
- 2026-09-24T13:03:46Z @tobiu referenced in commit `3458488` - "Merge pull request #454 from neomjs/vega/443-corpus-readers

fix(temporal-summary): the corpus readers take the projection root and stop pointing at a retired sync (#443)"
- 2026-09-24T13:03:47Z @tobiu closed this issue
- 2026-09-24T14:10:17Z @neo-opus-grace cross-referenced by #459

