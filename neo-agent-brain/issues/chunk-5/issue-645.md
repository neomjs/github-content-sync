---
id: 645
title: 'The orchestrator''s resume path tells a fresh session to read AGENTS_STARTUP.md, retired in June'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T18:13:00Z'
updatedAt: '2026-09-30T19:38:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/645'
author: neo-opus-grace
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
closedAt: '2026-09-30T19:38:44Z'
---
# The orchestrator's resume path tells a fresh session to read AGENTS_STARTUP.md, retired in June

## Context

@tobiu deprecated the `AGENTS_STARTUP.md` boot workflow on 2026-06-04 ("outdated, we are not even using it anymore") and on 2026-09-30 asked for the file's removal from the Engine. The Brain still sends sessions to it.

**Sweep attestations (2026-09-30):** org-wide issue search for `AGENTS_STARTUP` finds no retirement ticket (three open issues mention it in passing: neomjs/neo#14230, #144, #154). Latest 20 open issues here: nothing on this surface. Memory Core: the June deprecation, no decision to keep it. A2A: no claim.

## The Problem

`ai/scripts/lifecycle/resumeHarness.mjs:305` builds the boot prompt a resumed seat receives: *"hi ${identity}, please read @AGENTS_STARTUP.md, then call add_memory once as a boot heartbeat…"*. The orchestrator's `SwarmHeartbeatService` calls it on the sunset path (`:403`). Once the Engine drops the file, a resumed seat's first instruction names a file that is not there.

Four more places list the file as live substrate:

| file | line | role |
|---|---|---|
| `ai/scripts/lint/lint-agents.mjs` | `:42`, `:216`, `:313` | the per-turn substrate list the lint measures |
| `ai/scripts/diagnostics/check-retired-primitives.mjs` | `:51` | an `ACTIVE_ANTIGRAVITY_ROOTS` scan root; the file's own JSDoc says an absent root fails the whole check |
| `ai/scripts/diagnostics/consumerRelevanceMap.mjs` | `:86` | a subsystem mapping |
| `ai/scripts/migrations/bootstrapWorktree.mjs` | `:124` | a `@see` |

## The Fix

- The resume prompt names nothing that is retired. Maintainer instructions already load through the harness (the Engine's `AGENTS.md` today, the seat's home file once #644 lands), so the prompt keeps the boot heartbeat and sends recovery to the `context-recovery` skill.
- `AGENTS_STARTUP.md` leaves the three lists and the `@see`.

## Acceptance Criteria

- [ ] AC-1: `resumeHarness`'s boot prompt names no `AGENTS_STARTUP.md`, and its spec asserts the prompt it sends.
- [ ] AC-2: No code, lint or test path names `AGENTS_STARTUP.md`. The one JSDoc `@see` in `ai/scripts/migrations/bootstrapWorktree.mjs` stays until that file's own comments are cleaned, since editing it inherits the archaeology guard's audit of the whole file.
- [ ] AC-3: `check-retired-primitives` and `lint-agents` pass with the entry gone, run against an Engine checkout without the file.

## Out of Scope

- Deleting the file and its Engine references: neomjs/neo#19335, which lands after this one so no resume prompt dangles.
- The skills that cite it: neomjs/neo-agent-skills#128.

## Related

- #644 — the seat's home instructions, which make a boot-time read unnecessary.
- neomjs/neo#18985 — the contributor door, which still names the file.

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636


## Timeline

- 2026-09-30T18:13:01Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T18:13:01Z @neo-opus-grace added the `bug` label
- 2026-09-30T18:13:02Z @neo-opus-grace added the `ai` label
- 2026-09-30T18:13:02Z @neo-opus-grace added the `agent-os` label
- 2026-09-30T18:13:22Z @neo-opus-grace cross-referenced by #19335
- 2026-09-30T18:13:42Z @neo-opus-grace cross-referenced by #128
- 2026-09-30T18:53:55Z @neo-opus-grace cross-referenced by PR #647
- 2026-09-30T19:37:13Z @neo-gpt-emmy cross-referenced by PR #131
- 2026-09-30T19:38:44Z @tobiu referenced in commit `b5ceb43` - "Merge pull request #647 from neomjs/grace/645-resume-prompt

fix(lifecycle): a resumed seat starts from context recovery, not the retired AGENTS_STARTUP.md (#645)"
- 2026-09-30T19:38:44Z @tobiu closed this issue

