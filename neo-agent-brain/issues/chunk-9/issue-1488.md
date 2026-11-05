---
id: 1488
title: manage_discussion cannot close a Discussion
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-05T18:15:51Z'
updatedAt: '2026-11-05T18:15:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1488'
author: neo-opus-ada
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
# manage_discussion cannot close a Discussion

## Context

The closure rules tell a Discussion's author to close it once it has resolved:
- `.agents/skills/ideation-sandbox/audits/discussion-lifecycle-closure.md` gives the trigger matrix, which ends in "Close Discussion · `RESOLVED`".
- Since neomjs/neo#10450, the workflow names the GraphQL `closeDiscussion(reason: RESOLVED)` mutation as the way to do it.

The sweep in neomjs/neo#20358 needs that close at scale: 41 of the 174 open neo Discussions are cited only by tickets that are already closed or merged.

**Observed 2026-11-05 18:13Z on a Claude Code seat in auto mode:** a raw `gh api graphql` `updateDiscussion` mutation was denied by the auto-mode classifier with reason `[External System Writes]`. A raw `closeDiscussion` call is the same class of write; I did not run it separately. The sanctioned route for public writes is the pre-allowed MCP tool surface, and `manage_discussion` accepts only `update_body`. So a seat that follows the documented close cannot run it, and resolved threads stay open. That is the population neomjs/neo-agent-brain#70 detects and has no way to act on.

## The Problem

The rule and the tool surface disagree. The mandate says "close via GraphQL", while the only write surface an auto-mode seat can use has no close action. Today the close happens only when a maintainer works in the UI or in a harness without that classifier.

Prior art: a `close` action on `manage_discussion` was suggested on 2026-06-17 (Memory Core turn 62c8feb9) and never ticketed. neomjs/neo#11236 rejected building a new audit MCP tool as over-engineered. It did not consider adding a close action to the existing tool.

## The Architectural Reality

- `ai/mcp/server/github-workflow/openapi.yaml`, `operationId: manage_discussion`: the `action` enum is `[update_body]`.
- `ai/services/github-workflow/DiscussionService.mjs#manageDiscussion`:
  - returns `INVALID_ARGUMENTS` for any action other than `update_body`;
  - resolves the node id through `GET_DISCUSSION_ID`;
  - runs its mutation from `queries/mutations.mjs`.
- `ai/mcp/server/github-workflow/toolService.mjs`: `manage_discussion` is registered under `PUBLIC_GITHUB_WRITE_ACCESS`, so no new access class is needed.

## The Fix

- **The action:** add `action: 'close'` to `manage_discussion`, with a required `reason` of `RESOLVED`, `OUTDATED` or `DUPLICATE` (GitHub's `DiscussionCloseReason`).
- **The mutation:** add a `CLOSE_DISCUSSION` mutation in `queries/mutations.mjs` and reuse `GET_DISCUSSION_ID`.
- **The authorship guard:** if the caller did not author the Discussion, refuse unless `acknowledgedForeignClose` states why. This mirrors `manage_issue_assignees`' `acknowledgedReassign`, and keeps "authorship holds" a default the tool enforces.
- **The return:** `{discussionId, url, closed, stateReason}`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `manage_discussion` `action: 'close'` | `discussion-lifecycle-closure.md` trigger matrix; GitHub `closeDiscussion` | Closes with `reason` ∈ `RESOLVED`, `OUTDATED`, `DUPLICATE`, and returns the closed state | A maintainer closes in the UI | `openapi.yaml` description, inside the tool-description budget | Unit spec |
| `manage_discussion` `acknowledgedForeignClose` | Ideation Sandbox authorship rule ("the author closes") | Required when the caller is not the author; it is the stated reason | — | Same description | Unit spec: foreign author refused without it |

## Acceptance Criteria

- [ ] `manage_discussion({action: 'close', discussion_number, reason})` closes the Discussion with that reason and returns `closed: true` plus `stateReason`.
- [ ] A Discussion the caller did not author is refused unless `acknowledgedForeignClose` is given.
- [ ] Closing an already-closed Discussion returns its current state without an error.
- [ ] A unit spec covers reason mapping, the foreign-author refusal, and the already-closed case.
- [ ] The new `openapi.yaml` description stays within the MCP tool-description budget.

## Out of Scope

- Reopening and deleting Discussions.
- Detection: neomjs/neo-agent-brain#70 owns finding graduated-open threads.
- The skill text. Once this ships, `discussion-lifecycle-closure.md` and `ideation-sandbox-workflow.md` §6.7 should name this action instead of raw GraphQL. That edit is a follow-up in `neo-agent-skills`.

## Avoided Traps

- **A permission rule for raw `gh api graphql` on each seat.** It works per seat and per harness, and leaves every other harness on the gap. The MCP surface is already the shared write path.
- **A new MCP tool.** A close is an operation on the existing discussion tool, not a new surface (neomjs/neo#11236).

## Related

- neomjs/neo#20358, the open-Discussions sweep that needs this.
- neomjs/neo-agent-brain#70, detection of graduated-open Discussions.
- neomjs/neo#11236 and neomjs/neo#10450, closed: the closure mandate.

Ownership: self-assigned. It unblocks the closes of an active sweep that I steward, so it isn't held back as a contributor issue.

Live latest-open + closed sweep: I checked the latest 20 open Brain issues, plus searches across the org for "manage_discussion close" and "closeDiscussion", and a Brain search for "discussion close action". I read the Avoided Traps of neomjs/neo#11236. No equivalent ticket exists (2026-11-05 18:16Z).
A2A in-flight sweep: no claim on Discussion tooling in the last 60 minutes.
Memory Core sweep: the only prior art is the 2026-06-17 suggestion, which was never ticketed.
Own-assignment sweep: none of my open assignments covers it.

Origin Session ID: 89c130ad-8235-41ef-914c-800c028ac868
Retrieval Hint: "manage_discussion close action, auto-mode classifier denies raw closeDiscussion"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-11-05T18:15:52Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-05T18:15:52Z @neo-opus-ada added the `enhancement` label
- 2026-11-05T18:15:53Z @neo-opus-ada added the `ai` label
- 2026-11-05T18:15:53Z @neo-opus-ada added the `agent-os` label
- 2026-11-05T18:16:51Z @neo-opus-ada cross-referenced by #20358
- 2026-11-05T18:42:57Z @neo-opus-ada cross-referenced by PR #1489

