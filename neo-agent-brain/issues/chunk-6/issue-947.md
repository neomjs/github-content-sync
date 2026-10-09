---
id: 947
title: A seat-provisioning spec still pins 9 projected hook files; Fleet projects 11
state: OPEN
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-09T04:07:06Z'
updatedAt: '2026-10-09T04:08:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/947'
author: neo-opus-ada
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
# A seat-provisioning spec still pins 9 projected hook files; Fleet projects 11

## Context

`test/playwright/unit/ai/scripts/migrations/bootstrapWorktree.spec.mjs`, test "#250 provisioning PLACES the seat hooks — the projector is not merely callable", asserts `expect(expected.length).toBe(9)`, the comment says "the seven executables and both config artifacts". `enumerateProjection` now returns 11 entries, nine executables plus two config artifacts. #752 (2026-10-02) added `claude/wakeListenerHook.mjs` and #934 (2026-10-08) added `claude/harnessIdGuardHook.mjs`. Neither updated this pin. The test fails on clean `dev` 03da5025 (defect-note fingerprint `ee5bdd8511179e08`). Brain Unit CI blocks only new reds, so it has stayed red since 10-02.

## The Problem

The pin is a conscious-update guard: a hook added to the projection has to be counted here deliberately. While it is red it guards nothing, and it hides more than itself. The count is the test's first assertion, so the placement, executable-bit, settings-composition and clean-`git status` checks after it have not run since 10-02. #250's acceptance, that provisioning places the hooks, has been untested for a week.

## The Architectural Reality

- `ai/scripts/lifecycle/hooks/projectSeatHooks.mjs` `enumerateProjection` returns 11 entries. Five are Claude executables, two Codex and two Kimi; the other two are `.codex/hooks.json` and `.kimi-code/hooks/turn-presence.example.toml`.
- `projectSeatHooks.spec.mjs` already pins the executable census (5/2/2), updated by #934.

## The Fix

Pin 11, and let the comment name the parts: the nine executables `projectSeatHooks.spec.mjs` pins, plus both config artifacts. The next assertion was stale too, which the red count hid: provisioning also writes `PROVENANCE_RECEIPT` (`.agents/seat-projection.json`, pushed into `written` by `projectSeatHooks.mjs`), so the expected list includes it.

## Acceptance Criteria

- [ ] The test passes on `dev`, and its later assertions (placement, executable bit, settings composition, clean `git status`) run again. Red first: on `dev` it fails at the count.

## Out of Scope

- Brain Unit CI's tolerance of base reds.
- Deriving the count from `enumerateProjection` itself: an empty projection would then pass vacuously, which is what the pin exists to stop.

## Related

#250 · #752 · #934 · #571

Decision Record impact: none.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-09T04:06:37Z, no equivalent. Exact org search "bootstrapWorktree" / "PLACES the seat hooks": only #57 (live plane state in three unit specs), unrelated.
A2A sweep: no claim on this scope in the latest messages.
MC sweep: "bootstrapWorktree provisioning places the seat hooks test expects 9 projected files red base unit failure", 5 results, no prior decision about this pin.
Own-assignment sweep: none overlapping.

Origin Session ID: 3290d205-ee4b-4a8e-949f-fe4995197e08
Retrieval Hint: "bootstrapWorktree seat hook projection count pin 9 11 base red"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-09T04:07:06Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-09T04:07:07Z @neo-opus-ada added the `bug` label
- 2026-10-09T04:07:08Z @neo-opus-ada added the `ai` label
- 2026-10-09T04:07:08Z @neo-opus-ada added the `testing` label
- 2026-10-09T04:07:08Z @neo-opus-ada added the `agent-os` label
- 2026-10-09T04:11:31Z @neo-opus-ada cross-referenced by PR #948

