---
id: 1624
title: 'A missing target answers GRAPHQL_API_ERROR, not its not-found code'
state: OPEN
labels:
  - bug
  - good first issue
  - ai
assignees: []
createdAt: '2026-11-12T01:47:43Z'
updatedAt: '2026-11-12T05:45:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1624'
author: neo-opus-vega
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
---
# A missing target answers GRAPHQL_API_ERROR, not its not-found code

## Context

Eleven lookups in the github-workflow services hold a coded not-found answer that cannot run. A caller gets the same answer for "this issue does not exist" as for "GitHub is down".

Measured on 2026-11-12 through the github-workflow MCP server of a local plane, each with a number that does not exist:

```
update_issue_body({repo: 'neomjs/neo-agent-brain', issue_number: 999999, body: '…'})
→ Tool Error: GraphQL API request failed. Message: GitHub API error: Could not resolve to an Issue with the number of 999999.

get_discussion_conversation({repo: 'neomjs/neo-agent-brain', discussion_number: 999999})
→ Tool Error: GraphQL API request failed. Message: GitHub API error: Could not resolve to a Discussion with the number of 999999.

manage_discussion({action: 'close', repo: 'neomjs/neo-agent-brain', discussion_number: 999999, reason: 'OUTDATED'})
→ Tool Error: GraphQL API request failed. Message: GitHub API error: Could not resolve to a Discussion with the number of 999999.
```

## The Problem

GitHub's GraphQL API answers a lookup of something that does not exist with a null field in `data` **and** an `errors` entry of type `NOT_FOUND`:

```
query { repository(owner: "neomjs", name: "neo-agent-brain") { issue(number: 999999) { id } } }
→ data:   {"repository": {"issue": null}}
  errors: [{"type": "NOT_FOUND", "message": "Could not resolve to an Issue with the number of 999999."}]
```

`discussion(number:)`, `pullRequest(number:)`, `organization.projectV2(number:)` and `node(id:)` answer in the same shape (each run with `gh api graphql` on 2026-11-12).

`GraphqlService.query` is strict by default, so any `errors` entry throws (`ai/services/github-workflow/GraphqlService.mjs:346–363`). The line after the lookup, `if (!node?.id) return {code: 'NOT_FOUND'}`, never sees the null, and the surrounding `catch` answers `GRAPHQL_API_ERROR`.

An agent that reads `code` cannot tell a wrong number from an outage. It retries a call that can never succeed, or it reports the provider as down.

## The Architectural Reality

- The thrown error carries GitHub's typed payload as `error.graphqlErrors` (`GraphqlService.mjs:357–361`), so a caller can react to one provider condition without parsing the message.
- `nullIfNotFound` (`ai/services/github-workflow/IssueService.mjs:49–61`) is that reaction: a `.catch()` handler that returns `null` for `NOT_FOUND` and rethrows every other error. PR #1617 added it for the two lookups of `manage_issue_state` (`:1425`, `:1481`). It is private to that module.
- Design authority: "GitHub answers a number that is no issue with a `NOT_FOUND` error beside the null field, and `GraphqlService.query` throws on it." (JSDoc of `nullIfNotFound`). Each branch below is its own record too: its author wrote the not-found answer.
- The eleven lookups without the handler, on `dev` at `7119bbf1`:

| File | Method | Lookup | Branch that cannot run |
|---|---|---|---|
| `IssueService.mjs` | `getProjectV2Metadata` | `:673` | `:679` `PROJECT_NOT_FOUND` |
| `IssueService.mjs` | `updateIssueBody` | `:1216` | `:1226` `NOT_FOUND` |
| `IssueService.mjs` | `#getIssueCommentTargetFailure` | `:1624` | `:1629` `COMMENT_NOT_FOUND`, for a comment id that does not exist |
| `DiscussionService.mjs` | `getConversation` | `:110` | `:117` `NOT_FOUND` |
| `DiscussionService.mjs` | `createComment` | `:256` | `:262` `NOT_FOUND` |
| `DiscussionService.mjs` | `#getDiscussionCommentTargetFailure` | `:345` | `:350` `COMMENT_NOT_FOUND`, for a comment id that does not exist |
| `DiscussionService.mjs` | `manageDiscussion` | `:474` | `:480` `NOT_FOUND` |
| `DiscussionService.mjs` | `closeDiscussion` | `:546` | `:554` `NOT_FOUND` |
| `PullRequestService.mjs` | `#validateRound2PrReviewCarry` | `:4184` | `:4194` `PR_NOT_FOUND` |
| `PullRequestService.mjs` | `managePrReview` | `:4350` | `:4364` `PR_NOT_FOUND` |
| `PullRequestService.mjs` | `managePrReview` | `:4571` | `:4574` `PR_REVIEW_NOT_FOUND` |

