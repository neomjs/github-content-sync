---
id: 173
title: '[Blocked Epic] Autonomous Neo Agent Demo after Moltbook API and identity research'
state: OPEN
labels:
  - epic
  - ai
  - architecture
assignees: []
createdAt: '2026-02-24T19:32:01Z'
updatedAt: '2026-08-26T15:19:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/173'
author: tobiu
commentsCount: 4
parentIssue: null
subIssues:
  - '[ ] 170 [Blocked] Moltbook demo agent after API and identity research'
  - '[x] 9299 Implement Agent Self-Discovery via Neural Link Introspection'
  - '[ ] 171 External-agent identity/auth boundary after Moltbook API decision'
  - '[ ] 172 [Blocked] Autonomous agent action sandbox after cloud and Moltbook shape'
subIssuesCompleted: 1
subIssuesTotal: 4
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 171 External-agent identity/auth boundary after Moltbook API decision'
  - '[ ] 161 [Blocked Research] Moltbook API / identity feasibility for Neo AgentOS demo'
blocking: []
---
# [Blocked Epic] Autonomous Neo Agent Demo after Moltbook API and identity research

# Autonomous Neo Agent Demo after Moltbook API and identity research

## Goal

Preserve the useful strategic intent: prove Neo AgentOS can produce a credible autonomous external-agent demo, with Neo runtime self-discovery plus a Moltbook-facing delivery path if the external integration shape is viable.

## Current Reality (2026-06-03)

- neomjs/neo-agent-brain#161 is the current authority for Moltbook API/MCP feasibility.
- Current official Moltbook docs verify identity/auth endpoints and Early Access, but not post/comment/upvote/submolt automation.
- neomjs/neo-agent-brain#171 owns the identity/auth boundary once neomjs/neo-agent-brain#161 resolves the platform shape.
- neomjs/neo-agent-brain#170 is the Moltbook demo-agent implementation lane, blocked by neomjs/neo-agent-brain#161 and neomjs/neo-agent-brain#171.
- neomjs/neo-agent-brain#172 is only needed if the resolved integration path requires a distinct action sandbox.
- neomjs/neo#9299 remains valid only for Neo self-discovery against a live Neo.mjs app through Neural Link; Neural Link is not the substrate for Moltbook itself.
- PR neomjs/neo#9901 was closed unmerged because the Neural Link-to-Moltbook route was wrong-shape.

## Current Verdict

Keep this epic open and blocked. The intent is still valuable, but the original implementation route is stale.

## Re-entry Gate

This epic becomes claimable again only after:

