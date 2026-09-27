---
id: 558
title: 'explore_pull_request_history takes an origin: the PR bird view serves each tenant repository'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-26T20:54:48Z'
updatedAt: '2026-09-27T08:14:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/558'
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
closedAt: '2026-09-27T08:14:24Z'
---
# explore_pull_request_history takes an origin: the PR bird view serves each tenant repository

## Context

Operator, 2026-09-26: *`explore_pull_request_history` is a good topic — we created it before the repo split. Can it even work with multiple tenant repos inside the Agent OS?* It cannot. The plane now carries six tenant repositories (`neo-shared/neo`, `create-app`, `devindex`, `devindex-opt-in`, `devindex-opt-out`, `github-content-sync`) and a per-origin corpus (`_index.json` plus one directory per slug: `neo`, `neo-agent-brain`, `neo-agent-institution`, `neo-agent-skills`, `devindex`), while the bird view still answers for exactly one repository fixed at server start.

## The Problem

- `PullRequestHistoryService#explorePullRequestHistory(options, deps)` takes `owner` / `repo` from its dependency bag with `aiConfig.owner` / `aiConfig.repo` as defaults (`ai/services/github-workflow/PullRequestHistoryService.mjs:1372–1373`), and the Memory Core op (`ai/mcp/server/memory-core/toolService.mjs:229`) supplies only `runTemporal`, `generate` and `now`. Every call therefore runs against the plane's single configured repository, `neomjs/neo` by the pre-split default.
- The caller's `options` carry only the window (`preset`, `windowStart` / `windowEnd`, `resolution`, `release`); the tool schema has no origin parameter, so no client can ask for another repository.
- The corpus half reads one root: `pullsDir` / `archiveRoot` from `aiConfig.issueSync`, the pre-split `pr-N.md` layout. The tenant corpus is per origin, `<contentRoot>/<repoSlug>/pulls`, the shape #413 gave the fleet activity feed (`wireFleetActivityReadSource.mjs#resolveContentOrigins`, `:64–95`).
- Every returned drill-down is `get_conversation {pr_number}`: repo-implicit, so a view over a second repository would hand back handles that open the configured one at the same number.

## The Architectural Reality

- The GitHub reads are live and already parameterized: the resolved-PR search is `repo:${owner}/${repo} is:pr is:closed closed:…` (`:82`), the review-comment passes hit `/repos/${owner}/${repo}/pulls/…` (`:440`, `:516`), and `resolveReleaseWindow({release, query, owner, repo})` resolves a release tag per repository (`:828`). Any repository the plane's token can read works once a repository is passed.
- The temporal bird-view cache is partitioned `repository:${owner}/${repo}:${resolution}` (`:1378`), so per-origin results never collide.
- The service persists no graph node keyed by a bare PR number (a corpus scan records `pr-N.md` paths only, `:695`), so ADR 0004 §3.2.1's origin-qualified id rule (D#19051: `issue-N` / `pr-N` ids collide across origins) constrains the ingested graph, not this read.
- The corpus `_index.json` slugs are the plane's origin catalog; the activity feed already reads them (`resolveContentOrigins`). `get_conversation` already takes `repo` as a `RepositoryTarget` (`owner/name`).

## The Fix

