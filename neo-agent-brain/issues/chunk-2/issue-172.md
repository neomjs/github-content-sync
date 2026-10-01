---
id: 172
title: '[Blocked] Autonomous agent action sandbox after cloud and Moltbook shape'
state: OPEN
labels:
  - enhancement
  - ai
  - build
  - needs-re-triage
assignees: []
createdAt: '2026-02-24T19:32:10Z'
updatedAt: '2026-10-01T18:43:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/172'
author: tobiu
commentsCount: 1
parentIssue: 173
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 171 External-agent identity/auth boundary after Moltbook API decision'
  - '[ ] 161 [Blocked Research] Moltbook API / identity feasibility for Neo AgentOS demo'
blocking: []
---
# [Blocked] Autonomous agent action sandbox after cloud and Moltbook shape

# External agent action sandbox after Moltbook shape resolves

## Current Reality (2026-06-03)

The useful intent is still valid: autonomous agents may need an isolated execution/action environment before driving external surfaces, executing untrusted code, or running browser-like workflows.

The old `Dockerfile.agent` request is stale:

- The production Agent OS container topology already exists under `ai/deploy/*`.
- neomjs/neo-agent-brain#161 is the current authority for Moltbook API/MCP feasibility.
- neomjs/neo#9297 and neomjs/neo-agent-brain#170 are blocked on identity/auth plus Moltbook integration shape.
- No dedicated `Dockerfile.agent` or `ai/demo-agents/moltbook/` implementation exists.

## Current Verdict

Keep open only as a post-research implementation ticket for an **agent action sandbox**, not as a duplicate of the cloud deployment Docker stack.

## Re-entry Gate

Before implementation:

- neomjs/neo-agent-brain#161 resolves whether Moltbook integration is API/MCP-based, browser-based, or negative ROI.
- neomjs/neo#9297 resolves the identity/auth boundary for autonomous external actions.
- The implementation scope distinguishes:
  - cloud Agent OS service containers (`ai/deploy/*`) — already shipped baseline;
  - untrusted code execution isolation;
  - optional browser/action automation isolation.

If the resolved path does not require a separate action sandbox, close this ticket as superseded by the cloud deployment stack.

## Out of Scope

- Creating a parallel `Dockerfile.agent` that duplicates `ai/deploy/Dockerfile`.
- Treating Chrome DevTools browser automation as the default before neomjs/neo-agent-brain#161 completes.
- Adding Moltbook-specific runtime under `ai/demo-agents/` before neomjs/neo-agent-brain#170 becomes claimable.


## Timeline

- 2026-02-24T19:32:11Z @tobiu added the `enhancement` label
- 2026-02-24T19:32:11Z @tobiu added the `ai` label
- 2026-02-24T19:32:11Z @tobiu added the `build` label
- 2026-05-26T03:23:37Z @neo-gpt changed title from **Create Docker Sandbox for Autonomous Agents** to **[Blocked] Autonomous agent action sandbox after cloud and Moltbook shape**
### @neo-gpt - 2026-05-26T03:23:48Z

Retargeted this ticket to current reality instead of closing it. The baseline Dockerized Agent OS cloud stack now exists under `ai/deploy/*`, so the old `Dockerfile.agent` / `ai/demo-agents` body is stale. The remaining valid intent is narrower: a future isolated **agent action sandbox** for untrusted execution and/or browser action automation after the Moltbook integration shape is known.

Verified before update:
- `ai/deploy/Dockerfile` and compose files cover the current KB/MC/Chroma/orchestrator cloud topology.
- `learn/agentos/cloud-deployment/Day0Tutorial.md` documents the current Dockerized remote-MCP proof path.
- neomjs/neo-agent-brain#161 remains the Moltbook API/MCP feasibility authority.
- neomjs/neo#9297 and neomjs/neo-agent-brain#170 are now blocked/stale-shape upstream siblings.
- No dedicated `Dockerfile.agent` or `ai/demo-agents/moltbook/` exists.

Current routing: blocked / needs re-triage, not claimable as a duplicate Docker-stack task.

- 2026-05-26T03:23:48Z @neo-gpt added the `needs-re-triage` label
- 2026-05-26T03:32:54Z @tobiu cross-referenced by #173
- 2026-06-03T08:05:17Z @tobiu cross-referenced by #161
- 2026-06-03T08:05:27Z @neo-gpt removed the `needs-re-triage` label
- 2026-06-23T03:37:58Z @neo-gpt added the `needs-re-triage` label
- 2026-08-26T15:19:27Z @neo-gpt marked this issue as being blocked by #161
- 2026-08-26T15:20:01Z @neo-gpt marked this issue as being blocked by #161
- 2026-08-26T15:28:36Z @tobiu added parent issue #173
- 2026-08-26T15:28:37Z @tobiu marked this issue as being blocked by #171

