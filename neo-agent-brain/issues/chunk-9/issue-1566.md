---
id: 1566
title: 'GitHub and GitLab workflow servers: each tool line says what the tool does and when to use it'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-10T02:33:35Z'
updatedAt: '2026-11-10T22:07:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1566'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 1562
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1563 ToolService lists one line per tool: summary is its only source, title and x-neo-tool-summary go'
blocking: []
closedAt: '2026-11-10T22:07:46Z'
---
# GitHub and GitLab workflow servers: each tool line says what the tool does and when to use it

## Context

A server leaf of #1562 (graduated from neomjs/neo#20680, OQ-7). Blocked by the `ToolService` leaf, which makes `summary` the only source of a listed line. The two workflow servers share their vocabulary and their users, so their lines are written together and stay parallel.

## The Problem

Seven of the 39 summaries repeat the operation id (6 of 25 on GitHub, 1 of 14 on GitLab); the rest say what a tool does and not when to take it over its neighbour: `manage_issue_comment` or `update_issue_body` for a correction, `get_conversation` or `get_local_issue_by_id` for a read, `manage_pr_review` or a plain comment. The GitHub server records no calls per tool, so use cannot show which lines fail.

## The Architectural Reality

- `ai/mcp/server/github-workflow/openapi.yaml` (25 operations, 7 curated lines today) and `ai/mcp/server/gitlab-workflow/openapi.yaml` (14 operations, none).
- The guidance tiers of neomjs/neo#16602 put an imperative into the listed line of the comment tool; that text moves into `summary` with the `ToolService` leaf and keeps its meaning here.

## The Fix

Rewrite each `summary` of both servers in the form `<what>; use when <trigger>`, at most 120 bytes, written by a seat that files tickets and reviews through these tools. A GitLab operation that mirrors a GitHub one gets the same line with its own nouns.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `summary` of each operation on both servers | #1562, "The contract of the line" | `<what>; use when <trigger>`, at most 120 bytes; mirrored operations read alike | none | the OpenAPI files | the byte test of the `ToolService` leaf; the stranger read below |

## Acceptance Criteria

- [ ] All 39 summaries have the form; none repeats the operation id; the byte test is green.
- [ ] The line of `manage_issue_comment` still says that a correction belongs in the body, in at most 120 bytes.
- [ ] A second seat reads ten lines chosen by the reviewer, without the descriptions, and says for each when it would call the tool; a line it cannot place is rewritten.
- [ ] The compatibility gate: this leaf changes no `inputSchema` and no `outputSchema`; the listed `outputSchema` of every tool is byte-identical before and after.

## Out of Scope

Per-tool call recording on the GitHub server. Descriptions and the handbook. The catalog.

## Related

#1562 (parent), the `ToolService` leaf (blocks this one), neomjs/neo#16602.

unowned-rationale: ready for a seat that works through the workflow tools daily. No `good first issue`: the wording carries the team's review and ticket rules.

Live latest-open + closed sweep and A2A, MC and own-assignment sweeps: as on #1562 at 2026-11-10T02:27Z; this leaf is filed with it and has no equivalent.

Origin Session ID: fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

Retrieval Hint: "workflow server tool trigger lines summary what and when"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

## Timeline

- 2026-11-10T02:33:37Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-10T02:33:37Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-10T02:33:37Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-10T02:33:37Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-10T02:34:39Z @neo-fable-mnemosyne added parent issue #1562
- 2026-11-10T02:35:05Z @neo-fable-mnemosyne marked this issue as being blocked by #1563
- 2026-11-10T21:05:07Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-10T21:20:26Z @neo-gemini-maryam cross-referenced by PR #1579
- 2026-11-10T21:47:19Z @neo-gemini-maryam referenced in commit `a7ef710` - "fix(mcp): clarify scaffolded gitlab local-issue trigger summary (#1566)"
- 2026-11-10T22:07:45Z @tobiu referenced in commit `cbda126` - "feat(mcp): add trigger summaries to github and gitlab workflow tools (#1566) (#1579)

* feat(mcp): add trigger summaries to github and gitlab workflow tools (#1566)

* fix(mcp): clarify scaffolded gitlab local-issue trigger summary (#1566)"
- 2026-11-10T22:07:47Z @tobiu closed this issue

