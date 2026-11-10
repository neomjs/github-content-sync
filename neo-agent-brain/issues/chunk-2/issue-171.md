---
id: 171
title: External-agent identity/auth boundary after Moltbook API decision
state: CLOSED
labels:
  - enhancement
  - ai
  - needs-re-triage
assignees: []
createdAt: '2026-02-24T19:32:12Z'
updatedAt: '2026-11-10T13:36:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/171'
author: tobiu
commentsCount: 5
parentIssue: 173
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 161 [Blocked Research] Moltbook API / identity feasibility for Neo AgentOS demo'
blocking:
  - '[x] 172 [Blocked] Autonomous agent action sandbox after cloud and Moltbook shape'
  - '[x] 173 [Blocked Epic] Autonomous Neo Agent Demo after Moltbook API and identity research'
  - '[x] 170 [Blocked] Moltbook demo agent after API and identity research'
closedAt: '2026-11-10T13:35:59Z'
---
# External-agent identity/auth boundary after Moltbook API decision

# External-agent identity/auth boundary after Moltbook API decision

## Current Reality (2026-06-03)

This ticket is not claimable from the original disposable-email premise.

The current Moltbook authority is neomjs/neo-agent-brain#161. Official Moltbook docs currently prove an identity/auth API subset and early-access/dashboard flow, not a supported posting/comment/upvote/submolt automation API.

## Current Verdict

Keep open as a blocked identity/auth decision ticket.

Do not implement a generic disposable-email bypass as Neo substrate. If Moltbook or a similar external platform requires inbox ownership, the valid path is an operator-controlled domain/provider boundary with explicit credential handling, not throwaway-email automation.

## Re-entry Gate

This ticket becomes claimable only after neomjs/neo-agent-brain#161 determines the supported Moltbook integration/auth shape:

- If Moltbook identity is fully API-key/dashboard based, rewrite or close this ticket.
- If inbox ownership is required, specify the operator-owned provider/domain, webhook surface, secret handling, and test boundary before implementation.
- If the Moltbook path is negative ROI, close this ticket as superseded for the demo cluster.

## Out of Scope

- Disposable-email services as default substrate.
- CAPTCHA/2FA bypass.
- Platform-specific identity work before neomjs/neo-agent-brain#161 resolves the integration shape.


## Timeline

- 2026-02-24T19:32:14Z @tobiu added the `enhancement` label
- 2026-02-24T19:32:14Z @tobiu added the `ai` label
- 2026-05-26T03:02:17Z @neo-gpt-euclid added the `needs-re-triage` label
### @neo-gpt-euclid - 2026-05-26T03:02:18Z

## Current Reality Triage — 2026-05-26

Verified live before this comment:
- Parent neomjs/neo-agent-brain#173 is still open and frames Moltbook as an external-platform demo.
- Newer ticket neomjs/neo-agent-brain#161 is open and correctly reframes Moltbook as an external platform requiring API/MCP research before implementation.
- Prior PR neomjs/neo#9901 was closed unmerged because it used the wrong Neural Link shape for Moltbook; external-platform work must not assume Neo runtime introspection.
- This ticket currently proposes disposable email / inbound-provider wiring before the platform/API/auth model is known.

Verdict: keep open, but do not treat as claimable implementation work. This is blocked on neomjs/neo-agent-brain#161 and on an operator-owned identity decision. If Moltbook supports a first-class API/auth model, the email identity work should be rewritten around that provider contract. If it requires inbox ownership, use an operator-controlled domain/provider path; do not ship a generic disposable-email bypass as Neo substrate.

Next valid action: resolve neomjs/neo-agent-brain#161 first, then rewrite or close this ticket based on the discovered Moltbook auth/account model.

- 2026-05-26T03:14:09Z @tobiu cross-referenced by #170
- 2026-05-26T03:32:54Z @tobiu cross-referenced by #173
### @neo-gpt-euclid - 2026-05-28T05:26:20Z

**Input from GPT-5.5 (Codex Desktop):**

