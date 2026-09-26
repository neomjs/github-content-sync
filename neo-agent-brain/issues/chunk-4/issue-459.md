---
id: 459
title: Brain readers fall back to resources/content under projectRoot or __dirname
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-09-24T14:10:15Z'
updatedAt: '2026-09-26T08:52:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/459'
author: neo-opus-grace
commentsCount: 2
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
# Brain readers fall back to resources/content under projectRoot or __dirname

## Context

neomjs/neo#19158's census ([IC 5815306776](https://github.com/neomjs/neo/issues/19158#issuecomment-5815306776)) lists the Brain readers that fall back to `resources/content` when no corpus root is declared:
- **Fleet:** the `contentRoot` leaf defaults to `path.resolve(projectRoot, 'resources/content')` (`NEO_FLEET_CONTENT_ROOT`, `ai/configBase.mjs:322`).
- **github-workflow:** `issueSync.contentRoot` falls back to `projectRoot/resources/content` when `contentRootOverride` is null (`ai/mcp/server/github-workflow/configBase.mjs:9,337`).
- **Use-site fallbacks** (ADR-0019 A1):
  - from `aiConfig.projectRoot`: `ai/services/graph/GoldenPathSynthesizer.mjs:988` and `ai/services/ingestion/ConceptDiscoveryService.mjs:451,506`;
  - from `__dirname`, a module-level default: `ai/services/ingestion/IssueIngestor.mjs:18`.
- **Diagnostics:** `ai/scripts/diagnostics/detectTruncatedTimelines.mjs:40` (from `projectRoot`) and `audit-discussion-lifecycle.mjs:24` (from the working directory).
- **KB typing:** `QueryService#inferSourceType` (`ai/services/knowledge-base/QueryService.mjs:837-845`) and `SearchService.mjs:338` classify sources by the mirror's literal prefixes.

#443 already moved github-workflow's local issue read and the temporal summaries.

## The Problem

D#19051's cutover predicate is that no runtime reader derives `resources/content` from `projectRoot`, the working directory or `__dirname`, and the KB's prefix typing counts as a reader. A Brain checkout carries no `resources/content` (`ai/services/fleet/devFleetServer.mjs:288`). So each fallback reads nothing, or reads the frozen mirror on a plane that mounts an engine checkout.

## The Architectural Reality

**Declared roots already exist:** github-workflow's `issueSync.contentRootOverride` (`NEO_MCP_GITHUB_CONTENT_ROOT`) and the orchestrator's corpus projection (`ai/configBase.mjs:1399-1407`, `materializedRootOverride`).

ADR-0019 governs every change here:
- leaves are declarative, and a use site reads the resolved leaf (§5);
- a use-site fallback is A1;
- two leaves meaning the same root are C4, so a reader census picks the one that survives.

## The Fix

- **Readers:** each one reads the declared corpus root at its use site. The `projectRoot` and `__dirname` defaults and fallbacks are removed. With no root declared, a reader refuses by name instead of reading an absent tree.
- **KB typing:** follow where corpus-tenant rows get their `type`.
  - If those rows reach `inferSourceType`, it learns the corpus layout: `<origin>/<family>/…` plus the archive.
  - If they carry their own type, the prefix branches retire with the legacy rows (#417).
  - Either way, the release branch follows neomjs/neo#19157's path.
- **Diagnostics:** they read the same leaf, or retire.

**Decision Record impact:** depends on ADR-0019 (§3, §5) and ADR 0004 §2.1.1.

## Acceptance Criteria

- [ ] **AC-1:** No Brain source derives `resources/content` from `projectRoot`, the working directory or `__dirname`. The rows above are gone.
- [ ] **AC-2:** Query results type a corpus issue, pull, discussion and archived issue correctly. Record the red-first state of today's typing in the PR.
- [ ] **AC-3:** With no corpus root declared, each reader refuses by name.
- [ ] **AC-4:** `check-aiconfig-antipatterns` and `lint-config-template-ssot` pass, and no use-site default is added.

## Out of Scope

- Concepts (neomjs/neo#19093).
- The logical-identity move (#457).
- Release notes (neomjs/neo#19157).

## Related

neomjs/neo#17416 (parent) · neomjs/neo#19158 · #443 · #417 · #457

Sweeps at 14:09Z:
- **Live latest-open:** the latest 20 open issues here; none equivalent. #442 records a deployed read and is not a repoint.
- **Org search** (`source typing`, `literal prefix`, `IssueIngestor`, `contentRoot projectRoot`): nothing equivalent.
- **MC:** as in the census sweep.
- **A2A:** no claim.

unowned-rationale: filed from the census as an AC-3 leaf, and open to any Brain seat. It is an ADR-0019 surface, so read the ADR first.

Retrieval Hint: "Brain readers declared corpus root fallback projectRoot KB inferSourceType"

Origin Session ID: 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-09-24T14:10:17Z @neo-opus-grace added the `enhancement` label
- 2026-09-24T14:10:18Z @neo-opus-grace added the `ai` label
- 2026-09-24T14:10:18Z @neo-opus-grace added the `agent-os` label
- 2026-09-24T14:10:19Z @neo-opus-grace added parent issue #17416
- 2026-09-24T14:11:13Z @neo-opus-grace cross-referenced by #19158
- 2026-09-24T14:12:02Z @neo-opus-grace cross-referenced by PR #19167
### @neo-opus-grace - 2026-09-24T16:34:41Z

**Coupling, from D#19151's STEP_BACK ([DC_kwDODSospM4BG5MU](https://github.com/neomjs/neo/discussions/19151#discussioncomment-18584340)):** `IssueIngestor.mjs` is also edited by D#19151's H3 Brain sub, which projects `author` and `assignees` onto ISSUE and PULL_REQUEST nodes. Sequence the two, or land them in one PR.

The repoint here must also keep the ingest single-origin:
- Graph ids are origin-implicit: `issue-${n}` and `pr-${id}` (`IssueIngestor.mjs:372,764,821`).
- Non-`neo` origins collide on bare numbers.
- So a multi-origin content root needs the ADR 0004 §3.2.1 `(repoSlug, type, id)` ids first.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2


- 2026-09-24T20:34:12Z @neo-opus-vega cross-referenced by #471
- 2026-09-24T20:37:09Z @neo-opus-vega cross-referenced by #474
- 2026-09-25T11:05:42Z @neo-fable-clio cross-referenced by #482
- 2026-09-25T12:14:36Z @neo-opus-grace cross-referenced by #485
- 2026-09-25T14:13:56Z @neo-opus-grace cross-referenced by #490
- 2026-09-25T14:33:06Z @neo-opus-vega cross-referenced by PR #491
- 2026-09-26T07:26:59Z @neo-fable-clio cross-referenced by #533
- 2026-09-26T07:27:14Z @neo-fable-clio cross-referenced by #534
### @neo-opus-grace - 2026-09-26T08:52:10Z

**Order with #534 (AC-3 there): #534 lands first.** Both touch `ai/services/ingestion/IssueIngestor.mjs`. #534 adds `author` / `assignees` to the ISSUE and PULL_REQUEST upserts in `ingestIssueStates` and `ingestPullRequestFeedback`. This ticket's content-root fallback rebases onto that. The two changes sit in different hunks (the upsert properties vs the root resolution), so the rebase is mechanical.

Origin Session ID: 81d1894c-d8fd-4192-8350-42e32eb0101e


- 2026-09-26T08:55:43Z @neo-opus-grace cross-referenced by PR #542

