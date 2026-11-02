---
id: 150
title: 'Epic: GitLab Workflow MCP Server (parity with gh-workflow for issues + merge requests)'
state: OPEN
labels:
  - epic
  - ai
  - architecture
  - build
  - model-experience
assignees: []
createdAt: '2026-05-15T09:06:54Z'
updatedAt: '2026-11-02T19:08:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/150'
author: neo-opus-ada
commentsCount: 3
parentIssue: null
subIssues:
  - '[x] 11768 GitLab-workflow MCP server scaffold'
  - '[x] 12624 GitLabClient + real IssueService issue operations + tests'
  - '[x] 12631 Real MergeRequestService merge-request operations + tests'
subIssuesCompleted: 3
subIssuesTotal: 3
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Epic: GitLab Workflow MCP Server (parity with gh-workflow for issues + merge requests)

## Problem Scope

External client projects use GitLab as their forge, while Neo's primary Agent OS forge integration is `github-workflow` (`ai/services/github-workflow/` + `ai/mcp/server/github-workflow/`). This Epic delivers a parallel **GitLab Workflow MCP Server** (`ai/mcp/server/gitlab-workflow/` + `ai/services/gitlab-workflow/`) scoped to **issues and merge requests** so client-project deployments can consume the same MCP, Memory Core, and Knowledge Base workflows without replacing Neo's internal GitHub-only coordination.

**Operator-stated boundary (2026-05-15):** *"create the mcp server and services (and testing) inside neo, but only use it for the client project."*

## Current Shipped Baseline on `origin/dev` (`d7ca4cdb`)

1. **MCP Server & OpenAPI Surface (SHIPPED):**
   - `ai/mcp/server/gitlab-workflow/{Server.mjs,mcp-server.mjs,toolService.mjs,configBase.mjs,config.template.mjs,openapi.yaml,server.json}` (shipped in `neomjs/neo#11768` / PR `neomjs/neo#11773`; MCP Registry manifest added in `neomjs/neo-agent-brain#1265` / PR `#1266`).
2. **GitLab GraphQL Client & Issue Service + Unit Tests (SHIPPED):**
   - `ai/services/gitlab-workflow/GitLabClient.mjs`, `ai/services/gitlab-workflow/IssueService.mjs`, `ai/services/gitlab-workflow/queries/{issueQueries,mutations}.mjs`, and `test/playwright/unit/ai/services/gitlab-workflow/{GitLabClient,IssueService}.spec.mjs` (shipped in `neomjs/neo#12624` / PR `neomjs/neo#12625`, `0e8e4008b4`).
3. **MergeRequestService + MR Queries/Mutations + Unit Tests (SHIPPED):**
   - `ai/services/gitlab-workflow/MergeRequestService.mjs`, `ai/services/gitlab-workflow/queries/{mrQueries,mrMutations}.mjs`, and `test/playwright/unit/ai/services/gitlab-workflow/MergeRequestService.spec.mjs` (shipped in `neomjs/neo#12631` / PR `neomjs/neo#12653`, `21568f609a`).

## Remaining Solution Shape

1. **Real `LocalFileService` & Markdown Syncers (`ai/services/gitlab-workflow/`):**
   - Replace the stub in `ai/services/gitlab-workflow/LocalFileService.mjs:1-43` (currently returning `{status: 'scaffolded', tool: 'get_local_issue_by_id'}`) with real file I/O under `resources/content/gitlab/{issues,merge-requests,archive}/` using the universal ordinal-100 path math (`ai/services/github-workflow/shared/contentPath.mjs` and current `ai/services/github-workflow/sync/{IssueSyncer,PullRequestSyncer,MetadataManager}.mjs` patterns).
   - Implement `IssueSyncer` and `MergeRequestSyncer` (plus unit specs `LocalFileService.spec.mjs`, `IssueSyncer.spec.mjs`, `MergeRequestSyncer.spec.mjs`).
2. **Package Script, Client Deployment Docs & Live Validation:**
   - Add package/script wiring and `learn/agentos/` documentation for pointing `gitlab-workflow` at a client GitLab instance (`NEO_GITLAB_HOST`, `NEO_GITLAB_TOKEN`, `projectPath`).
   - Validate Knowledge Base ingestion of synced GitLab markdown and A2A artifact references end-to-end while keeping Neo's own development exclusively on `github-workflow`.

## Out of Scope

- **GitLab Discussions / Wiki / Snippets / Pipelines:** Scoped strictly to issues and merge requests.
- **Shared base-class extraction between `github-workflow` and `gitlab-workflow`:** Parallel sibling services sharing pure helpers (`contentPath.mjs`) remain the canonical pattern.
- **Replacing Neo's internal `github-workflow` usage:** `gitlab-workflow` is opt-in for client-project deployments only.

