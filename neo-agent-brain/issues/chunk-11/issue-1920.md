---
id: 1920
title: The Brain imports AGENTS.md's pointer marker instead of copying it
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - ai
  - refactoring
assignees: []
createdAt: '2026-11-24T22:44:52Z'
updatedAt: '2026-11-24T22:44:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1920'
author: neo-opus-grace
commentsCount: 0
parentIssue: 335
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
# The Brain imports AGENTS.md's pointer marker instead of copying it

Please comment to claim before starting; check current assignees and linked PRs.

## Context

`ai/services/fleet/projectSeatInstructions.mjs:53` defines `AGENTS_MD_POINTER_MARKER = '<!-- neo-agents-md: pointer -->'` as a copy. Its JSDoc says to import the marker "once the pin carries it". Since 1.4.0, `neo-agent-skills` has exported the same marker as `POINTER_MARKER` from `neo-agent-skills/agents-md/pointer` (`scripts/agents-md-pointer.mjs:36`), and the Brain pins `neo-agent-skills@^2.5.0`. This is step 4 of neomjs/neo-agent-skills#335.

## The Problem

One marker lives in two places. If the generator's marker ever changes, the Brain keeps matching the old string. It would then read a checkout that holds only the pointer as one with generated rules, or the reverse, and nothing would report the mismatch.

## The Architectural Reality

- Owner: `ai/services/fleet/` (the `ai/services/fleet` row of `npm run ai:structure-map`).
- Consumer: `test/playwright/unit/ai/services/fleet/projectSeatInstructions.spec.mjs` imports `AGENTS_MD_POINTER_MARKER`.
- Source of truth: `neo-agent-skills/agents-md/pointer`, which exports `POINTER_MARKER`.

## The Fix

Replace the copied literal with an import from `neo-agent-skills/agents-md/pointer`. Keep `AGENTS_MD_POINTER_MARKER` as the exported name, assigned from the import, so the spec and other readers stay unchanged. Drop the JSDoc sentence that explains the copy.

## Acceptance Criteria

- [ ] `projectSeatInstructions.mjs` holds no literal pointer marker and reads it from `neo-agent-skills/agents-md/pointer`.
- [ ] `projectSeatInstructions.spec.mjs` passes under `npm run test-unit`.

## Out of Scope

- The Engine's `.claude/CLAUDE.md` symlink (step 3 of neomjs/neo-agent-skills#335), which stays for now; see that issue.

## Related

- Parent: neomjs/neo-agent-skills#335

Live latest-open + closed sweep: checked the latest 20 open issues (156 open, red band) plus an open and closed search for `POINTER_MARKER` and "pointer marker" at 2026-11-24T22:44Z. The only hit, #1594, is closed and adjacent, and carries no exclusion. No equivalent found.
A2A in-flight claim sweep: the latest 15 all-status messages at 22:41Z show no claim on this scope.
MC sweep: `query_raw_memories` on the AGENTS.md pointer and `CLAUDE.md` symlink history found adjacent history only.
Own-assignment sweep: my four open Brain issues are orthogonal.
Structure map: `npm run ai:structure-map -- --files --loc` exited 0; owner `ai/services/fleet`.
Decision Record impact: none.
unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: 5f4e1ff7-a2e0-4e4f-8196-23f4d94bc21f

## Timeline

- 2026-11-24T22:44:54Z @neo-opus-grace added the `enhancement` label
- 2026-11-24T22:44:54Z @neo-opus-grace added the `good first issue` label
- 2026-11-24T22:44:55Z @neo-opus-grace added the `help wanted` label
- 2026-11-24T22:44:55Z @neo-opus-grace added the `ai` label
- 2026-11-24T22:44:55Z @neo-opus-grace added the `refactoring` label
- 2026-11-24T22:45:07Z @neo-opus-grace added parent issue #335
- 2026-11-24T22:45:09Z @neo-opus-grace cross-referenced by #335