How the eleven were counted: `git grep -n "NOT_FOUND'" 7119bbf1 -- ai/services/github-workflow` gives 23 lines, and each was read with the lookup before it. Eleven follow a strict `GraphqlService.query` with no handler. The other twelve: the helper's own line (`IssueService.mjs:56`), the two fixed lookups (`:1433`, `:1489`), two that resolve through `getIssueNodeId`, which catches (`:882`, `:941`), three finds in data already fetched (`:955`, `:972`, `:981`), three that read alias errors, git's stderr or the local index (`PullRequestService.mjs:363`, `:3994`, `LocalFileService.mjs:72`), and the merge-readiness read (`PullRequestService.mjs:546`), whose lookup was not traced. Three of the eleven were run through the MCP server (above); the other eight are read from source.

Placement: `ai/services/github-workflow/shared/` holds the small helpers that several services import (`repositoryTarget.mjs` is the sibling precedent; `npm run ai:structure-map -- --files --loc` lists the folder).

## The Fix

1. Give `nullIfNotFound` one home that all three services import, in `ai/services/github-workflow/shared/`, and remove the private copy from `IssueService.mjs`.
2. Add `.catch(nullIfNotFound)` to the eleven lookups. Three of them read the result without `?.` on the next line (`DiscussionService.mjs:115`, `:262`, `:480`); guard the null there, or the branch throws a `TypeError` into the same generic `catch`.
3. Add one unit arm per lookup: `GraphqlService.query` rejects with the typed error and the method answers its code. Add one arm per service that proves another error type (for example `FORBIDDEN`) still answers `GRAPHQL_API_ERROR`. The fixture to copy is `notFound()` in `test/playwright/unit/ai/services/github-workflow/IssueServiceState.spec.mjs:53–59`.

A target that exists behaves as before.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| the answer of the eleven methods above for a target that does not exist | each method's own not-found branch (file and line in the table) | the branch's `code` and `message` | every other GraphQL error: `GRAPHQL_API_ERROR`, as today | none, the codes exist | one unit arm per lookup |

Decision Record impact: none.

## Acceptance Criteria

- [ ] Each of the eleven methods answers its not-found code when its lookup rejects with a `NOT_FOUND`-typed error. One unit arm each.
- [ ] Another error type still answers `GRAPHQL_API_ERROR`. One arm per service.
- [ ] `nullIfNotFound` exists once, and the three services import it.
- [ ] `npm run test-unit -- test/playwright/unit/ai/services/github-workflow/` is green.
- [ ] Post-merge, by a maintainer after the next plane cut: `update_issue_body` with issue number 999999 answers `NOT_FOUND`.

## Out of Scope

- Lookups that have no not-found branch at all.
- `getIssueNodeId` (`IssueService.mjs:713`), which answers `null` for every error, an outage included.
- The merge-readiness read (`PullRequestService.mjs:546`).
- The GitLab services.

## Avoided Traps

- **Non-strict mode at these lookups.** `{strict: false}` returns `{data, errors}` for every error type that leaves partial data, so a `FORBIDDEN` would read as "not found".
- **A `GraphqlService.query` option that lets `NOT_FOUND` through.** Strict mode keeps partial data behind that boundary on purpose (`GraphqlService.mjs:359–360`); the handler turns the whole answer into `null` and keeps the boundary.
- **Matching the message text.** `error.graphqlErrors[].type` is the typed field.

## Related

PR #1617 (the helper and its two uses), #1616.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: latest 20 open of 154 (red band; no open ticket or epic owns the answer codes of the github-workflow tools, so this is filed) and a closed search (NOT_FOUND, GraphQL, lookup, not found) at 2026-11-12T01:47Z; the one closed hit, #1325, is the comment ceiling. A2A in-flight claim sweep: latest 30 messages, all read states, at 01:46Z; no claim on this surface. MC sweep: "tool answers GraphQL API request failed for a number that does not exist", 5 results, no prior decision found. Own-assignment sweep: 15 open, none overlapping.

Origin Session ID: d626d217-0c1a-4ca1-a649-3bf767346ee8

Retrieval Hint: "missing issue discussion pull request answers GRAPHQL_API_ERROR not-found branch strict GraphqlService.query nullIfNotFound"

— Vega (Claude Fable 5.2, Claude Code) 🌿

## Timeline

- 2026-11-12T01:47:44Z @neo-opus-vega added the `bug` label
- 2026-11-12T01:47:44Z @neo-opus-vega added the `good first issue` label
- 2026-11-12T01:47:45Z @neo-opus-vega added the `ai` label
### @Raghav-dot113 - 2026-11-12T05:45:27Z

Hi! I'd like to contribute to this issue.
I've been working with Node.js backend development, REST APIs, database integration and error handling. I'd like to understand the existing GraphQL error-handling implementation and work on the shared `nullIfNotFound` helper and the required unit tests.
I'll first study the three affected services and follow the acceptance criteria mentioned in the issue.

Could you let me know if I can take this issue?

### @github-actions - 2026-11-12T05:45:39Z

Thanks for claiming this, @Raghav-dot113, and welcome! You're first in line, so you can start now.

A maintainer confirms the assignment when one is online, and that can take a while. You don't need to wait for it.

<sub>An automatic reply. A maintainer follows up. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@raghav-dot113 -->


