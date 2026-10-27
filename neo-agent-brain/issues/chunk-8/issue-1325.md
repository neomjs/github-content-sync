---
id: 1325
title: Enforce mechanical 25-comment ceiling in github-workflow MCP tools
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-10-27T22:38:27Z'
updatedAt: '2026-10-27T22:38:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1325'
author: neo-gemini-julia
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
---
# Enforce mechanical 25-comment ceiling in github-workflow MCP tools

## Context

During `neomjs/neo-stage#443`, `@neo-gemini-julia` posted a full-figure bearer statement as comment #71 on `neomjs/neo-stage#5` (which had already accumulated 70 comments across appearance statements, OQ-7 debates, and wardrobe sandboxes). Following that comment, an unscoped `get_conversation` read on `#5` pulled all 70+ comments into the active context window at once, triggering immediate context compaction. `@tobiu` flagged this as a recurring team failure and `Friction -> Gold` topic:

> *"adding comments to a ticket that already has 70 comments is just absurd. it should never ever be more than 25 comments. token context window nuke bombs. this is a recurring team failure and friction->gold topic."*

Comment `6454919556` on `neomjs/neo-stage#5` was immediately deleted and folded into the body of `neomjs/neo-stage#443`. This ticket codifies the mechanical 25-comment ceiling in `neo-mjs-github-workflow` so mega-threads cannot be created or read as unscoped 100-comment context bombs.

- **Live latest-open sweep:** checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1299` down to `#875`) at `2026-10-27T22:36:32Z`; no equivalent found.
- **A2A in-flight claim sweep:** checked `list_messages({status: 'all', limit: 30})` at `2026-10-27T22:36:34Z`; no competing `[lane-claim]` or `[lane-intent]` on `github-workflow` comment caps.
- **MC sweep:** `query_raw_memories({query: "github-workflow manage_issue_comment get_conversation 25 comments cap maxCommentsPerPullRequest context window", nResults: 6})`, 6 results, no prior decision found; `gh search issues --owner neomjs "maxCommentsPerPullRequest"`, 0 results.
- **Own-assignment sweep:** 0 open issues assigned to `@neo-gemini-julia` in `neomjs/neo-agent-brain`, none overlapping.
- **Structure Map Gate (`§1c`):** ran `npm run ai:structure-map -- --files --loc`; owning surfaces are `ai/mcp/server/github-workflow/configBase.mjs` (`352` lines), `ai/mcp/server/github-workflow/openapi.yaml`, `ai/services/github-workflow/IssueService.mjs` (`1816` lines), `ai/services/github-workflow/DiscussionService.mjs` (`479` lines), `ai/services/github-workflow/PullRequestService.mjs` (`4848` lines), and `ai/services/github-workflow/queries/` (`mutations.mjs`, `pullRequestQueries.mjs`, `issueQueries.mjs`, `discussionQueries.mjs`).

## The Problem

1. **No write-side comment ceiling on `manage_issue_comment` (`action: 'create'`) or `manage_discussion_comment` (`action: 'create'`):**
   - `IssueService.createComment` (`ai/services/github-workflow/IssueService.mjs:407-468`) and `DiscussionService.createComment` (`ai/services/github-workflow/DiscussionService.mjs:242-282`) allow agents to append comments indefinitely to an issue, PR, or discussion regardless of how many comments already exist.
   - Prose guidance alone (`#16602`, `CommentGuidanceTiers.spec.mjs`: *"corrections go in the BODY, not a comment"*) warns against using comments for state corrections, but does not stop multi-agent dialogue or precedent-following from turning long-lived issues or discussions into 70-comment mega-threads.
2. **Unscoped read ceiling is `100` (`maxCommentsPerPullRequest: leaf(100)`):**
   - `ai/mcp/server/github-workflow/configBase.mjs:315` sets `pullRequest.maxCommentsPerPullRequest: leaf(100)`.
   - `PullRequestService.getConversation` (`:3819`), `IssueService.getConversation` (`:105`), and `DiscussionService.getConversation` (`:96-97`) all pass `aiConfig.pullRequest.maxCommentsPerPullRequest` to `comments(first: $maxComments)`.
   - An unscoped `get_conversation` or `get_discussion_conversation` call on a 70-comment thread dumps up to 100 full comment bodies into the caller's context window in one tool response — a context-window nuke bomb (especially on 258k-token harnesses).
   - Furthermore, because `GET_CONVERSATION` (`pullRequestQueries.mjs:26`), `GET_ISSUE_CONVERSATION` (`issueQueries.mjs:954`), and `GET_DISCUSSION_CONVERSATION` (`discussionQueries.mjs:84`) use `comments(first: $maxComments)` (oldest first), lowering `maxCommentsPerPullRequest` to `25` without switching the conversation read queries to `comments(last: $maxComments)` would cause legacy `>25`-comment threads to return only their 25 *oldest* comments and miss recent handoff `comment_id` / `since_comment_id` / `last_n` selectors.

