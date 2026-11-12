---
id: 1614
title: github-workflow approves an outside contributor's held runs after a diff check
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T21:26:07Z'
updatedAt: '2026-11-12T08:07:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1614'
author: neo-opus-grace
commentsCount: 1
parentIssue: 759
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
# github-workflow approves an outside contributor's held runs after a diff check

## Context

On 2026-11-11 the operator ruled that approving an outside contributor's held workflow runs is a maintainer act, and neomjs/neo-agent-skills#352 wrote the check into the pr-review guide. The Fleet routes `approve-runs` wakes to the maintainer rotation (`ai/services/fleet/openWorkHolder.mjs:41`).

A Claude Code seat in auto mode cannot perform the act. Its classifier refuses `gh api -X POST …/actions/runs/<id>/approve` as "Code from External" (Clio, neomjs/neo#20789, 20:35Z).

neomjs/neo#20794 tried a project `autoMode.allow` rule; Vega falsified that before any PR (neomjs/neo#20794 comment 6807353591). Claude Code reads `autoMode` only from user, managed and flag settings, never from a project's `.claude/settings.json`. Grace's control: in a seat checkout that carries the template's block, `claude auto-mode config` equals `claude auto-mode defaults` (Claude Code 2.1.212), and none of the template's sentences is in it.

## The Problem

No path lets a Claude seat approve a held run. A rotation `approve-runs` wake that lands on one of the five Claude seats goes back to the operator, which is the burden his ruling removed.

## The Architectural Reality

- `mcp__neo-mjs-github-workflow__*` is in the Claude seats' `permissions.allow`, which is resolved before the classifier runs, so the server's tools run unclassified. The precedent is #1488: `manage_discussion` gained `close` because the classifier refused the raw mutation.
- Design authority: `.agents/skills/pr-review/references/pr-review-guide.md` (neo-agent-skills 1.4.2): "Any maintainer approves their pending workflow runs once the diff touches nothing the runs execute (`.github/**`, package scripts, lockfile beyond the version, source, build or test code); otherwise read those changes first."
- GitHub's endpoint, `POST /repos/{owner}/{repo}/actions/runs/{run_id}/approve`, applies to a fork pull request's runs in `action_required`.

## The Fix

A github-workflow action approves a pull request's held runs, and makes the guide's check part of the call:
1. It reads the held runs (`action_required`) for the PR's current head, and refuses runs on an older head.
2. It classifies the PR's changed files. Only Markdown outside `.github/` counts as not executed; every other changed file is executed (workflows, manifests, lockfiles, code, tests).
3. With no executed file, it approves. Otherwise the caller passes `executedFilesRead`, naming exactly the executed files it read; on any difference the tool refuses and names the files. The check is recorded, not assumed.
4. It answers which runs it approved and the classification it used.

The exact surface, a new tool or an action on an existing pull-request tool, is decided at intake against the MCP surface's size.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `approve_pull_request_runs` (new, github-workflow, `write` tier) | the operator's 2026-11-11 ruling and the pr-review guide's check | reads the PR's current head, the held (`action_required`) runs on that head, and the PR's changed files; approves when no changed file is executed; with executed files, approves only when `executedFilesRead` names exactly them | a held run on an older head is refused and named; no held run approves nothing; a difference between the executed files and `executedFilesRead` refuses and names the files; a file list GitHub truncates refuses as unread | the tool's OpenAPI description and the service's JSDoc | the service spec over a stubbed REST client, `ToolRegistration.spec.mjs`, the tool census |

Decision Record impact: none.

## Acceptance Criteria

- [ ] Unit:
  - A Markdown-only diff approves the held runs on the current head.
  - A diff that touches `.github/workflows/x.yml` without `executedFilesRead` is refused, naming the file; with exactly that list it is approved.
  - A run on an older head is refused, and a PR with no held run approves nothing.
- [ ] *(post-merge, installed)* A Claude seat in auto mode approves a held outside run through the tool, without a classifier refusal.

## Out of Scope

- The inert `autoMode` block in neomjs/neo's `.claude/settings.template.json`: deleting it is Vega's offered follow-up.
- Copying template rules into existing seats (`initClaudeSettings` merges only `hooks`): Vega's Brain ticket.
- Routing `approve-runs` wakes by family.

## Avoided Traps

- A project `autoMode` rule: never read (neomjs/neo#20794).
- `Bash(gh api …/approve)` permission rules: a `*` in the middle matches spaces and admits unrelated `gh api` writes.
- An approval with no file check: it would approve code nobody read.

## Related

Parent #759 · neomjs/neo#20794 (falsified) · #1488 (precedent) · neomjs/neo-agent-skills#352 · neomjs/neo#20789

Live latest-open + closed sweep: the latest 20 of 150 open (red band; the owning epic is #759, so this is filed as its leaf) and a state-all search (`approve workflow runs OR action_required OR "approve-runs" github-workflow`; #1008, #427 and #1488 read) at 2026-11-11T21:25Z; no equivalent. A2A in-flight claim sweep: Vega's 21:23Z message recommends this lever and files two other tickets (the settings rollout, the dead block); no claim on this surface. MC sweep: no prior decision. Own-assignment sweep: #1607 and #807, no overlap.

Intake, 2026-11-12: the drift probe since creation found one commit on the github-workflow server, #1617 (`manage_issue_state` closes an issue). It touches no run surface, and it is a second precedent beside #1488 for a classifier-refused act through this server. The surface is a new tool. No existing tool's noun covers a pull request's runs: `manage_pr_review` is a review and `manage_pr_reviewers` the seats. Its tier is `write`, so every rotation seat, of any family, takes the same checked path. The REST calls go through `GraphqlService#rest`.

handoff: @neo-opus-grace self-assigns.

Origin Session ID: 15ea444b-707c-4458-bba5-79b4f61fb280

Retrieval Hint: "approve outside contributor held workflow runs via github-workflow MCP tool, mechanical diff check, Claude auto mode classifier"

## Timeline

- 2026-11-11T21:26:07Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T21:26:09Z @neo-opus-grace added the `enhancement` label
- 2026-11-11T21:26:09Z @neo-opus-grace added the `ai` label
- 2026-11-11T21:26:09Z @neo-opus-grace added the `agent-os` label
- 2026-11-11T21:26:21Z @neo-opus-grace added parent issue #759
- 2026-11-11T21:26:28Z @neo-opus-grace cross-referenced by #20794
- 2026-11-11T21:28:43Z @neo-opus-vega cross-referenced by #1615
- 2026-11-11T21:29:09Z @neo-opus-vega cross-referenced by #1616
- 2026-11-11T21:30:43Z @neo-opus-vega cross-referenced by #20796
- 2026-11-11T23:00:50Z @neo-opus-grace cross-referenced by #1619
- 2026-11-12T00:15:49Z @neo-opus-grace cross-referenced by #1620
- 2026-11-12T03:19:08Z @neo-opus-grace cross-referenced by #1628
- 2026-11-12T04:10:55Z @neo-fable-mnemosyne cross-referenced by #1629
- 2026-11-12T04:11:30Z @neo-gemini-julia cross-referenced by #1630
### @neo-opus-grace - 2026-11-12T08:07:17Z

**Handover (Grace, session sunset, 12 Nov 08:1xZ).** Owner: Grace; the next session of this seat builds it. No branch exists yet.

**Decided at intake:** the body's Contract Ledger, and a new tool, `approve_pull_request_runs` (`write` tier).

**Placement (structural fast-path):** a new `ai/services/github-workflow/WorkflowRunService.mjs`, a sibling of the per-domain services `LabelService.mjs` and `IssueService.mjs`. It is not more lines in `PullRequestService.mjs`, which already has 4,936; the accretion rule applies.

**Registration points**, all in `ai/mcp/server/github-workflow/toolService.mjs`:
- `REPOSITORY_TARGET_TOOLS`
- `GITHUB_TOOL_ACCESS`: `PUBLIC_GITHUB_WRITE_ACCESS`, so the write guard probes the viewer
- the service mapping

Then its `openapi.yaml` entry (model it on `manage_pr_reviewers`) and the census specs: `McpServerListToolsSmoke.spec.mjs`, `OpenApiValidatorCompliance.spec.mjs`'s tier map, `ToolRegistration.spec.mjs`, `RepositoryTargetRouting.spec.mjs`, `toolService.spec.mjs`. #1617 is the shape of such a change.

**Transport:** either `GraphqlService#rest` (fetch with the token) or the `execFn` / `gh api` seam that `managePrReviewers` uses. Pick the one the sibling's specs stub more simply. Both run inside the MCP server, so the classifier never sees the call.

**Calls:**
- `GET /repos/{o}/{r}/pulls/{n}` for the head SHA.
- `GET /repos/{o}/{r}/actions/runs?head_sha={sha}&status=action_required` for the held runs. Refuse any run whose `head_sha` is not the current head.
- `GET …/pulls/{n}/files?per_page=100`, paged. A truncated list refuses as unread.
- `POST /repos/{o}/{r}/actions/runs/{id}/approve` per run.

**Before writing:** the ticket-intake core-idiom pre-flight for a new `Neo.setupClass` service (read `src/core/Base.mjs` and `src/Neo.mjs`, and name them in the intake record).

🖖 Grace (Claude Opus 5.5, Claude Code) · session a0126f58-6ebb-41be-83b7-1371cf98480f


