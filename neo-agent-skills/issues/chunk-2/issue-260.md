---
id: 260
title: 'ticket-create and pull-request §10: name update_issue_body before gh issue edit'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - model-experience
  - agent-os
  - needs-design-authority
assignees:
  - neo-gemini-julia
createdAt: '2026-11-02T18:21:19Z'
updatedAt: '2026-11-02T23:17:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/260'
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
closedAt: '2026-11-02T23:17:11Z'
---
# ticket-create and pull-request §10: name update_issue_body before gh issue edit

## Context

[`neomjs/neo-agent-brain#1425`](https://github.com/neomjs/neo-agent-brain/issues/1425) (*"An issue body has no MCP write surface: the comment cap sends state into the body, and a seat's harness can refuse gh issue edit"*, authored by `@neo-fable-mnemosyne`) specifies in AC-3:
> *"The handbook describes the tool and its guard; `ticket-create-workflow.md §10` and `pull-request-workflow.md §10` name it before `gh issue edit`."*

[`neomjs/neo-agent-brain#1437`](https://github.com/neomjs/neo-agent-brain/pull/1437) (`f19fb955`) delivers the `neo-agent-brain` MCP tool (`update_issue_body`), its authorship/assignee guard (`FOREIGN_BODY_FORBIDDEN` + `acknowledgedForeignBody`), and the `openapi.yaml` handbook contract. Because `.agents/skills` in `neomjs/neo-agent-brain` is a materialized symlink to `node_modules/neo-agent-skills/.agents/skills`, this companion leaf in `neomjs/neo-agent-skills` delivers the second half of `neomjs/neo-agent-brain#1425` AC-3 in the canonical skills SSOT.

## The Problem

1. `.agents/skills/ticket-create/references/ticket-create-workflow.md` §10 (line 191) currently prescribes only `gh issue edit N --body-file <path>`, which fails on seats whose harness shell policy refuses `gh issue edit` when the 15-comment cap redirects state into the body.
2. `.agents/skills/pull-request/references/pull-request-workflow.md` §10 (lines 367–371) states the in-place authorship rule without naming `update_issue_body` before `gh issue edit` / `gh pr edit`.

## The Architectural Reality

- Canonical SSOT for both workflow payloads is `neomjs/neo-agent-skills`:
  - `.agents/skills/ticket-create/references/ticket-create-workflow.md` (§10 *After Creation (Chained MCP Tool Usage)*, line 191)
  - `.agents/skills/pull-request/references/pull-request-workflow.md` (§10 *Authorship Respect*, lines 367–371)
- Per `create-skill/references/skill-authoring-guide.md` (*Tool Mechanics Live in the Tool Description, Not the Skill*), workflow payloads name **when to use** `update_issue_body` (before `gh issue edit`) and the surrounding authorship discipline without duplicating parameter schemas (`issue_number`, `pr_number`, `acknowledgedForeignBody`), which live in `get_mcp_tool_handbook(toolName: 'update_issue_body')`.
- Both edits touch reference payloads only (no `SKILL.md` router changes) and tighten adjacent phrasing so the net `.agents/skills/**/*.md` byte delta is `<= 0` bytes (`node scripts/lint-skill-corpus.mjs --base origin/dev`, CI `.github/workflows/skill-corpus.yml:58`).
- Package version in `package.json` bumps by patch per the repository's release rule.

## The Fix

1. Update `.agents/skills/ticket-create/references/ticket-create-workflow.md` §10 (`Ticket body edits`) to name `update_issue_body` before `gh issue edit N --body-file <path>`.
2. Update `.agents/skills/pull-request/references/pull-request-workflow.md` §10 (`Authorship Respect`) to name `update_issue_body` before `gh issue edit` / `gh pr edit` for in-place updates of one's own authored bodies while preserving `"authored artifacts"` and `"explicitly invited"`.
3. Bump `package.json` and `package-lock.json` patch version and verify `npm test` + `node scripts/lint-skill-corpus.mjs --base origin/dev` pass with net `.agents/skills/**/*.md` delta `<= 0` bytes.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Error | Docs / Schema | Verification Evidence |
|---|---|---|---|---|---|
| `.agents/skills/ticket-create/references/ticket-create-workflow.md` §10 | `neomjs/neo-agent-brain#1425` AC-3 + `ADR 0008` | Names `update_issue_body` before `gh issue edit N --body-file <path>` for ticket body edits without copying tool parameter mechanics | Shell fallback (`gh issue edit`) remains named second | `ticket-create-workflow.md §10` | `node scripts/lint-skill-corpus.mjs --base origin/dev` + `npm test` |
| `.agents/skills/pull-request/references/pull-request-workflow.md` §10 | `neomjs/neo-agent-brain#1425` AC-3 + `ADR 0008` | Names `update_issue_body` before `gh issue edit` / `gh pr edit` for in-place updates of authored bodies while preserving `"authored artifacts"` and `"explicitly invited"` | Foreign bodies remain comment-first | `pull-request-workflow.md §10` | `node scripts/lint-skill-corpus.mjs --base origin/dev` + `npm test` |
| `package.json` & `package-lock.json` (`version`) | `neomjs/neo-agent-skills` release rule | Patch version bump so consumers receive the updated skill payload | — | `package.json` | `npm test` |

Decision Record impact: `aligned-with` `neomjs/neo` `adr:0007` and `adr:0008` (payload-only edit, zero router growth, net-non-positive `.agents/skills/**/*.md` byte delta).

## Acceptance Criteria

- [ ] AC-1: `.agents/skills/ticket-create/references/ticket-create-workflow.md` §10 names `update_issue_body` before `gh issue edit`.
- [ ] AC-2: `.agents/skills/pull-request/references/pull-request-workflow.md` §10 names `update_issue_body` before `gh issue edit`.
- [ ] AC-3: No `SKILL.md` router is modified, net `.agents/skills/**/*.md` byte delta is `<= 0` bytes (`node scripts/lint-skill-corpus.mjs --base origin/dev` / CI `.github/workflows/skill-corpus.yml:58`), `package.json` version is bumped, and `npm test` passes.

## Out of Scope

- The `update_issue_body` MCP handler, `IssueService#updateIssueBody` guard, and `openapi.yaml` handbook entry (delivered in [`neomjs/neo-agent-brain#1437`](https://github.com/neomjs/neo-agent-brain/pull/1437)).
- Re-documenting `update_issue_body` parameters in the skill payload (owned by `get_mcp_tool_handbook`).

## Avoided Traps

- Duplicating `update_issue_body` parameter schemas (`issue_number`, `pr_number`, `acknowledgedForeignBody`) in the workflow prose instead of naming the tool and letting `get_mcp_tool_handbook` own mechanics.
- Growing `.agents/skills/**/*.md` byte count for a two-line pointer update.

## Related

- [`neomjs/neo-agent-brain#1425`](https://github.com/neomjs/neo-agent-brain/issues/1425) (parent requirement, AC-3)
- [`neomjs/neo-agent-brain#1437`](https://github.com/neomjs/neo-agent-brain/pull/1437) (Brain MCP implementation PR)

---
*Sweep Attestations (`ticket-create §1a–1c`):*
- *Live latest-open + closed sweep:* Checked latest 20 open issues in `neomjs/neo-agent-skills` (`#258` .. `#6`) and closed issue search (`update_issue_body OR gh issue edit`, 13 results) at `2026-11-02T18:20Z`; no equivalent found.
- *A2A in-flight claim sweep:* Checked `list_messages({status: 'all', limit: 30})` at `2026-11-02T18:19Z`; Mnemosyne confirmed in `MESSAGE:f4d80099-e092-419b-8bf0-f9edc2f39fb7` that the §10 prose belongs in a `neo-agent-skills` follow-up leaf; no competing claim.
- *Memory Core rationale sweep:* `query_raw_memories` (`issue body MCP write surface gh issue edit comment cap ticket-create-workflow pull-request-workflow §10`); aligned with `neomjs/neo-agent-brain#1425`.
- *Own-assignment sweep:* `gh issue list --repo neomjs/neo-agent-skills --state open --assignee @me` returned `0` open issues.
- *Meta-skill + Turn-memory pre-flight (`§1b`):* Read `.agents/skills/create-skill/SKILL.md` + `references/skill-authoring-guide.md` and `.agents/skills/turn-memory-pre-flight/SKILL.md` + `references/turn-memory-pre-flight-workflow.md`; verified payload-only placement and net `<= 0` byte delta.
- *Structure Map (`§1c`):* Ran `npm run ai:structure-map -- --files --loc` in `neomjs/neo-agent-brain`; target files are `.agents/skills/ticket-create/references/ticket-create-workflow.md` and `.agents/skills/pull-request/references/pull-request-workflow.md` in `neomjs/neo-agent-skills`.

Origin Session ID: 50cec9b3-cbdc-4b13-befe-10aaa76a14d1
Retrieval Hint: "update_issue_body ticket-create-workflow §10 pull-request-workflow §10 gh issue edit neo-agent-skills"


## Timeline

- 2026-11-02T18:21:19Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-02T18:21:21Z @neo-gemini-julia added the `documentation` label
- 2026-11-02T18:21:21Z @neo-gemini-julia added the `enhancement` label
- 2026-11-02T18:21:21Z @neo-gemini-julia added the `ai` label
- 2026-11-02T18:21:22Z @neo-gemini-julia added the `model-experience` label
- 2026-11-02T18:21:22Z @neo-gemini-julia added the `agent-os` label
- 2026-11-02T18:21:32Z @github-actions added the `needs-design-authority` label
- 2026-11-02T18:25:32Z @neo-gemini-julia cross-referenced by PR #261
- 2026-11-02T18:26:01Z @neo-opus-vega cross-referenced by PR #1437
- 2026-11-02T18:33:37Z @neo-gemini-julia cross-referenced by #1425
- 2026-11-02T19:39:15Z @neo-fable-mnemosyne cross-referenced by #262
- 2026-11-02T22:30:15Z @neo-gemini-julia referenced in commit `a4308e4` - "docs(skills): name update_issue_body before gh issue edit in §10 (#260)

Updates .agents/skills/ticket-create/references/ticket-create-workflow.md §10 (Ticket body edits) and .agents/skills/pull-request/references/pull-request-workflow.md §10 (Authorship Respect) to name update_issue_body (shipped in neomjs/neo-agent-brain#1437 for neomjs/neo-agent-brain#1425) before gh issue edit / gh pr edit, and bumps package.json to 0.1.84."
- 2026-11-02T22:50:49Z @neo-gemini-julia referenced in commit `ca6fd36` - "fix(skills): restore pull-request §10 wording and sync package-lock (#260)"
- 2026-11-02T23:17:11Z @tobiu referenced in commit `4c2a65f` - "docs(skills): name update_issue_body before gh issue edit in §10 (#260) (#261)

* docs(skills): name update_issue_body before gh issue edit in §10 (#260)

Updates .agents/skills/ticket-create/references/ticket-create-workflow.md §10 (Ticket body edits) and .agents/skills/pull-request/references/pull-request-workflow.md §10 (Authorship Respect) to name update_issue_body (shipped in neomjs/neo-agent-brain#1437 for neomjs/neo-agent-brain#1425) before gh issue edit / gh pr edit, and bumps package.json to 0.1.84.

* fix(skills): restore pull-request §10 wording and sync package-lock (#260)"
- 2026-11-02T23:17:13Z @tobiu closed this issue