## The Architectural Reality

`Design authority: "Selectors are applied client-side after a single GraphQL fetch (the fetch itself already caps at aiConfig.pullRequest.maxCommentsPerPullRequest). Server-side pagination optimization is a follow-up concern if empirical volume demands it; for current conversation sizes (up to a few dozen comments) client-side filter is simpler and avoids multi-query cursor choreography."` (`ai/services/github-workflow/PullRequestService.mjs:3727-3731`; `ai/mcp/server/github-workflow/configBase.mjs:312-315`: `maxCommentsPerPullRequest: leaf(100)`).

- **Zero-extra-round-trip write gate:**
  - `IssueService.createComment` (`IssueService.mjs:437-443`) already executes `GET_PULL_REQUEST_ID` (`pullRequestQueries.mjs:660-667`) or `GET_ISSUE_ID` (`mutations.mjs:156-168`) before running `ADD_COMMENT`.
  - `DiscussionService.createComment` (`DiscussionService.mjs:247-261`) already executes `GET_DISCUSSION_ID` (`discussionQueries.mjs:37-45`) before running `ADD_DISCUSSION_COMMENT`.
  - Verified live via `gh api graphql`: selecting `comments { totalCount }` alongside `id` on `Issue`, `PullRequest`, and `Discussion` is valid in GitHub's GraphQL schema without fetching any comment nodes (`first`/`last` not required for `totalCount`).
  - Note that `IssueService.getIssueNodeId` (`IssueService.mjs:665-679`) also reuses `GET_ISSUE_ID` and reads `result.repository?.issue?.id || null`, so adding `comments { totalCount }` to `GET_ISSUE_ID` is backward-compatible with `getIssueNodeId` and `#createReassignAuditComment` (`IssueService.mjs:385-390`, which calls raw `ADD_COMMENT` for system-generated reassignment audit trails).
- **Read-side tail window (`25`):**
  - Lowering `pullRequest.maxCommentsPerPullRequest` in `ai/mcp/server/github-workflow/configBase.mjs:315` from `leaf(100)` to `leaf(25)` caps the per-thread comment budget at `25` across `IssueService`, `PullRequestService`, and `DiscussionService` (reading `aiConfig.pullRequest.maxCommentsPerPullRequest` directly at the use site per ADR 0019).
  - Switching `GET_CONVERSATION` (`pullRequestQueries.mjs:26`), `GET_ISSUE_CONVERSATION` (`issueQueries.mjs:954`), and `GET_DISCUSSION_CONVERSATION` (`discussionQueries.mjs:84`) from `comments(first: $maxComments)` (and `replies(first: $maxReplies)`) to `comments(last: $maxComments)` (and `replies(last: $maxReplies)`) plus `totalCount` ensures that:
    - On compliant threads (`<= 25` comments), all comments are returned in chronological order as before.
    - On legacy mega-threads (`> 25` comments), at most the latest 25 comments are returned (preserving `comment_id`, `since_comment_id`, and `last_n` resolution for recent comments while preventing >25-comment context bombs).

## The Fix

1. **`ai/mcp/server/github-workflow/configBase.mjs` (`:315`):**
   - Lower `pullRequest.maxCommentsPerPullRequest` from `leaf(100)` to `leaf(25)` and update its JSDoc to state that `25` is both the conversation read cap and the write-time comment ceiling for issues, PRs, and discussions.
2. **Pre-mutation ID queries (`mutations.mjs:156`, `pullRequestQueries.mjs:660`, `discussionQueries.mjs:37`):**
   - Add `comments { totalCount }` alongside `id` in `GET_ISSUE_ID` (`ai/services/github-workflow/queries/mutations.mjs:156-168`), `GET_PULL_REQUEST_ID` (`ai/services/github-workflow/queries/pullRequestQueries.mjs:660-667`), and `GET_DISCUSSION_ID` (`ai/services/github-workflow/queries/discussionQueries.mjs:37-45`).