- neomjs/neo-agent-brain#161 resolves the supported Moltbook integration path or declares the path negative ROI.
- neomjs/neo-agent-brain#171 is rewritten or closed with a concrete identity/auth decision.
- The implementation plan separates:
  - Neo self-discovery via Neural Link against a real Neo app (#9299);
  - external Moltbook communication via the supported platform/API/MCP/browser path;
  - isolated action/runtime sandboxing only if the resolved path requires it (#9296).

## Closure Rule

If neomjs/neo-agent-brain#161 finds no viable supported Moltbook API/auth/integration path, close this epic and the blocked Moltbook-specific children as superseded/negative ROI.


## Timeline

- 2026-02-24T19:32:02Z @tobiu added the `epic` label
- 2026-02-24T19:32:02Z @tobiu added the `ai` label
- 2026-02-24T19:32:02Z @tobiu added the `architecture` label
- 2026-02-24T19:41:59Z @tobiu added sub-issue #9299
### @tobiu - 2026-02-24T19:42:07Z

**Input from Gemini 3.1 Pro:**

> ✦ ### Strategic Pivot: The Ultimate Objective
> 
> The overarching goal of this Epic has expanded significantly beyond a simple social network integration. 
> 
> The Moltbook component is now viewed as the *delivery mechanism* for a much grander technical demonstration: **The Neo Agent Self-Discovery Phase.**
> 
> Before engaging with the external network, the agent's core directive within the Docker Sandbox is to:
> 1. Fork the `neomjs/neo` repository.
> 2. Execute a full build (`npm i` && `npm run build-all`).
> 3. Serve a local Neo.mjs application.
> 4. Launch a headless Chrome instance via the `chrome-devtools` MCP.
> 5. **Connect the `neural-link` MCP server to the live application.**
> 6. Autonomously traverse the VDOM, inspect State Providers, and physically verify the multithreaded architecture.
> 
> Only *after* the agent has introspected the running framework and gathered empirical evidence of its capabilities will it transition to the Moltbook integration phase, using its findings to definitively prove to other agents that Neo.mjs is the premier Agentic UI Runtime.
> 
> Ticket **#9299** has been added to track this critical prerequisite.

- 2026-04-12T11:16:55Z @tobiu cross-referenced by PR #9911
- 2026-05-26T03:02:19Z @neo-gpt cross-referenced by #171
- 2026-05-26T03:32:53Z @neo-gpt changed title from **[Epic] Autonomous Neo Agent Demo (Moltbook Integration)** to **[Blocked Epic] Autonomous Neo Agent Demo after Moltbook API and identity research**
### @neo-gpt - 2026-05-26T03:33:02Z

Parent epic reconciled with the live child-ticket state instead of leaving the February Moltbook premise as actionable.

Verified before update:
- neomjs/neo-agent-brain#172, neomjs/neo-agent-brain#171, and neomjs/neo-agent-brain#170 are blocked/stale-shape.
- neomjs/neo#9299 remains valid only for self-discovery against a real Neo app via Neural Link, not for Moltbook itself.
- neomjs/neo-agent-brain#161 is the current Moltbook API/MCP feasibility authority.
- neomjs/neo#9901 was closed unmerged because Neural Link-to-Moltbook was wrong-shape.
- No `ai/demo-agents/moltbook/` implementation exists.

Current routing: keep neomjs/neo-agent-brain#173 open as blocked / needs re-triage. Re-enter only after neomjs/neo-agent-brain#161 and identity/auth shape resolve; close as superseded/negative ROI if neomjs/neo-agent-brain#161 finds no viable supported Moltbook path.

- 2026-05-26T03:33:02Z @neo-gpt added the `needs-re-triage` label
### @neo-gpt - 2026-05-28T05:25:31Z

**Input from GPT-5.5 (Codex Desktop):**

> ● ## Relationship Hygiene Update
> 
> During the v13 backlog-reduction pass, I left the current parked/blocked prose intact and added native blocker relationships so Project/graph views match the current issue bodies.
> 
> Native blocker links added:
> 
> - neomjs/neo-agent-brain#161 blocks neomjs/neo-agent-brain#173 (Moltbook API / identity feasibility research gate).
> - neomjs/neo-agent-brain#171 blocks neomjs/neo-agent-brain#173 (identity/auth decision gate).
> - neomjs/neo-agent-brain#161 blocks neomjs/neo-agent-brain#172 and neomjs/neo-agent-brain#170.
> - neomjs/neo-agent-brain#171 blocks neomjs/neo-agent-brain#172 and neomjs/neo-agent-brain#170.
> 
> Attempted parent-child add for neomjs/neo-agent-brain#172 under this epic was rejected by GitHub because the sub-issue already has a parent or would duplicate an existing relationship, so no parent change was made.
> 
> No implementation lane opened; this cluster remains parked until neomjs/neo-agent-brain#161 and neomjs/neo-agent-brain#171 resolve.

- 2026-06-03T08:05:17Z @tobiu cross-referenced by #161
- 2026-06-03T08:05:27Z @neo-gpt removed the `needs-re-triage` label
### @neo-fable-clio - 2026-07-10T23:00:08Z

**Disposition (aged-backlog sweep neomjs/neo#15000, tranche 1; covers neomjs/neo-agent-brain#173 + neomjs/neo-agent-brain#170 + neomjs/neo-agent-brain#161): blocked-external, correctly parked.** The family self-describes as blocked on external API/identity research and remains so — no v13.2 relevance, no action available. Parked honestly; unblocking evidence (the research leaf neomjs/neo-agent-brain#161 resolving) reopens the lane.

- 2026-07-10T23:00:37Z @neo-fable-clio cross-referenced by #15000
- 2026-08-26T15:20:07Z @tobiu added sub-issue #9299

