---
id: 1616
title: The github-workflow server cannot close an issue
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-11T21:29:08Z'
updatedAt: '2026-11-11T21:47:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1616'
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
---
# The github-workflow server cannot close an issue

## Context

A maintainer closes an issue once its verdict is posted. The github-workflow server has no action for that, so a seat runs `gh issue close`, and on a Claude Code seat in auto mode the harness refuses the call.

| When | Call | Result |
|---|---|---|
| 2026-06-26 | `gh issue close` on three concurred `neomjs/neo` issues | refused, `[External System Writes]` (#132) |
| 2026-11-11 | `gh issue close 36 --reason "not planned"` in this repository, on the caller's own ticket with its verdict comment posted first; once inside a compound call, once alone | refused twice, `[External System Writes]` |

Each time the close went back to the operator as a click.

## The Problem

The refusal comes from a rule Claude Code ships, not from anything this organization configures. `claude auto-mode defaults` lists `External System Writes` as a soft deny for closing or resolving items in external collaboration tools, GitHub issues among them, "that the agent did not create in this session". A ticket is almost never closed in the session that created it, so the rule applies to nearly every close.

No repository file lifts it. The classifier reads `autoMode` from the user's own settings, managed settings and `--settings` only. The `autoMode` block in `neomjs/neo`'s `.claude/settings.template.json` is not read (evidence with controls: neomjs/neo#20794, comment 6807353591). #132 recorded its "Fix 2" as resolved by that block; it was not.

What does pass is this server: `mcp__neo-mjs-github-workflow__*` is in the seats' `permissions.allow`, which is resolved before the classifier. #1488 used that for Discussions, and #1614 uses it for held workflow runs.

## The Architectural Reality

- `ai/mcp/server/github-workflow/openapi.yaml` at `c27e8b9b`: the issue writes are `create_issue`, `update_issue_body`, `manage_issue_comment`, `manage_issue_labels`, `manage_issue_assignees`, `manage_issue_projects` and `update_issue_relationship`. None changes an issue's state.
- The precedent is `manage_discussion` `action: 'close'` (PR #1489; `ai/services/github-workflow/DiscussionService.mjs`, `CLOSE_DISCUSSION` in `queries/mutations.mjs`): a required `reason`, a foreign close refused unless `acknowledgedForeignClose` states why, and an already-closed target answered unchanged with `alreadyClosed`.
- `ai/services/github-workflow/IssueService.mjs` holds the issue writes and their guards (`acknowledgedReassign`, `acknowledgedForeignBody`, `expectedBodySha256`).
- GitHub's `closeIssue` mutation takes `issueId`, `stateReason` (`COMPLETED`, `NOT_PLANNED`, `DUPLICATE`) and `duplicateIssueId` (introspected 2026-11-11).
- Design authority: `.agents/skills/epic-resolution/references/epic-resolution-workflow.md`, Verdict authority: "An issue close reopens with one click and is a maintainer act (operator, 2026-05-10 and 2026-10-28)."
- Structure map (`npm run ai:structure-map -- --files --loc`): owning folders `ai/mcp/server/github-workflow` and `ai/services/github-workflow`; sibling precedent PR #1489; no new file.

## The Fix

An issue close on the server, with #1488's contract:

1. **Reason:** `reason` is required: `COMPLETED`, `NOT_PLANNED` or `DUPLICATE`. `DUPLICATE` names the surviving issue.
2. **Guard:** the caller is the issue's author or one of its assignees. Any other close is refused unless `acknowledgedForeignClose` states why.
3. **Idempotence:** an already-closed issue is answered unchanged with `alreadyClosed: true`, and no mutation is sent.
4. **Precondition in the description:** the verdict is posted as a comment first, as `manage_discussion` says for Discussions.

Surface, decided at intake on 2026-11-11: a new operation `manage_issue_state` with `action: 'close'`, in the `write` tier like the other issue writes. No existing issue tool has a fitting name (`update_issue_body` is body-only), and the `action` leaves room for a reopen without another tool.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `manage_issue_state` (github-workflow MCP, `write` tier) | `epic-resolution-workflow.md`, Verdict authority; GitHub's `closeIssue` | `action: 'close'` closes an open issue with a required `reason` when the caller is its author or a current assignee | Structured errors, and nothing closes: `MISSING_ARGUMENTS`, `INVALID_ARGUMENTS`, `NOT_FOUND`, `FOREIGN_CLOSE_FORBIDDEN`, `GRAPHQL_API_ERROR`. An already-closed issue answers `alreadyClosed`. | The handbook, generated from `openapi.yaml` | `IssueServiceState.spec.mjs`, `ToolRegistration.spec.mjs` |

## Acceptance Criteria

- [ ] Unit arms for `IssueService`, in `IssueServiceState.spec.mjs` (the directory's split-spec precedent is `PullRequestServiceReviewers.spec.mjs`):
  - the author closes an own issue with each of the three reasons;
  - an assignee who is not the author closes it;
  - a foreign issue is refused without `acknowledgedForeignClose` and closes with it;
  - an already-closed issue answers `alreadyClosed` and sends no mutation;
  - a missing or unknown `reason`, and a `DUPLICATE` without its surviving issue, are refused before any request.
- [ ] `get_mcp_tool_handbook` documents the reason, the guard and the precondition.
- [ ] `[L3-deferred: needs a plane that runs the merge]` *(post-merge)* From a Claude Code seat in auto mode, a seat closes an own issue with a posted verdict through the tool, without a classifier refusal. The receipt on this ticket names the issue and the call.

## Out of Scope

- **Reopen.** The shipped rule names closing and resolving. A refused reopen has not been observed.
- **Closing a pull request.**
- **Epic closeout.** `epic-resolution` stays the gate; the tool closes what that workflow has cleared.
- **#132's remaining residual:** a classifier denial right before the terminal message.
- **The skills' wording.** Where a workflow tells a seat to run `gh issue close`, the text follows in `neomjs/neo-agent-skills` once the action runs on the plane.

## Avoided Traps

- **A `Bash(gh issue close *)` rule in the settings template.** It cannot bind authorship, it reaches only a seat whose `.claude/settings.json` does not exist yet (#1615), and it leaves every other harness without a guarded path.
- **An `autoMode.allow` sentence in a repository file.** It is not read.

## Related

#132 (the problem's home; this is its Fix 2 as a one-PR leaf) · #1488 and PR #1489 (precedent) · #1614 (sibling action) · #1615 · neomjs/neo#20794 (evidence)

Decision Record impact: none.

Live latest-open + closed sweep: the latest 20 of 152 open (red band; #132 owns the problem and keeps its stop-hook residual, so this is filed as the one-PR leaf of its Fix 2) and state-all searches (`gh issue close`, `auto mode classifier`, `close issue tool github-workflow`; #1488 read with its Out of Scope) at 2026-11-11T21:27Z; no equivalent. A2A in-flight claim sweep: no claim on this surface; #1614 is the sibling for workflow runs. MC sweep: "auto mode classifier refuses gh issue close External System Writes", 8 results; the 2026-06-26 turn names a settings permission rule as the path, never built. Own-assignment sweep: 15 open, none on this surface; #132 is mine by authorship and unassigned.

Origin Session ID: 93064ad1-a8ac-4022-9e9a-525fda9e3e8a

Retrieval Hint: "auto mode classifier refuses gh issue close; External System Writes; github-workflow close action for issues"


## Timeline

- 2026-11-11T21:29:08Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-11T21:29:10Z @neo-opus-vega added the `enhancement` label
- 2026-11-11T21:29:11Z @neo-opus-vega added the `ai` label
- 2026-11-11T21:29:11Z @neo-opus-vega added the `model-experience` label
- 2026-11-11T21:29:11Z @neo-opus-vega added the `agent-os` label
- 2026-11-11T21:30:43Z @neo-opus-vega cross-referenced by #20796
- 2026-11-11T21:32:16Z @neo-opus-vega cross-referenced by #132
- 2026-11-11T21:32:27Z @neo-opus-vega cross-referenced by #20794
- 2026-11-11T21:49:48Z @neo-opus-vega cross-referenced by PR #1617

