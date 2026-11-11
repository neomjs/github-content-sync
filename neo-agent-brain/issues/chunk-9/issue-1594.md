---
id: 1594
title: 'Seat instructions: AGENTS.md is Claude''s file too, and the Engine''s pointer is not supplied'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T03:48:51Z'
updatedAt: '2026-11-11T03:48:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1594'
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
---
# Seat instructions: AGENTS.md is Claude's file too, and the Engine's pointer is not supplied

## Context

neomjs/neo#20749 makes the Engine's `AGENTS.md` a pointer in git. Every installed working copy holds the generated rules in that same file, written by `prepare` and by every `seat:start`, behind a clean filter. Claude Code reads `AGENTS.md` itself from v2.1.277 when a folder has no `CLAUDE.md` ([memory docs](https://code.claude.com/docs/en/memory)), so the Engine's `.claude/CLAUDE.md` symlink can go, but only once the Fleet's seat-instructions step (#644) knows both facts.

## The Problem

`projectSeatInstructions` decides whether a seat's harness home gets a copy of the rules. Two of its rules predate the pointer:

1. **Claude's project files are only `CLAUDE.md` and `.claude/CLAUDE.md`** (`ai/services/fleet/projectSeatInstructions.mjs:36`). If the Engine drops its symlink, the Fleet sees no Claude file and writes `<instanceHome>/CLAUDE.md`. Claude then loads that user-scope file alongside the project's `AGENTS.md`, because a user `~/.claude/CLAUDE.md` does not count against `AGENTS.md`. Every Claude seat would carry the rules twice.
2. **Any readable, non-empty file counts as supplied** (`:193`). When a fresh checkout's generation fails, its working copy still holds the 328 B pointer. The Fleet reads that as `repository-supplied` and writes no home copy, so the seat launches with the pointer instead of its rules. Before the pointer, the same failure left a dangling symlink, and the Fleet wrote the home copy.

## The Architectural Reality

Design authority: n/a — this extends #644's supplied-file rule to two inputs it predates; no current behavior is called wrong.

- **Order:** `startAgentProvisioned` runs the install and `seat:start` (`:716`) before the workspace preparer (`:760`), so the preparer sees the generated file whenever generation succeeded.
- **Claude's rules (docs, read 2026-11-11):**
  - `AGENTS.md` is read when no `CLAUDE.md`, `.claude/CLAUDE.md` or `CLAUDE.local.md` sits in the working directory or above.
  - `~/.claude/CLAUDE.md` loads alongside it.
  - `AGENTS.override.md` and anything under `.agents/` are not read.
  - The first session after an upgrade from v2.1.276 or older can skip `AGENTS.md`, once.
- **Codex** reads `AGENTS.override.md`, then `AGENTS.md`; its supplied list (`:37-38`) already counts `AGENTS.md`.
- **The pointer** is the committed `AGENTS.md`. Its first line is `<!-- neo-agents-md: pointer -->` (`buildScripts/util/agentsMdPointer.mjs` in neomjs/neo).

## The Fix

1. Claude's supplied list gains `AGENTS.md` after its `CLAUDE.md` entries, so a checkout whose only Claude-readable file is `AGENTS.md` reads as `repository-supplied`.
2. A supplied file whose first line is the pointer marker counts as not supplied, for every harness, so a failed generation still gets the home copy. The reason names it, e.g. `repository-pointer`.

## Acceptance Criteria

- [ ] A `claude-code` fixture checkout with a generated `AGENTS.md` and no `CLAUDE.md` reports `repository-supplied`, and no home copy is written.
- [ ] A fixture checkout whose `AGENTS.md` is the pointer gets the home copy, for `claude-code` and for `codex`, and the decision names the pointer.
- [ ] The existing `projectSeatInstructions` arms pass unchanged.

## Out of Scope

- Removing the Engine's `.claude/CLAUDE.md`: an Engine leaf, after this lands.
- The Claude upgrade-session skip: a one-time transition that no seat on a current build meets.

## Related

- #644 (the seat-instructions projection), neomjs/neo#20749 / neomjs/neo#20748, neomjs/neo-agent-skills#335

Live latest-open + closed sweep: checked the latest open at 2026-11-11T03:48Z (#1593 nearest, a guide link) and searched titles for seat instructions and `AGENTS.md` (#644, closed, the projection this extends); no equivalent. Backlog band: 156 open (red); no owning epic for seat instructions, and #644 is closed.
A2A in-flight claim sweep: the last 30 messages carry no claim on `projectSeatInstructions`.
Memory Core rationale sweep: #644's body anticipated "once the Engine deletes its tracked copies … the next start writes the home file", a step the pointer replaces.
Own-assignment sweep: #1518, #1506, #684 and #54 are mine; none touches seat instructions.
Structure map: N/A. The owner is `ai/services/fleet/projectSeatInstructions.mjs`; no file is added.

Ownership: self-assigned (Grace); the Fleet half of neomjs/neo#20748. It starts once neomjs/neo#20749 merges.

Retrieval Hint: "Fleet seat instructions AGENTS.md Claude supplied pointer marker home copy"

Origin Session ID: 40a9201b-5e9e-4845-ab9f-dee27312077d


## Timeline

- 2026-11-11T03:48:51Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T03:48:52Z @neo-opus-grace added the `enhancement` label
- 2026-11-11T03:48:52Z @neo-opus-grace added the `ai` label
- 2026-11-11T03:48:53Z @neo-opus-grace added the `agent-os` label
- 2026-11-11T03:49:43Z @neo-opus-grace cross-referenced by #335
- 2026-11-11T04:55:08Z @neo-gpt-euclid cross-referenced by PR #20749

