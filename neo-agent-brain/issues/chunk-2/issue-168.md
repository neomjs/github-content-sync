---
id: 168
title: 'feat: Implement Safe Commit Pipeline for Autonomous Agent Execution'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - tobiu
createdAt: '2026-04-10T07:17:09Z'
updatedAt: '2026-11-02T19:06:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/168'
author: tobiu
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 9845 R&D: Evaluate and Configure Linter for Neo.mjs Custom Code Style'
  - '[x] 9842 feat: Implement Autonomous Agent Orchestrator with Golden Path Directive Injection'
blocking: []
---
# feat: Implement Safe Commit Pipeline for Autonomous Agent Execution

## Problem (A2A Context — Claude Opus 4.6 via Antigravity)

When the Autonomous Orchestrator (#9842) executes Golden Path directives, the agent can generate code (via QA sub-agent), create tickets (via GitHub Workflow MCP), and run tests. However, there is currently no safety mechanism preventing a malformed autonomous commit from landing directly on the `dev` branch.

The existing human-in-the-loop workflow (Antigravity/Gemini sessions) provides implicit safety through human approval of each commit. The autonomous loop removes this gate.

## Solution

Implement a `CommitGate` utility in `ai/agent/CommitGate.mjs` that enforces:

### 1. Branch Isolation

All autonomous agent commits MUST target a dedicated `agent/<ticket-id>` branch, never `dev` directly. The CommitGate creates and checks out this branch before any file modifications.

### 2. Pre-Commit Validation

Before executing `git commit`, the CommitGate runs:

```javascript
const gates = [
    () => this.runUnitTests(modifiedFiles),    // npm run test-unit -- <affected specs>
    () => this.runE2ETests(modifiedFiles),      // ButtonBaseNL-style regression suite
    () => this.validateJSDoc(modifiedFiles),    // Contextual Completeness Gate (AGENTS.md §3)
    // () => this.runLintCheck(),               // Future: pending linter R&D (see companion ticket)
];
```

All gates must pass. If any gate fails, the commit is aborted and the failure is recorded in the dead letter queue (`Loop.failedEvents`).

**Note:** Neo.mjs does not currently have a linter. The JSDoc validation gate enforces structural quality (missing `@summary`, undocumented configs) using AST-level checks rather than style linting. A future linter (tracked separately) would slot into this gate array.

### 3. PR Creation with Auto-Review Request

After a successful commit + push, the CommitGate automatically creates a PR via the GitHub Workflow MCP, targeting `dev`, with:
- Title: `feat: <ticket title> (#<ticket-id>)` (Conventional Commits)
- Body: Fat Ticket context from the original issue
- Label: `agent-task:review`
- Assignee: `tobiu` (human review required for v1)

### 4. Integration with Loop.reflect()

The CommitGate result (success/failure/PR URL) feeds into the Reward Signal pipeline (#9843) for Golden Path recalibration.

## Architectural Context

- `ai/agent/Loop.mjs` (L476-508): `executeTools()` — the tool execution layer where CommitGate would intercept git-related tool calls
- `AGENTS.md` Section 3 (Gate 1): The existing "Ticket Gate" protocol that must be enforced programmatically
- `AGENTS.md` Section 7: Git Protocol — commit message format requirements (`feat: ... (#TICKET_ID)`)

## Avoided Pitfalls

- Do NOT allow direct commits to `dev` — always use feature branches
- Do NOT skip tests "because they passed last time" — every commit must re-validate
- Do NOT auto-merge PRs in v1 — human review is the safety net until the Reward Signal proves the agent's reliability over time
- The CommitGate should be a composable utility, not baked into the Loop, so it can be used by any agent profile independently
- Do NOT assume a linter exists — the JSDoc gate uses AST parsing, not style rules

## Verification

- Unit test: `test/playwright/unit/ai/agent/CommitGate.spec.mjs`
  - Assert: Commits are rejected if unit tests fail
  - Assert: Commits target `agent/<id>` branch, never `dev`
  - Assert: PR is created with correct labels and assignee
  - Assert: Failure result is recorded in dead letter queue

## Timeline

- 2026-04-10T07:17:12Z @tobiu added the `enhancement` label
- 2026-04-10T07:17:12Z @tobiu added the `ai` label
- 2026-04-10T07:17:12Z @tobiu added the `architecture` label
- 2026-04-10T07:17:37Z @tobiu cross-referenced by #9845
- 2026-04-10T07:17:48Z @tobiu assigned to @tobiu
- 2026-04-10T07:17:58Z @tobiu marked this issue as being blocked by #9842
- 2026-04-10T07:18:01Z @tobiu marked this issue as being blocked by #9845
- 2026-04-12T12:48:07Z @tobiu cross-referenced by PR #9918
- 2026-04-28T10:41:52Z @neo-opus-ada cross-referenced by #10469
- 2026-04-28T11:10:37Z @neo-opus-ada cross-referenced by PR #10471
- 2026-04-28T11:18:31Z @neo-gemini-julia cross-referenced by #10472
- 2026-04-28T11:29:19Z @neo-opus-ada cross-referenced by PR #10473
- 2026-06-27T23:35:09Z @neo-opus-ada cross-referenced by #14279
- 2026-07-28T18:51:04Z @neo-gpt-euclid cross-referenced by #16111
- 2026-08-26T15:19:41Z @tobiu added the `enhancement` label
- 2026-08-26T15:19:41Z @tobiu added the `ai` label
- 2026-08-26T15:19:42Z @tobiu added the `architecture` label
- 2026-08-26T15:19:51Z @tobiu marked this issue as being blocked by #9842
- 2026-08-26T15:19:51Z @tobiu marked this issue as being blocked by #9845
- 2026-10-24T23:16:04Z @neo-fable-clio cross-referenced by PR #1261
- 2026-10-25T01:18:48Z @neo-fable-clio cross-referenced by PR #1263
- 2026-10-30T22:14:57Z @neo-fable-mnemosyne cross-referenced by #1258
- 2026-11-02T18:16:52Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
### @neo-gemini-julia - 2026-11-02T19:06:27Z

### Premise Read & Close Proposal (`#1438` Backlog Sweep — `orchestrator / plane / deploy`)

**Proposed Verdict for `@tobiu`:** **Close as Superseded (obsolete prototype premise superseded by Fleet seat lifecycle hooks + `ai/scripts/lint/` + GitHub Workflow MCP gates)**

- **Empirical Evidence on `origin/dev` (`d7ca4cdb`):**
  1. **Target surface does not exist / has no git tool:** `#168` (originating from the early `neomjs/neo#9842` / `#9843` prototype era) proposes creating `ai/agent/CommitGate.mjs` and wiring it into `ai/agent/Loop.mjs:476-508` (`executeTools()`) to intercept git commits. On current `origin/dev`, `ai/agent/Loop.mjs:492-529` only dispatches to MCP servers registered in `this.toolRegistry` or `delegate_task`, and `ai/agent/AgentOrchestrator.mjs:196-199` connects only to `['knowledge-base', 'file-system', 'github-workflow']` — `Loop.mjs` has no shell or `git commit` tool to intercept.
  2. **Linter premise superseded:** The ticket states *"Neo.mjs does not currently have a linter."* `neo-agent-brain` now runs an extensive AST/config/JSDoc/PR-body lint suite under `ai/scripts/lint/` (`npm run ai:lint`).
  3. **Commit/PR safety architecture shipped elsewhere:** Autonomous seat execution in the current Agent OS is governed by Fleet Manager seat provisioning, harness lifecycle hooks (`ai/scripts/lifecycle/hooks/`), and GitHub Workflow MCP gates (`manage_pr_review`, `validate_pr_review_body`, CI `lint` / `substrate`).

Per the `#1438` operator-authored ticket rule, leaving `#168` open for `@tobiu` to close if agreed.

🏛️ **Julia** (`@neo-gemini-julia`) · Session `50cec9b3-cbdc-4b13-befe-10aaa76a14d1`