1. Tool schema and op: an optional `origin` argument (`owner/repo`) on `explore_pull_request_history`; the op passes it through. Absent, the AiConfig repository applies, so a single-repository plane is unchanged.
2. Validation: an origin is known when the plane's corpus catalog names it — `resolveContentOrigins(fleet.contentRoot)`, moved to `ai/services/graph/contentOrigins.mjs` and read identically by the fleet activity feed and this view — or when it is the configured repository itself. `.` and `..` are refused before the catalog is read; the request never builds a path (a catalog row is selected by exact slug equality). Anything else is refused by name (`unknown-origin`, the origin echoed), never silently answered from the default. A catalogued origin whose tree is missing is known and its scan degrades by name, as the activity feed's does. *(Corrected 2026-09-27: the first draft also admitted tenant-registry entries; the registry names `neo-shared/*` mirrors whose GitHub reads would target another owner, so it is not an admission source. PR #560 Round 1 replaced directory existence with the catalog.)*
3. Corpus paths per origin: the catalog row's `pulls` directory and its sibling `archive`; the configured repository outside the catalog keeps the pre-split roots.
4. Every returned drill-down names its repository: `get_conversation {pr_number, repo: 'owner/repo'}`, for the configured origin too.
5. The tool description names the argument, the refusal and the drill-down rule. *(Corrected 2026-09-27: the Memory Core handbook entry **is** the OpenAPI operation description — `ToolService#getToolHandbook` serves the OpenAPI-derived mapping — so there is no separate KB entry to author.)*

## Contract Ledger Matrix

Public surface of `explore_pull_request_history` (Memory Core MCP tool). Added 2026-09-27 at the reviewer's request on PR #560.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `origin` (query, optional) | `resolvePullRequestOrigin` (`PullRequestHistoryService.mjs`), run before any read | `owner/repo`; the owner must equal the configured owner; the repo must be a catalog slug (`resolveContentOrigins(fleet.contentRoot)`) or the configured repo | absent / `null` / `''` → the configured repository and roots, no catalog read | `openapi.yaml` `origin` description (the handbook entry) | `PullRequestHistoryService.spec.mjs` fixture + resolver arms |
| refusal | same | `Error` with `code: 'unknown-origin'` and `origin` echoed: malformed form, `.` / `..` (refused before the catalog is read), foreign owner, a name the catalog lacks (an existing but unindexed directory included) | none — never the default repository | same | the refusal arm (no query, no scan runs) + resolver arm (red controls) |
| corpus roots | the catalog row | catalogued origin → `<row.pullsDir>` + `<dirname(row.pullsDir)>/archive` (per-slug and pre-split layouts alike); a missing tree scans as `missingRoots` and degrades by name | configured repo outside the catalog → `issueSync.pullsDir` / `archiveRoot` | resolver JSDoc | the two-slug fixture arm with the real `scanPullRequestCorpus` and real `resolveContentOrigins` |
| live reads + release window + cache partition | existing `owner` / `repo` plumbing | the search names `repo:owner/repo`, `FETCH_RELEASES_FOR_HISTORY` receives the origin's `{owner, repo}`, the partition is `repository:owner/repo:resolution` | unchanged for the default path | operation description | the fixture arm (searches, partitions) + the origin-plus-release arm |
| `citations[].drillDown` | `buildPullRequestSource` | `{operation: 'get_conversation', arguments: {pr_number, repo: 'owner/repo'}}` for every origin, the configured one included | none: always qualified | operation description | the fixture arm (two repositories, the same PR number, each handle names its own) + the revision arm |
| failure behaviour | the op | an `unknown-origin` refusal surfaces as the tool error; nothing is cached or written | — | — | the refusal arm |

## Acceptance Criteria

- [ ] AC-1: `explore_pull_request_history({preset: 'month', origin: 'neomjs/neo-agent-brain'})` returns that repository's bird view — its resolved PRs from the live search, corpus rows from the catalog's `neo-agent-brain` roots, a cache partition naming `repository:neomjs/neo-agent-brain:…`, and drill-downs naming `neomjs/neo-agent-brain`. **L2 (integration arm):** a two-slug fixture corpus with a real `_index.json`, the real catalog reader and the real corpus scanner, two repositories holding the same PR number. **[L4-deferred — operator handoff needed]** the plane receipt's home is #64 AC-8. Measured 2026-09-26 22:0xZ: the mc-server container has no corpus content root (`fleet.contentRoot` defaults to `/app/resources/content`, absent there; the plane's only `_index.json` is the orchestrator's single-origin materialized root), so today the plane refuses every origin but the configured one — correctly; the positive receipt needs the multi-origin corpus reachable from mc-server, a mount the operator decides.
- [ ] AC-2: the same call without `origin` returns exactly today's result for the configured repository (unit arm pinning the default path with a catalog reader that must not be called; every pre-existing arm runs without an origin).
- [ ] AC-3: an origin the catalog does not name is refused with `unknown-origin` and the origin echoed, and never falls back to the default (unit arms, red-first: the refusal before any read; `.` / `..` before the catalog read; a foreign owner; an unindexed directory with a real tree).
- [ ] AC-4: the `release` preset resolves the tag against the requested origin, not the configured one (unit arm: `FETCH_RELEASES_FOR_HISTORY` receives `{owner: 'neomjs', repo: 'neo-agent-brain'}` and the window resolves from that repository's tags).

## Out of Scope

- One bird view across several repositories at once (cross-origin ranking, merged windows, per-origin quotas): a product question for the Ideation Sandbox if wanted; this ticket makes each origin addressable.
- The live-feed question for the cockpit's activity stream (#459): this tool stays a synthesis over resolved PRs.
- Origin-qualified graph ids for ingested issues and PRs (ADR 0004 §3.2.1; D#19051).
- Mounting the multi-origin corpus into the mc-server container (the AC-1 receipt's precondition): a plane decision, recorded on #64 AC-8.

## Decision Record impact

aligned-with ADR 0004 §3.2.1 (origins are explicit, never implicit).

## Related

#413 (the activity feed's per-origin corpus read) · #387 (origin-qualified corpus emit) · #459 · #64 (AC-8, the plane receipt) · D#19051 · defect-note fingerprint `47eb7b52b72a9eb4` (2026-09-26 20:48Z, this finding's capture)

Live latest-open sweep: checked the latest 20 open Brain issues at 20:53Z; no equivalent. A2A claim sweep: the only item on this scope is my own defect-note of 20:48Z. Memory sweep: `query_raw_memories` on the tool and tenant nouns returned no prior decision. Own-assignment sweep: none of my open tickets covers it. Structure map: `ai/services/graph/contentOrigins.mjs` (the shared catalog reader, moved out of the fleet module); the rest lives in `ai/services/github-workflow/` and `ai/mcp/server/memory-core/`.

Origin Session ID: 27467eea-851e-486b-a0ca-55f744b67fdf
Retrieval Hint: "explore_pull_request_history origin owner repo tenant corpus catalog resolveContentOrigins unknown-origin refusal drill-down repo cache partition"


## Timeline

- 2026-09-26T20:54:48Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-26T20:54:49Z @neo-opus-vega added the `enhancement` label
- 2026-09-26T20:54:49Z @neo-opus-vega added the `ai` label
- 2026-09-26T21:20:20Z @neo-opus-vega cross-referenced by PR #560
- 2026-09-26T21:28:18Z @neo-opus-vega referenced in commit `13cb262` - "feat(memory-core): explore_pull_request_history takes an origin, so the PR bird view serves each tenant repository (#558)

The tool ran against the one repository AiConfig named at server start and read one
pre-split corpus root. An optional origin (owner/repo) now selects the repository: the
owner must be the plane's configured owner, the repository a corpus origin under
fleet.contentRoot or the configured repository itself, and the corpus roots become that
slug's pulls and archive directories; anything else is refused as unknown-origin before
any read. Absent, nothing changes. The live reads and the cache partition were already
repo-qualified."
- 2026-09-26T23:11:51Z @neo-opus-vega referenced in commit `56eb483` - "feat(memory-core): explore_pull_request_history admits origins from the corpus catalog and its drill-downs name their repository (#558)"
- 2026-09-26T23:11:53Z @neo-opus-vega cross-referenced by #64
- 2026-09-26T23:17:01Z @neo-opus-vega cross-referenced by #459
- 2026-09-27T08:14:24Z @tobiu referenced in commit `4bc885b` - "Merge pull request #560 from neomjs/vega/558-pr-bird-view-origin

feat(memory-core): explore_pull_request_history takes an origin, so the PR bird view serves each tenant repository (#558)"
- 2026-09-27T08:14:24Z @tobiu closed this issue

