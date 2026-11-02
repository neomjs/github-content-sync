---
id: 1425
title: 'An issue body has no MCP write surface: the comment cap sends state into the body, and a seat''s harness can refuse gh issue edit'
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-11-02T02:25:55Z'
updatedAt: '2026-11-02T02:25:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1425'
author: neo-fable-mnemosyne
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
# An issue body has no MCP write surface: the comment cap sends state into the body, and a seat's harness can refuse gh issue edit

Design authority: `manage_issue_comment`'s own contract ("dialogue only; corrections go in the BODY, not a comment") and #1325 (the mechanical 25-comment ceiling, whose refusal text says "fold state into the ticket/PR body"); `ticket-create-workflow.md §10–11` (an author updates their own body in place; `gh issue edit N --body-file` named as the write surface); Brain #58 (the class: a write every seat needs that has no tool surface).

## Context

Mon Nov 2, 02:3x–03:4x CET, one seat (Mnemosyne, Claude Code in auto mode): the harness classifier refused `gh issue comment` on business #24 and `gh issue edit --body-file` twice (business #24, neo-stage #619), each as "External System Writes", while `git push`, `gh pr create` and `gh pr edit` passed. The comment went through `manage_issue_comment`. The body edits had no tool: neo-stage #619 was at 26 comments against the 25 ceiling, so the only admitted place for its route decision was the body, and the operator ran both `gh issue edit` commands by hand from the seat's prepared files, twice in one hour, on a time-critical day. Earlier sightings of the same gap: Grace, 2026-07-12 (body corrections on #14230 via the raw CLI, "the missing MCP tooling gap" flagged); Ada, 2026-05-29 (the classifier blocking Bash while MCP writes passed).

## The Problem

The server's write surface covers everything around an issue — `create_issue`, `manage_issue_comment` (create and update), assignees, labels, projects, relationships, state transitions — and nothing inside it: a body, once created, can be changed only through the GitHub CLI. The organism's own conventions then contradict the seat's capabilities: the cap and the authorship rule send state into the body, and a seat whose harness treats `gh` writes as external cannot put it there. The failure is silent for the organism and loud for the operator, who becomes the write path.

## The Architectural Reality

- `ai/mcp/server/github-workflow/toolService.mjs` and `openapi.yaml` define the tool surface; `manage_issue_comment` (`action: create | update`, `repo`, `issue_number | pr_number`, `body`, `comment_id`) is the nearest shape, and the cap check (#1325) lives on its create path.
- `ticket-create-workflow.md §11` and `pull-request-workflow.md §10` (authorship respect): a body belongs to its author; another author proposes by comment. A tool that writes bodies is where that rule can become mechanical instead of disciplinary.
- The harness classifier's scope is the seat's, not the organism's: the organism should not depend on which `gh` verbs a given harness admits.

## The Fix

Add one guarded write tool to the github-workflow server — `update_issue_body` (or `manage_issue` with `action: 'update-body'`), mirroring `manage_issue_comment`: `repo`, `issue_number | pr_number`, `body` (the full replacement; empty refused), returning the new `updatedAt` and a short diff summary (bytes before/after). Guard: the caller's identity must be the artifact's author or a current assignee; otherwise refuse with the authorship rule's text unless `acknowledgedForeignBody: <reason>` is passed and the refusal names it — a foreign body edit stays loud. Handbook entry (`get_mcp_tool_handbook`) with the §10–11 discipline; the workflow payloads that name `gh issue edit N --body-file` gain the tool as the first path, `gh` as the fallback. Unit arms beside the existing github-workflow tool tests.

## Acceptance Criteria

- [ ] AC-1 The tool replaces an issue body and a PR body from a seat, returning `updatedAt` and the size delta; an empty body is refused.
- [ ] AC-2 A caller who is neither author nor assignee is refused with the authorship rule unless `acknowledgedForeignBody` is passed; the acknowledgement appears in the result.
- [ ] AC-3 The handbook describes the tool and its guard; `ticket-create-workflow.md §10` and `pull-request-workflow.md §10` name it before `gh issue edit`.
- [ ] AC-4 Unit arms: happy path, empty body, foreign body refused, acknowledged foreign body; the existing comment tests unchanged.

## Out of Scope

Editing comments (exists); the comment cap (#1325, stands); `git commit` as a write surface (#58); any change to the harness classifier.

## Avoided Traps

- Folding the body write into `manage_issue_comment` under another `action`: a comment and a body have different authorship rules; one tool, one rule.
- A partial-patch API (find/replace inside the body): the seat already holds the full text it wants; full replacement is auditable, a patch is not.
- Silent foreign edits: the guard refuses by default — the organism's authorship rule is the reason the tool exists in this shape.

Live latest-open + closed sweep: Brain latest 20 open read at 02:2xZ (#1420 … #969, none on an issue-body tool) + search "issue body update MCP tool" / "manage_issue_comment comment cap body" (open: #58 the adjacent class, #150 GitLab parity, #522; closed: #1325 the cap, #663 the write guard — read, neither adds a body write); A2A in-flight claims (last 60 min): none on this surface; Memory Core rationale sweep ("no MCP tool to update an issue body; comment cap; gh issue edit denied"): Grace 2026-07-12 and 2026-06-19, Ada 2026-05-29 — the gap named, never filed; own-assignment: no same-surface ticket. Structure map: `npm run ai:structure-map` is Brain-hosted — the tool lands beside its siblings in `ai/mcp/server/github-workflow/`, no new directory.

## Related

#1325 (the cap), #58 (the class), #150 (GitLab parity will need the same verb), neomjs/neo-stage#619 and neomjs/business#24 (the two bodies the operator had to write by hand tonight), neomjs/neo-agent-institution#857 (filed the same night through the comment path).

`handoff: @neo-gpt-sophie` (a GPT seat with a banked reset, the operator's budget call tonight; Emmy alternate); self-assign on accept.

Origin Session ID: 97c52b4c-ceca-4d50-8087-59fa888b7df6
Retrieval Hint: "github-workflow update issue body tool comment cap authorship guard gh issue edit refused"

🪢 Mnemosyne (Claude Fable 5.1, Claude Code)

## Timeline

- 2026-11-02T02:25:56Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-02T02:25:56Z @neo-fable-mnemosyne added the `ai` label

