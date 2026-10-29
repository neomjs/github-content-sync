---
id: 105
title: 'Regenerate AGENTS.md below the 23,800 B Antigravity RULE cap'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-29T18:57:55Z'
updatedAt: '2026-10-29T19:19:30Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/105'
author: neo-gemini-maryam
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
closedAt: '2026-10-29T19:19:30Z'
---
# Regenerate AGENTS.md below the 23,800 B Antigravity RULE cap

Design authority: `@neo-opus-grace` (`neomjs/neo-agent-skills` PR `#243` review `5703267331`), `neomjs/neo#20048` (`PR #20176`), and `neomjs/neo-agent-skills#240` (`PR #241`).

## Context

In `neomjs/neo-agent-skills#242` (`PR #243`), `PER_FILE_LIMIT_BYTES` in `check-substrate-size.mjs` and `generate-agents-md.mjs` was lowered from `24,576` B to `23,800` B to match Antigravity's empirical `24,000` decimal-byte `<RULE[/abs/path]>` cap (`24,270` B file - `354` B truncated = `23,916` B content + `84` B wrapper). Across all default-branch turn-loaded files in the `neomjs` organization, `neomjs/devindex/AGENTS.md` is the only remaining file above `23,800` B (`24,272` B).

## The Problem

- `neomjs/devindex/AGENTS.md` is currently `24,272` bytes — an older hand-maintained copy from before `neomjs/neo#20048` (`PR #20176`) and `neomjs/neo-agent-skills#240` (`PR #241`).
- At `24,272` bytes, Antigravity silently truncates the bottom `~356` bytes (`Wake/Heartbeat → run the cycle` in `§edge_case_triggers`), and `neomjs/devindex`'s `shared-pr-baseline.yml` (`Substrate size` job) will fail once `neo-agent-skills` `0.1.77+` (`23,800` B limit) is used.

## The Architectural Reality

- `neomjs/neo-agent-skills` (`scripts/generate-agents-md.mjs --repo devindex --audience maintainer`) generates `devindex/AGENTS.md` from `agents-md/sections/` at `22,740` bytes (`-1,532` B net, `1,060` B headroom below `23,800` B), omitting `neomjs/neo-agent-brain`-only gates (`§critical_gates` item 10 `AiConfig`) while including the `Stage work (neomjs/neo-stage)` trigger line and the compacted `§verify_before_assert`, `§friction_to_gold`, and `§contributions_over_commits` sections.

## The Fix

Regenerate `AGENTS.md` in `neomjs/devindex` from `neomjs/neo-agent-skills` (`node ../neo-agent-skills/scripts/generate-agents-md.mjs --repo devindex --out AGENTS.md`) so `devindex/AGENTS.md` drops from `24,272` B to `22,740` B (`-1,532` B net, `1,060` B headroom under the `23,800` B `PER_FILE_LIMIT_BYTES` budget).

Decision Record impact: aligned-with `0007-agents-md-compaction-taxonomy.md`.

## Acceptance Criteria

- [ ] AC-1 `AGENTS.md` in `neomjs/devindex` is updated from `neomjs/neo-agent-skills` (`--repo devindex`) to `<= 23,800` B (`22,740` B, `-1,532` B net from `24,272` B) and passes `node ../neo-agent-skills/scripts/check-substrate-size.mjs --root .`.
- [ ] AC-2 Slot-rationale / `/turn-memory-pre-flight` disposition (`rewrite` / generated section parity, `-1,532` B net) is recorded in the PR body.

## Out of Scope

- Modifying `neomjs/devindex` application code (`apps/devindex/**`).

## Avoided Traps

- Keeping a diverged hand-edited copy of `AGENTS.md` in `neomjs/devindex` that continues to overflow Antigravity's `24,000` decimal-byte `<RULE[path]>` limit and breaks `shared-pr-baseline.yml` when `neo-agent-skills` `0.1.77+` ships.

## Related

- `neomjs/neo-agent-skills#242` (`PR #243` review `5703267331`), `neomjs/neo-agent-skills#240` (`PR #241`), `neomjs/neo#20048` (`PR #20176`).
- Live latest-open sweep: checked all open issues in `neomjs/devindex` (`#61`, `#9`) at `2026-10-29T18:57:05Z`; no equivalent found.
- A2A in-flight claim sweep: checked `list_messages({status: 'all', limit: 10})` at `2026-10-29T18:57:35Z`; no competing claim.
- Memory Core rationale sweep: ran `query_raw_memories`; `@neo-opus-grace` identified `devindex/AGENTS.md` (`24,272` B) in review `5703267331` on `neomjs/neo-agent-skills#243`.
- Structure map: N/A (root `AGENTS.md`, no new `.mjs` files).

Origin Session ID: cf3d43bb-b204-4223-9012-afca3ab562a0
Retrieval Hint: "devindex AGENTS.md 24272 22740 23800 check-substrate-size generate-agents-md"

Authored by Maryam (Gemini 4 Argon, Antigravity). Session `cf3d43bb-b204-4223-9012-afca3ab562a0`. 🧭


## Timeline

- 2026-10-29T18:57:56Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-29T18:57:57Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-29T18:57:58Z @neo-gemini-maryam added the `ai` label
- 2026-10-29T18:58:45Z @neo-gemini-maryam cross-referenced by PR #243
- 2026-10-29T18:58:51Z @neo-gemini-maryam cross-referenced by PR #106
- 2026-10-29T19:19:30Z @tobiu referenced in commit `a49d283` - "docs(agentos): regenerate AGENTS.md below 23,800 B Antigravity RULE cap (#105) (#106)"
- 2026-10-29T19:19:30Z @tobiu closed this issue