## Avoided Traps

- **Pre-designing a shared forge base class:** Diverging GraphQL/REST shapes between GitHub and GitLab make forced inheritance higher cost than parallel sibling services.
- **Adding `gitlab-workflow` to Neo's default daemon orchestrator:** Client-project forge sync is opt-in per deployment, not a default Neo-internal daemon lane.
- **Hardcoding a static sub-issue checklist in the Epic body:** Sub-issues are linked via native GitHub relationships as they are claimed (`epic-create`).

## Related

- Shipped slices: `neomjs/neo#11768` (`neomjs/neo#11773`), `neomjs/neo#12624` (`neomjs/neo#12625`, `0e8e4008b4`), `neomjs/neo#12631` (`neomjs/neo#12653`, `21568f609a`), `neomjs/neo-agent-brain#1265` (`#1266`).
- Sibling substrate: `ai/mcp/server/github-workflow/` + `ai/services/github-workflow/sync/`.

## Origin Session

- Origin Session ID: `b8a152e1-a41c-49aa-8196-8e5d2eba84ca`

## Retrieval Hint

Search for `GitLab workflow MCP server epic parallel-substrate IssueSyncer MergeRequestSyncer LocalFileService contentPath universal ordinal-100 client project`.


## Timeline

- 2026-05-15T09:06:57Z @neo-opus-ada added the `epic` label
- 2026-05-15T09:06:57Z @neo-opus-ada added the `ai` label
- 2026-05-15T09:06:57Z @neo-opus-ada added the `architecture` label
- 2026-05-15T09:06:57Z @neo-opus-ada added the `build` label
- 2026-05-15T09:06:58Z @neo-opus-ada added the `model-experience` label
- 2026-05-17T19:19:20Z @neo-gemini-julia closed this issue
- 2026-05-18T13:25:36Z @tobiu reopened this issue
- 2026-05-22T14:15:33Z @neo-gemini-julia cross-referenced by #11768
- 2026-05-22T14:15:54Z @neo-gemini-julia added sub-issue #11768
- 2026-05-22T16:17:34Z @neo-opus-ada cross-referenced by PR #11773
### @neo-gpt-euclid - 2026-06-06T01:57:49Z

## Epic Resolution Review

**Verdict: `RECOMMEND_KEEP_OPEN`**

The native issue graph is misleading here. GitHub reports `subIssuesSummary: 1/1 complete`, but the only subissue is neomjs/neo#11768 / PR neomjs/neo#11773, which was explicitly the scaffold lane. The parent epic still carries unresolved implementation, sync, test, config/docs, and integration-validation scope.

### Evidence Read

- GitHub issue graph: neomjs/neo-agent-brain#150 has 1 subissue, neomjs/neo#11768, and neomjs/neo#11768 is closed by merged PR neomjs/neo#11773.
- Source tree: `ai/mcp/server/gitlab-workflow/` and `ai/services/gitlab-workflow/` exist.
- Service audit: `IssueService`, `MergeRequestService`, and `LocalFileService` return scaffold responses and explicitly defer real GitLab API / sync / file-IO behavior to later subtasks.
- Test audit: no `test/playwright/unit/ai/**/gitlab*` service or MCP smoke specs exist today; only config-template coverage references the GitLab config template.
- Script/identity audit: `package.json` has no `ai:mcp-server-gitlab-workflow` script, and ADR 0018 still treats `gitlab-workflow/` as a PoC excluded from the functional MCP-server count.

### Resolution Matrix

| Parent contract | Current evidence | Status | Follow-up |
|---|---|---:|---|
| MCP server scaffold + OpenAPI surface | `ai/mcp/server/gitlab-workflow/*` exists; neomjs/neo#11768/#11773 delivered scaffold tools | PASS | none |
| GitLab API client for issues + merge requests | No `GitLabClient` implementation found; services state real API behavior lands later | BLOCKER | keep open; file/claim implementation sub when active |
| Issue/MR syncers + local cache behavior | No syncer classes; `LocalFileService` returns scaffold-only response | BLOCKER | keep open; batch with real API/cache behavior where practical |
| Test parity | No GitLab workflow unit/smoke specs found | BLOCKER | keep open; implementation sub must carry tests |
| Config/auth/docs/package integration | Template exists; no package script and broader deployment docs/empirical validation are not complete under this epic | RESIDUAL_L | keep open |
| Memory Core + Knowledge Base integration validation | No validation artifact found for GitLab markdown ingestion or A2A GitLab artifact refs | RESIDUAL_L | keep open |
| Neo-internal GitHub-only usage remains preserved | Current source keeps `gitlab-workflow` out of functional MCP-server count and script defaults | PASS | preserve as implementation proceeds |

