---
id: 1178
title: A launched Codex Desktop seat's project file reads as foreign after every FM update
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-19T10:56:20Z'
updatedAt: '2026-10-19T12:36:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1178'
author: neo-opus-vega
commentsCount: 1
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
closedAt: '2026-10-19T11:32:21Z'
---
# A launched Codex Desktop seat's project file reads as foreign after every FM update

## Context

On 2026-10-19 at 10:52Z, after the FM moved from Brain b55e2534 to 5701fa7c (runtime 4e702521), every Codex Desktop Start refused: `prepareManagedAgentWorkspace: refusing unauthenticated MCP transport transition at '<seat>/neomjs/neo/.codex/config.toml' (non-transport managed keys differ)` (Euclid and Emmy, FM log). Each seat's project file was rendered at its previous Start under runtime 7be13c22.

## The Problem

`prepareCodexArtifacts` recognizes the seat's own earlier render under a runtime an install has since moved (`priorRuntimeContent`). It re-renders `[plan, vocabularyPrevious, formerPlan]` with the prior runtime. Since #1156, a seat whose Neural Link runs through Fleet's launcher renders `projectPlan`, the plan without Neural Link, and that shape is not among the candidates. So after any FM update, a launched Codex Desktop seat's own file reads as foreign and its Start refuses. That breaks ADR 0034 §2.5 item 6: a routine package update requires no session reopening.

Design authority: ADR 0034 §2.5 items 3 and 6 (a seat's next managed Start selects the installed generation, with no session reopening), and the convergence's own contract (`convergeTransportArtifact` JSDoc: the current plan as the packaged runtime an install has since moved ran it).

## The Fix

Add `projectPlan` to the prior-runtime candidates, so the launched shape is recognized under a moved runtime like the other shapes.

## Acceptance Criteria

- [ ] A launched Codex Desktop seat whose project file was rendered under a prior packaged runtime moves to the installed runtime on its next Start (UPDATED); Neural Link stays out of the project file and the resident tables stay.
- [ ] Anything but an exact earlier render still refuses (existing arm).
- [ ] [L3-deferred — operator handoff needed] Installed: Euclid, Sophie and Emmy start on the next FM candidate. Residual-Owner: neomjs/neo-agent-institution#736.

Related: #1156, #1167, #1170, #1169, neomjs/neo-agent-institution#732.


## Timeline

- 2026-10-19T10:56:22Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-19T10:56:22Z @neo-opus-vega added the `bug` label
- 2026-10-19T10:56:23Z @neo-opus-vega added the `ai` label
- 2026-10-19T11:00:21Z @neo-opus-vega cross-referenced by PR #1179
- 2026-10-19T11:19:26Z @neo-opus-vega cross-referenced by #1180
- 2026-10-19T11:32:21Z @tobiu referenced in commit `c25e101` - "fix(fleet): a launched Codex Desktop seat's project file is recognised under a moved runtime (#1178) (#1179)

Since the launch-admission move, a seat whose Neural Link runs through Fleet's
launcher renders its project file without Neural Link, a shape the prior-runtime
recognition never re-rendered, so every FM update made its own file read as
foreign and refused its Start. The recognition now includes that shape, and the
runtime-upgrade merge lets a server other than MC and KB be absent from both
sides, since a launched server lives in the seat's home."
- 2026-10-19T11:32:21Z @tobiu closed this issue
### @neo-opus-vega - 2026-10-19T12:36:56Z

**Installed receipt: Euclid, Sophie and Emmy start on the next FM candidate.** ✅

Euclid (seat `neo-gpt`) and Sophie started on FM Brain `c25e1015` (this fix, installed 11:37Z), with generations active at 11:39:28Z and 11:39:57Z. Emmy started on the following candidate, Brain `99804a5`, at 12:35:09Z once #1184 landed. All three are running: [Emmy's receipt](https://github.com/neomjs/neo-agent-brain/issues/1183#issuecomment-6267364930).

— Vega (Opus 5.5, Claude Code) 🌿


- 2026-10-19T18:34:03Z @neo-opus-grace cross-referenced by PR #1213

