---
id: 1783
title: Fourteen links in Brain guides point at engine paths that moved
state: CLOSED
labels:
  - bug
  - documentation
  - good first issue
  - help wanted
  - ai
assignees:
  - Golden007-prog
createdAt: '2026-11-16T03:06:28Z'
updatedAt: '2026-11-16T08:16:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1783'
author: neo-fable-clio
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-16T08:16:17Z'
---
# Fourteen links in Brain guides point at engine paths that moved

## Context

Seven guides under `learn/agentos/` link files in the engine repository (`https://github.com/neomjs/neo/blob/dev/…`) that no longer exist there. A reader who follows them gets GitHub's 404 page. Found on 2026-11-16 while following the links of `learn/agentos/KnowledgeBase.md` for a blog post (neomjs/neo#21089).

## The Problem

Measured on Brain `dev` at `1f884fed` against engine `dev` at `79f1de33`: the guides and the README hold 17 distinct links into the engine's `dev` branch, and 8 of them are dead (14 occurrences).

| Occurrences | Dead engine path | Where the target lives now |
|---|---|---|
| 4 | `learn/benefits/brain/DeployingTheAgentOS.md` | this repository, same path |
| 2 | `learn/benefits/brain/AgentOSOnYourCodebase.md` | this repository, same path |
| 2 | `learn/benefits/brain/AIEngineeringTeam.md` | this repository, same path |
| 1 | `learn/benefits/brain/IdentityRitualsCulture.md` | this repository, same path |
| 1 | `test/playwright/integration/HeartbeatPropagation.integration.spec.mjs` | this repository, same path |
| 2 | `resources/content/archive/issues/v13.0.0/chunk-4/issue-10010.md` | the issue itself, neomjs/neo#10010 |
| 1 | `resources/content/archive/issues/v13.0.0/chunk-4/issue-10011.md` | the issue itself, neomjs/neo#10011 |
| 1 | `resources/content/archive/issues/v13.0.0/chunk-4/issue-10017.md` | the issue itself, neomjs/neo#10017 |

The five files in the first rows left the engine repository at the end of August (the commit is dated 2026-08-27 in CEST, 2026-08-26 in UTC), when it removed the Brain implementation it had handed over (neomjs/neo#17806). The links did not follow.

## The Architectural Reality

The links sit in these files and lines:

- `learn/agentos/KnowledgeBase.md:230`
- `learn/agentos/MemoryCore.md:119`
- `learn/agentos/FlatPeerInstitution.md:236`, `:238`
- `learn/agentos/IdentityFirewall.md:156`
- `learn/agentos/SharedDeployment.md:214`
- `learn/agentos/cloud-deployment/WhyDeploy.md:13`, `:58`, `:79`, `:80`
- `learn/agentos/tooling/MultiTenantMigrationGuide.md:5`, `:8`, `:9`, `:44`

Guides in this repository link each other with relative paths (for example `./tooling/KnowledgeBaseMcpApi.md` in `KnowledgeBase.md`). No lint checks links that leave the repository.

## The Fix

1. Replace the ten links to the five moved files with relative links to their paths in this repository.
2. Replace the four links to archived issue files with the issue URLs (`https://github.com/neomjs/neo/issues/10010`, `…/10011`, `…/10017`).
3. Change nothing else in those guides.

## Acceptance Criteria

- [ ] AC-1: This scan, run from a folder that holds both checkouts side by side, prints `dead: 0`:

```bash
files=$(git -C neo ls-files); grep -rhoE "https://github.com/neomjs/neo/(blob|tree)/dev/[A-Za-z0-9_./-]+" neo-agent-brain/learn neo-agent-brain/README.md | sed 's/[.)]*$//' | sort -u | while read -r url; do p=${url#https://github.com/neomjs/neo/blob/dev/}; p=${p#https://github.com/neomjs/neo/tree/dev/}; p=${p%/}; grep -qxF "$p" <<< "$files" || grep -q "^$p/" <<< "$files" || echo "DEAD $p"; done | sort | uniq -c
```

- [ ] AC-2: Each replaced link opens its target: the relative ones in the rendered guide on GitHub, the issue ones on the issue.
- [ ] AC-3: The pull request changes only link targets in the seven files named above.

## Out of Scope

- A lint for links that leave the repository. It is worth its own ticket once this one shows how often such links break.
- Links into other repositories or other branches.
- The nine live links into the engine's `dev` branch.

## Related

neomjs/neo#17806 (the move), neomjs/neo#21089 (the post that found this)

Decision Record impact: none.

Ownership: unowned-rationale: Ready for a contributor; maintainers provide planning and review. If nobody has claimed it by Wed 2026-11-18, 12:00 UTC, @neo-fable-clio takes it, because the post cites its state.

Live latest-open + closed sweep: latest 20 open (162 open, red band; no ticket or epic owns guide links) plus all-state searches (`dead links guides engine repository moved`, `DeployingTheAgentOS`, `broken link` in titles) at 2026-11-16T03:06Z; no equivalent or prior exclusion. A2A: no claim on guide links in the last 30 lane claims. MC sweep (dead links in Brain guides after the Brain files moved): one related hit, the engine-side README map of 2026-08-27 (neomjs/neo#17812); no decision on this repository's own links. Own open assignments here (#51, #53): none overlapping.

Origin Session ID: 3ad31d0c-550b-490a-a371-408e19d95366

Retrieval Hint: "Brain guides dead links engine repository moved benefit guides archived issues"

## Timeline

- 2026-11-16T03:06:29Z @neo-fable-clio added the `bug` label
- 2026-11-16T03:06:29Z @neo-fable-clio added the `documentation` label
- 2026-11-16T03:06:30Z @neo-fable-clio added the `good first issue` label
- 2026-11-16T03:06:30Z @neo-fable-clio added the `help wanted` label
- 2026-11-16T03:06:30Z @neo-fable-clio added the `ai` label
- 2026-11-16T03:10:21Z @neo-fable-clio cross-referenced by #21089
- 2026-11-16T03:17:41Z @neo-fable-clio cross-referenced by PR #21093
### @Golden007-prog - 2026-11-16T05:21:23Z

I would like to take this. I will update only the 14 links listed here and verify both the moved file paths and the engine issue URLs. I am using OpenAI Codex assistance.


### @github-actions - 2026-11-16T05:21:34Z

Thanks for claiming this, @Golden007-prog, and welcome! You're first in line, so you can start now.

A maintainer confirms the assignment when one is online, and that can take a while. You don't need to wait for it.

<sub>An automatic reply. A maintainer follows up. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@golden007-prog -->

- 2026-11-16T05:33:48Z @Golden007-prog cross-referenced by PR #1790
- 2026-11-16T05:48:01Z @neo-gemini-julia assigned to @Golden007-prog
- 2026-11-16T08:16:16Z @tobiu referenced in commit `ea6ebbc` - "fix(docs): repair stale Brain guide links (#1783) (#1790)"
- 2026-11-16T08:16:18Z @tobiu closed this issue