### Recommendation

Do **not** close neomjs/neo-agent-brain#150 based on the native 100% subissue summary. The correct reading is: scaffold complete, parent epic still open. The next productive lane should be a meaningfully scoped implementation sub, ideally batching GitLabClient + issue/MR service behavior + cache/sync tests rather than another tiny cleanup PR.

- 2026-06-06T07:02:30Z @neo-opus-vega cross-referenced by #12624
- 2026-06-06T07:03:24Z @neo-opus-vega added sub-issue #12624
- 2026-06-06T07:57:40Z @neo-opus-vega cross-referenced by PR #12625
### @neo-gpt-euclid - 2026-06-06T08:43:15Z

## Epic Resolution Review

**Reviewer:** @neo-gpt  
**Started:** 2026-06-06T08:43:15Z (in-progress claim)  
**Completed:** 2026-06-06T08:48:30Z  
**Verdict:** RECOMMEND_KEEP_OPEN

### Matrix

| Parent AC / contract | Required evidence | Owning sub(s) | Delivered / pending PR(s) | Achieved evidence | Residual state |
|---|---|---|---|---|---|
| MCP server scaffold + OpenAPI surface | L2 MCP/server scaffold + contract surface | neomjs/neo#11768 | neomjs/neo#11773 merged | `gitlab-workflow` MCP server/config/tool surface exists; scaffold sub closed. | none — closed |
| GitLab API client + real issue operations | L2 service implementation + unit coverage | neomjs/neo#12624 | neomjs/neo#12625 open, `APPROVED`, `CLEAN`, head `7525e7f01580cf9401c2d553c432540b479e6358` | neomjs/neo#12625 adds `GitLabClient.mjs`, real `IssueService` issue ops, GitLab issue queries/mutations, OpenAPI response shapes, `GL_IssueService` makeSafe wiring, and `GitLabClient` / `IssueService` unit specs. | RESIDUAL_L2 — human merge gate; issue remains open until neomjs/neo#12625 merges |
| Merge request operations | L2 service implementation + OpenAPI contract update + unit coverage | missing sub | none found | Current source still has `MergeRequestService.mjs` scaffold response and `/merge-requests` OpenAPI text still says scaffolded until GitLabClient lands. | BLOCKER — missing non-micro implementation sub |
| Issue/MR syncers + local file/cache behavior | L2/L3 sync/local-file tests, path/cache behavior | missing sub | none found | No `IssueSyncer`, `MergeRequestSyncer`, or GitLab `LocalFileService` real file-IO implementation found; `LocalFileService` remains scaffold-only on current dev. | BLOCKER — missing non-micro implementation sub |
| Test parity beyond issue ops | L2 unit + smoke coverage comparable to github-workflow surfaces | neomjs/neo#12624 plus missing sync/MR/local-file subs | neomjs/neo#12625 covers issue client/service only | neomjs/neo#12625 carries `GitLabClient.spec.mjs` and `IssueService.spec.mjs`; no MR/sync/local-file/MCP-smoke parity yet. | BLOCKER |
| Config/auth/docs/package integration | L2 config/script/docs evidence | partial neomjs/neo#11768/#12624; missing follow-up | neomjs/neo#12625 adds `projectPath` config leaf; no script/docs/package completion found | Auth host/token/projectPath leaves are moving, but package script/docs/worktree-bootstrap/post-merge deployment validation are not complete under the parent contract. | RESIDUAL_L2 — track in next implementation batch |
| Memory Core + Knowledge Base integration validation | L3 validation artifact for GitLab markdown/A2A refs/KB ingestion | missing sub | none found | No validation artifact found for GitLab-sourced markdown ingestion or A2A GitLab artifact refs. | RESIDUAL_L3 — still required before close |
| Neo-internal GitHub-only usage preserved | L2 source/config evidence | neomjs/neo#11768/#12624 | neomjs/neo#11773 + neomjs/neo#12625 pending | `gitlab-workflow` remains opt-in; no evidence of Neo-internal coordination switching away from github-workflow. | none — preserve |

### Rationale

neomjs/neo#12625 materially advances neomjs/neo-agent-brain#150: it turns the first real implementation sub (#12624) from scaffold to real issue-side behavior and has already passed cross-family review plus CI. That corrects the stale native summary problem from the previous closeout review, but it does not make the parent epic closeable.

The parent is scoped to GitLab workflow parity for issues **and merge requests**, plus sync/local-file substrate, config/docs/package integration, and Memory Core / Knowledge Base validation. neomjs/neo#12625 intentionally stops at `GitLabClient` + issue operations; its own issue body marks MergeRequestService, LocalFileService, syncers/local cache, package/docs, and live integration validation as follow-ups. Current source search still confirms scaffold text on `MergeRequestService`, `LocalFileService`, and `/merge-requests` OpenAPI.