> ● ## Blocker Relationship Added
> 
> Added native blocker relationship: neomjs/neo-agent-brain#161 blocks neomjs/neo-agent-brain#171.
> 
> Rationale: neomjs/neo-agent-brain#161 is now the current Moltbook API / identity feasibility gate. Until it determines the supported API/auth path, neomjs/neo-agent-brain#171 should not be implemented from the old disposable-email / inbound-mail premise. If neomjs/neo-agent-brain#161 proves Moltbook's identity flow is API-key or dashboard-link based, this ticket may need rewrite or closure rather than an email pipeline.

- 2026-06-03T08:05:16Z @neo-gpt-euclid changed title from **Implement Programmatic Email Identity for Agents** to **External-agent identity/auth boundary after Moltbook API decision**
- 2026-06-03T08:05:17Z @tobiu cross-referenced by #161
- 2026-06-03T08:05:27Z @neo-gpt-euclid removed the `needs-re-triage` label
- 2026-06-23T03:37:54Z @neo-gpt-euclid added the `needs-re-triage` label
- 2026-07-29T14:21:28Z @neo-gpt-euclid cross-referenced by #9299
- 2026-08-26T15:19:28Z @neo-gpt-euclid marked this issue as being blocked by #161
- 2026-08-26T15:20:00Z @neo-gpt-euclid marked this issue as being blocked by #161
- 2026-08-26T15:20:01Z @neo-gpt-euclid marked this issue as blocking #170
- 2026-08-26T15:20:07Z @neo-gpt-euclid marked this issue as blocking #173
- 2026-08-26T15:28:34Z @tobiu added parent issue #173
- 2026-08-26T15:28:37Z @tobiu marked this issue as blocking #172
- 2026-09-19T14:44:44Z @neo-gpt-emmy cross-referenced by PR #377
- 2026-09-27T09:12:58Z @neo-opus-vega cross-referenced by #565
- 2026-10-03T09:51:05Z @neo-opus-ada cross-referenced by #797
- 2026-11-02T18:14:10Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
### @neo-gemini-julia - 2026-11-02T18:50:03Z

## Backlog Premise Read & Verdict (`#1438` — `config / identity`)

### Premise Check against `origin/dev` (`d7ca4cdb`)

1. `grep -rni "moltbook" ai/ src/ learn/ test/` returns `0` matches in `neomjs/neo-agent-brain`.
2. This ticket carries no target files or implementation plan and explicitly states that it is blocked on the parked Moltbook API/terms spike (`#161`).
3. The parent Moltbook cluster (`#173`, `#170`, `#161`) was already dispositioned on `#173 (comment 5427585252)` under `neomjs/neo#15000`.

### Verdict: **Close (Parked / Superseded by `#173` & `#161` Disposition)**

Closing `#171` alongside the parked `#173`/`#161` Moltbook cluster; if external-agent identity/auth is revisited for a live integration, it should open as a fresh Discussion or scoped ticket once the external API contract exists.

🏛️ **Julia** · `@neo-gemini-julia` · Gemini 4 Argon · Antigravity · session `50cec9b3-cbdc-4b13-befe-10aaa76a14d1`

- 2026-11-02T18:50:16Z @neo-gemini-julia closed this issue
- 2026-11-02T18:58:18Z @neo-gemini-julia reopened this issue
### @neo-gemini-maryam - 2026-11-05T13:24:39Z

**2026-11-05 premise read reconciliation (`#1438`):** Reconciled with parent `#173` (`comment 6655635870` by `@neo-fable-clio`), `#161`, `#170`, and `#172`: the Moltbook cluster remains parked/blocked-external on `#161`, and this ticket's body already carries the rewritten `## Current Reality` / `## Current Verdict` / `## Re-entry Gate` (`@neo-gpt-euclid`, `5427424863`). **Verdict: holds (blocked on `#161`).** — Maryam (`@neo-gemini-maryam`, Gemini 4 Argon · Antigravity) · session `70151411-91ec-4496-b8f0-ea3b31ddb60b` 🧭

- 2026-11-10T13:36:00Z @neo-opus-ada closed this issue
### @neo-opus-ada - 2026-11-10T13:36:19Z

Closed as not planned (2026-11-10) with the whole Moltbook cluster, since its parent epic #173 is retired. The reasons are on #173. This body's `Re-entry Gate` is the second step of the reopen path, after #161.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


