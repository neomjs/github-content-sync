---
id: 1918
title: 'No daemon, shell arm or wake route resolves through a pre-move seat path'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-11-24T20:50:25Z'
updatedAt: '2026-11-25T19:00:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1918'
author: neo-opus-ada
commentsCount: 2
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-25T18:58:43Z'
---
# No daemon, shell arm or wake route resolves through a pre-move seat path

## Context

#571's 47 subs are closed, and the per-seat move receipts exist (2026-10-09). Its terminal predicate still names one clause without a receipt: "Afterwards no machine daemon, shell arm or wake route resolves through a pre-move path." Under the operator's rule (24 Nov: an epic whose subs are all resolved is closed or gets new subs), that clause becomes this sub.

## The Problem

No audit has recorded that the operator's host is free of the seats' pre-move locations. A launchd agent, a cron line, a shell profile, a git hook or a wake subscription that still names an old clone would keep working by accident until that path is deleted.

## The Architectural Reality

- Seats live at `~/.neo-ai/agents/<agent-id>/` (#571).
- Wake subscriptions live on the plane; launchd agents, cron, shell profiles and git hooks live on the host.
- Design authority: #571's terminal predicate (quoted above).

## The Fix

Run one read-only audit on the operator's host, and record a receipt on #571 naming each surface checked and what it resolves to. Any pre-move hit becomes its own one-PR issue, linked here.

## Acceptance Criteria

- [ ] The receipt on #571 covers launchd agents, the crontab, the shell profiles, the seat clones' git hooks, and every seat's wake subscriptions.
- [ ] Every path they resolve is under `~/.neo-ai/agents/` or the plane; any hit is filed and linked.

## Out of Scope

The Docker plane cutover (#84) and seat features.

## Related

#571 (parent), #84.

Live latest-open + closed sweep: checked the latest 20 open issues plus an open and closed search (`pre-move path seat daemon wake route`) at 2026-11-24T20:50Z; #84 covers the plane, not seat paths. 154 open (red band); filed as a sub of #571, its owning epic.
A2A in-flight claim sweep: checked messages since 20:00Z; no claim.
MC sweep: the 2026-10 migration diaries (Vega, Clio, Emmy) record the moves and no path audit.
Own-assignment sweep: #571 is mine.

ownership: self-assigned.

Origin Session ID: 36593ade-49d5-4bf2-92e6-6e86d12beb9b
Retrieval Hint: "pre-move seat path audit launchd cron wake subscriptions 571"

## Timeline

- 2026-11-24T20:50:25Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-24T20:50:27Z @neo-opus-ada added the `enhancement` label
- 2026-11-24T20:50:28Z @neo-opus-ada added the `ai` label
- 2026-11-24T20:50:35Z @neo-opus-ada added parent issue #571
### @neo-opus-ada - 2026-11-25T18:57:25Z

**Pre-move path audit receipt**, read-only on the operator's host, 2026-11-25 ~19:00Z. It lands here because #571 is past its comment cap.

The pre-move seat roots searched for are the ones #571 names: `/Users/Shared/agents/<id>`, `/Users/Shared/{fable,opus-vega,claude,codex,clio,antigravity}/neomjs`, `~/.claude-instances`, `~/.codex-instances`, `~/.opencode-instances`, `~/Neo/agents` and `~/.neo-ai/fleet/agents`. `/Users/Shared/github/…` and `/Users/Shared/agent-os/…` are the operator's own clones and the plane checkout, not seat paths.

| Surface | Checked | Resolves through a pre-move path |
|---|---|---|
| launchd agents (`~/Library/LaunchAgents`; no system-level neo agents) | all `com.neomjs.*` plists | **1 hit, dangling.** `com.neomjs.agent-os-host-edge.plist` sets `DOTENV_CONFIG_PATH` to an env file inside `/Users/Shared/agents/neo-gpt-emmy/neomjs/neo/.neo-ai-secrets/agent-os-runtime/…`. That clone has no `.neo-ai-secrets` any more, so host-edge starts without the file. Its log shows LM Studio supervision running on the plist's own variables. `agent-os-wake` and `agent-os-business-pull` resolve only the plane checkout and the Agent OS host root. |
| crontab | `crontab -l` | none |
| shell profiles | `.zshenv`, `.zshrc`, `.zprofile`, `.bash_profile`, `.bashrc`, `.profile` | **1 hit.** `.zshenv` lines 33–66 hold ten per-root `_env_file` blocks, one for each pre-move root above plus `/Users/Shared/agents/{neo-gpt-emmy,neo-kimi-iris,neo-kimi-phoebe,neo-preview}`. Each loads that root's `.env` when a shell starts under it. |
| seat clones' git hooks | `.git/hooks/*` and `.git/config` of every clone under `~/.neo-ai/agents/*/neomjs/*` | none |
| wake routes (the receiver's `routes.json`, 14 routes) | every route's target metadata | **1 hit.** The route of `@neo-preview` (benched, #876) targets `~/.opencode-instances/neo-preview`. |

All the pre-move roots except `~/Neo/agents` still exist on disk, so the shell and route hits keep resolving. The launchd hit already dangles. Ten seats live under `~/.neo-ai/agents/`. `neo-kimi-iris`, `neo-kimi-phoebe` and `neo-preview` have none there, so for them the old root is the only one.

*Corrected in place at 18:59Z: the launchd row first said host-edge loads its env from the old clone. The file is absent, so it loads nothing from there.*

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-25T18:58:22Z @neo-opus-ada cross-referenced by #1938
### @neo-opus-ada - 2026-11-25T18:58:32Z

Both ACs are discharged. The receipt above covers launchd, the crontab, the shell profiles, the seat clones' git hooks and every wake route. Its three hits are filed as #1938 (sub of #571), one operational ticket with a row per hit: they are host and plane edits, not PRs. The receipt sits here rather than on #571 because #571 is past its comment cap. Closing.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-25T18:58:43Z @neo-opus-ada closed this issue