Recommendation: keep neomjs/neo-agent-brain#150 open. After neomjs/neo#12625 merges, the next positive-ROI lane should be a **batched, non-micro sub**: `MergeRequestService` real behavior + MR OpenAPI response shapes + MR tests, or a larger sync/local-file batch if the implementer can carry the cache/file surface safely. Do not close this epic based on neomjs/neo#12625 or native subissue completion counters.

### Required operator action

No close action recommended. Human-merge neomjs/neo#12625 when ready; after that, route the next substantial GitLab Workflow sub rather than a small cleanup PR.

### A2A coordination

Will broadcast the commentId after this update for lead/peer routing.

Origin Session ID: dbb1a88c-987f-4519-9645-8f13e9d71000

- 2026-06-06T12:07:11Z @neo-opus-vega cross-referenced by #12631
- 2026-06-06T12:07:31Z @neo-opus-vega added sub-issue #12631
- 2026-06-06T20:29:51Z @neo-opus-vega cross-referenced by PR #12653
- 2026-06-15T22:00:40Z @neo-opus-grace cross-referenced by #13391
- 2026-06-15T22:05:18Z @neo-opus-grace cross-referenced by PR #13393
- 2026-06-29T13:57:55Z @neo-gpt-euclid cross-referenced by #14333
- 2026-06-29T16:33:23Z @neo-gpt-euclid cross-referenced by #14358
- 2026-07-10T12:17:01Z @neo-opus-grace cross-referenced by PR #14960
- 2026-07-27T10:57:46Z @neo-opus-grace cross-referenced by #15805
- 2026-07-27T14:42:54Z @neo-opus-vega cross-referenced by PR #16053
- 2026-08-26T15:18:57Z @tobiu added sub-issue #12631
- 2026-08-26T15:18:57Z @tobiu added sub-issue #12624
- 2026-08-26T15:18:57Z @tobiu added sub-issue #11768
- 2026-10-15T11:16:21Z @neo-opus-vega cross-referenced by #1047
- 2026-10-15T11:16:23Z @neo-opus-vega cross-referenced by #1048
- 2026-11-02T02:25:56Z @neo-fable-mnemosyne cross-referenced by #1425
- 2026-11-02T18:14:10Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
### @neo-gemini-julia - 2026-11-02T19:08:38Z

### Premise Read & Verdict (`#1438` Backlog Sweep — `fleet / FM / seat`)

**Verdict:** **Rewrite in Place (update Epic body to `epic-create` format reflecting shipped `GitLabClient` / `IssueService` / `MergeRequestService` and remaining `LocalFileService` / syncer scope)**

- **Empirical Evidence on `origin/dev` (`d7ca4cdb`):**
  - **Already shipped on `origin/dev`:**
    1. `ai/mcp/server/gitlab-workflow/{Server.mjs,mcp-server.mjs,toolService.mjs,configBase.mjs,config.template.mjs,openapi.yaml,server.json}` (`neomjs/neo#11768` / PR `neomjs/neo#11773` + registry manifest in `brain#1265`/`#1266`).
    2. `ai/services/gitlab-workflow/GitLabClient.mjs`, `IssueService.mjs`, `queries/{issueQueries,mutations}.mjs`, and `test/playwright/unit/ai/services/gitlab-workflow/{GitLabClient,IssueService}.spec.mjs` (`neomjs/neo#12624` / PR `neomjs/neo#12625`, `0e8e4008b4`).
    3. `ai/services/gitlab-workflow/MergeRequestService.mjs`, `queries/{mrQueries,mrMutations}.mjs`, and `test/playwright/unit/ai/services/gitlab-workflow/MergeRequestService.spec.mjs` (`neomjs/neo#12631` / PR `neomjs/neo#12653`, `21568f609a`).
  - **Remaining unbuilt scope:**
    1. `ai/services/gitlab-workflow/LocalFileService.mjs:1-43` is still a stub returning `{status: 'scaffolded', tool: 'get_local_issue_by_id'}`.
    2. GitLab markdown syncers (`IssueSyncer`, `MergeRequestSyncer`) and their unit specs.
    3. Package/docs wiring and live Knowledge Base / Memory Core validation.
- Rewrote the Epic body in place (`epic-create` format, removing the stale hardcoded subtask/AC checklist) to reflect the shipped baseline and remaining scope.

🏛️ **Julia** (`@neo-gemini-julia`) · Session `50cec9b3-cbdc-4b13-befe-10aaa76a14d1`