3. **Write-time refusal in `IssueService.createComment` (`IssueService.mjs:407-468`) and `DiscussionService.createComment` (`DiscussionService.mjs:242-282`):**
   - After resolving the target resource from `idData`, check missing target first (`NOT_FOUND` if `!targetNode?.id`), then read `const commentCount = targetNode.comments?.totalCount ?? 0` and `const maxComments = aiConfig.pullRequest.maxCommentsPerPullRequest`.
   - When `commentCount >= maxComments`, refuse before `ADD_COMMENT` / `ADD_DISCUSSION_COMMENT` with:
     ```js
     {
         error       : 'Comment Cap Exceeded',
         code        : 'COMMENT_CAP_EXCEEDED',
         commentCount,
         maxComments,
         message     : `${label} #${number} already has ${commentCount} comments (hard cap: ${maxComments} to prevent context-window nuke bombs). Do not append to this thread: fold state/AC updates into the body, update an existing comment via action: 'update', or open a fresh linked issue/discussion.`
     }
     ```
   - Keep `action: 'update'` (`IssueService.updateComment`, `DiscussionService.updateComment`) and system reassignment audit comments (`#createReassignAuditComment`) unobstructed.
4. **Conversation read queries (`pullRequestQueries.mjs:26`, `issueQueries.mjs:954`, `discussionQueries.mjs:84`):**
   - Change `comments(first: $maxComments)` to `comments(last: $maxComments)` (and include `totalCount`) in `GET_CONVERSATION`, `GET_ISSUE_CONVERSATION`, and `GET_DISCUSSION_CONVERSATION` (plus `replies(last: $maxReplies)` in `GET_DISCUSSION_CONVERSATION`).
5. **OpenAPI & Tool Handbook (`ai/mcp/server/github-workflow/openapi.yaml`):**
   - Document the 25-comment ceiling (`COMMENT_CAP_EXCEEDED`) in `manage_issue_comment` and `manage_discussion_comment`, and the 25-comment tail bound in `get_conversation` and `get_discussion_conversation` (preserving the `#16602` `CommentGuidanceTiers.spec.mjs` invariants).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `aiConfig.pullRequest.maxCommentsPerPullRequest` (`ai/mcp/server/github-workflow/configBase.mjs:315`) | Operator mandate (`2026-10-27`) + ADR 0019 | Defaults to `leaf(25)` as the single SSOT ceiling for per-thread comment reads and writes. | Overridable only via sanctioned `AiConfig` leaf declaration. | Yes (`configBase.mjs` JSDoc) | `ConfigCompleteness.spec.mjs` + service unit tests |
| `IssueService.createComment` (`ai/services/github-workflow/IssueService.mjs:407-468`) | This ticket + `GET_ISSUE_ID` / `GET_PULL_REQUEST_ID` | Selects `comments { totalCount }` in the existing ID lookup; creates comment when `totalCount < 25`. | Returns `{ error: 'Comment Cap Exceeded', code: 'COMMENT_CAP_EXCEEDED', commentCount, maxComments, message }` when `totalCount >= 25` without calling `ADD_COMMENT`. | Yes (`IssueService.mjs` JSDoc + `openapi.yaml`) | `IssueService.spec.mjs` unit tests (under cap vs `>= 25` refusal for both issue and PR paths) |
| `DiscussionService.createComment` (`ai/services/github-workflow/DiscussionService.mjs:242-282`) | This ticket + `GET_DISCUSSION_ID` | Selects `comments { totalCount }` in `GET_DISCUSSION_ID`; creates comment when `totalCount < 25`. | Returns `{ error: 'Comment Cap Exceeded', code: 'COMMENT_CAP_EXCEEDED', commentCount, maxComments, message }` when `totalCount >= 25` without calling `ADD_DISCUSSION_COMMENT`. | Yes (`DiscussionService.mjs` JSDoc + `openapi.yaml`) | `DiscussionService.spec.mjs` unit tests (under cap vs `>= 25` refusal) |
| `GET_CONVERSATION`, `GET_ISSUE_CONVERSATION`, `GET_DISCUSSION_CONVERSATION` (`pullRequestQueries.mjs:26`, `issueQueries.mjs:954`, `discussionQueries.mjs:84`) | This ticket | Fetches `comments(last: $maxComments)` with `$maxComments = 25` and `totalCount`. | On legacy `>25`-comment threads, returns the latest 25 comments in chronological order so recent `comment_id` / `since_comment_id` / `last_n` selectors still resolve without loading 70+ comments. | Yes (`openapi.yaml` + query JSDoc) | `IssueService.spec.mjs`, `PullRequestService.spec.mjs`, `DiscussionService.spec.mjs` |

## Decision Record impact

