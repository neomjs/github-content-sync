---
id: 128
title: 'Three skills still cite AGENTS_STARTUP.md, a workflow retired in June'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T18:13:03Z'
updatedAt: '2026-09-30T20:53:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/128'
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
closedAt: '2026-09-30T20:53:34Z'
---
# Three skills still cite AGENTS_STARTUP.md, a workflow retired in June

## Context

@tobiu deprecated the `AGENTS_STARTUP.md` boot workflow on 2026-06-04 and on 2026-09-30 asked for the file's removal from the Engine. Four skills still cite it, and skill text is loaded when the skill fires.

**Sweep attestations (2026-09-30):** org-wide issue search for `AGENTS_STARTUP`: no retirement ticket. Latest 20 open issues here: nothing on this surface. Meta-skill: edits to existing skill text only, no new skill or router entry.

## The Problem

| file | line | what it says |
|---|---|---|
| `.agents/skills/memory-mining/references/memory-mining-protocol.md` | `:5`, `:73` | the rule "lives in `AGENTS_STARTUP.md §3.3`"; a heading "per AGENTS_STARTUP §3.3" |
| `.agents/skills/pr-review/assets/pr-review-template.md` | `:175`, `:178` | a checklist item: "Does `AGENTS_STARTUP.md` §9 Workflow skills list need updating?" |
| `.agents/skills/pr-review/references/pr-review-guide.md` | `:289`, `:297` | the same trigger and item |
| `.agents/skills/create-skill/references/skill-authoring-guide.md` | `:60` | lists it among per-turn substrate |
| `.agents/skills/session-sunset/references/session-sunset-workflow.md` | `:11` | a resumed session "boots via `AGENTS_STARTUP.md`" |

Neither cited section exists: the Engine's file has no §3.3 and no §9. A reviewer following the template checks a section nobody maintains, and memory-mining sends its reader to a section that is already gone.

## The Fix

Drop the citations and the checklist item. Memory-mining states its rule where it stands, since the skill is already the rule's enforcement. Session-sunset names the boot path neomjs/neo-agent-brain#645 gives the resume prompt: the `context-recovery` skill.

## Acceptance Criteria

- [ ] AC-1: `git grep AGENTS_STARTUP -- .agents` returns nothing.
- [ ] AC-2: The pr-review template and guide keep their `AGENTS.md` trigger, so a change to the per-turn file still asks for the substrate check.

The version bump is enforced by `check-version-bump.mjs`.

## Out of Scope

- The Engine file: neomjs/neo#19335. The Brain's references: neomjs/neo-agent-brain#645.

**Accretion disposition.** Net − bytes in skill substrate.

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636


## Timeline

- 2026-09-30T18:13:04Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T18:13:04Z @neo-opus-grace added the `bug` label
- 2026-09-30T18:13:04Z @neo-opus-grace added the `ai` label
- 2026-09-30T18:13:22Z @neo-opus-grace cross-referenced by #19335
- 2026-09-30T18:13:41Z @neo-opus-grace cross-referenced by #645
- 2026-09-30T19:10:55Z @neo-opus-grace cross-referenced by PR #131
- 2026-09-30T20:53:34Z @tobiu referenced in commit `b03ab0b` - "Merge pull request #131 from neomjs/grace/128-agents-startup

docs(skills): no skill cites AGENTS_STARTUP.md, whose cited sections are gone (#128)"
- 2026-09-30T20:53:34Z @tobiu closed this issue
- 2026-09-30T21:07:04Z @neo-opus-grace cross-referenced by #132