`aligned-with ADR 0019` — updates the declared leaf default `pullRequest.maxCommentsPerPullRequest` in `ai/mcp/server/github-workflow/configBase.mjs:315` and reads `aiConfig.pullRequest.maxCommentsPerPullRequest` directly at the use site in `IssueService` and `DiscussionService` with no secondary resolver or module-scope copy.

## Acceptance Criteria

- [ ] `ai/mcp/server/github-workflow/configBase.mjs` sets `pullRequest.maxCommentsPerPullRequest: leaf(25)`.
- [ ] `GET_ISSUE_ID` (`mutations.mjs`), `GET_PULL_REQUEST_ID` (`pullRequestQueries.mjs`), and `GET_DISCUSSION_ID` (`discussionQueries.mjs`) select `comments { totalCount }` alongside `id` in the existing pre-mutation lookup.
- [ ] `IssueService.createComment` (for both `issue_number` and `pr_number`) and `DiscussionService.createComment` refuse with `code: 'COMMENT_CAP_EXCEEDED'` (`commentCount`, `maxComments`, actionable `message`) when `comments.totalCount >= aiConfig.pullRequest.maxCommentsPerPullRequest` (`25`), while keeping `action: 'update'` and `#createReassignAuditComment` working on threads of any length.
- [ ] `GET_CONVERSATION`, `GET_ISSUE_CONVERSATION`, and `GET_DISCUSSION_CONVERSATION` use `comments(last: $maxComments)` (and `replies(last: $maxReplies)` on discussions) plus `totalCount` so legacy `>25`-comment threads return at most the latest 25 comments.
- [ ] `ai/mcp/server/github-workflow/openapi.yaml` documents `COMMENT_CAP_EXCEEDED` and the 25-comment ceiling without regressing `CommentGuidanceTiers.spec.mjs`.
- [ ] Unit tests in `IssueService.spec.mjs`, `DiscussionService.spec.mjs`, `PullRequestService.spec.mjs`, and `CommentGuidanceTiers.spec.mjs` pass.

## Out of Scope

- Deleting or migrating historical comments on existing `>25`-comment closed or legacy threads.
- Modifying `manage_pr_review` formal review submission (`PullRequestReview` nodes are governed separately by the review-budget gate in `PullRequestService.mjs:3350-3500`).
- Changing the paginated corpus syncers (`IssueSyncer`, `DiscussionSyncer`), which paginate via `maxTimelineItems` / `FETCH_DISCUSSION_COMMENTS_PAGE`.

## Avoided Traps

- **Prose-only rule in `AGENTS.md` or `SKILL.md`:** Prose reminders do not stop an agent that does not know a thread's current comment count before calling `manage_issue_comment`. Enforcing the cap in `IssueService.createComment` and `DiscussionService.createComment` catches the violation mechanically at zero extra GraphQL round-trip cost.
- **Keeping `comments(first: $maxComments)` when lowering `$maxComments` to `25`:** On any existing thread with `>25` comments, `first: 25` fetches the 25 oldest comments and blinds `comment_id`, `since_comment_id`, and `last_n` to recent comments. Using `last: $maxComments` returns the latest 25 comments in chronological order.
- **Blocking `action: 'update'` on `>=25`-comment threads:** Agents must still be able to edit or collapse their own existing comments on legacy threads to reduce sediment.

## Related

- `neomjs/neo-stage#5` — 70-comment mega-thread incident anchor.
- `neomjs/neo-stage#443` — remediated bearer statement moved from `#5` comment into `#443` issue body.
- `neomjs/neo#16602` (`CommentGuidanceTiers.spec.mjs`) — `BODY = state` vs `COMMENT = dialogue` tool-description guidance.

unowned-rationale: Filed immediately as a Friction -> Gold mechanical guard while @neo-gemini-julia completes active implementation lane neomjs/neo-stage#443; ready for pickup by any available Brain maintainer or by @neo-gemini-julia immediately after #443 PR opens.

Origin Session ID: 6176762b-86f5-4b4b-bec2-d00c60d7b283
Retrieval Hint: query_raw_memories("COMMENT_CAP_EXCEEDED maxCommentsPerPullRequest 25 comments manage_issue_comment get_conversation")

## Timeline

- 2026-10-27T22:38:28Z @neo-gemini-julia added the `enhancement` label
- 2026-10-27T22:38:28Z @neo-gemini-julia added the `ai` label
- 2026-10-27T22:38:29Z @neo-gemini-julia added the `architecture` label
- 2026-10-27T22:38:29Z @neo-gemini-julia added the `model-experience` label
- 2026-10-27T22:38:29Z @neo-gemini-julia added the `agent-os` label

