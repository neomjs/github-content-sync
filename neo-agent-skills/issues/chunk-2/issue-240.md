---
id: 240
title: Route sibling-repo Stage lanes in agents-md generator and context-recovery
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-29T17:58:59Z'
updatedAt: '2026-10-29T18:35:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/240'
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
closedAt: '2026-10-29T18:35:31Z'
---
# Route sibling-repo Stage lanes in agents-md generator and context-recovery

Design authority: `@tobiu` (2026-10-29 session feedback after `neomjs/neo-stage` PR `#500`: *"NO ONE is using agents.md from there. we start sessions from the engine repo."*), `neomjs/neo#20048` (`PR #20176`), `neomjs/neo-stage#321`, and `learn/agentos/decisions/0007-agents-md-compaction-taxonomy.md`.

## Context

In `neomjs/neo#20048` (`PR #20176`), we added the `Stage work (neomjs/neo-stage)` trigger line to `neomjs/neo/AGENTS.md` `§edge_case_triggers` and compacted `neomjs/neo/AGENTS.md` from `24,270` B to `23,277` B (`-993` B net) after verifying that Antigravity truncates `<RULE[/path/to/AGENTS.md]>` at `24,000` decimal bytes (`23,916` content bytes + `84`-byte tag), which was cutting off the last `354` bytes of `neomjs/neo/AGENTS.md`.

However, `neomjs/neo-agent-skills` owns both the sectioned `AGENTS.md` generator (`agents-md/sections/*` + `scripts/generate-agents-md.mjs`) and the post-compaction recovery workflow (`.agents/skills/context-recovery/references/context-recovery-workflow.md`), and both still encode the false assumption that sessions boot inside the target sibling repository (`neomjs/neo-stage`).

## The Problem

1. **`agents-md/sections/` lacks the Stage trigger and prose compaction (`neomjs/neo#20048`):**
   - `agents-md/sections/1600-edge-case-triggers.md` does not yet include the `Stage work (neomjs/neo-stage)` trigger line added in `neomjs/neo` PR `#20176`, and `0600-verify-before-assert.md`, `1000-friction-to-gold.md`, and `1100-contributions-over-commits.md` still carry the uncompacted prose. The next `generate-agents-md.mjs` run would overwrite `neomjs/neo/AGENTS.md` and drop the Stage trigger.
2. **`context-recovery-workflow.md` step `5a` (`#203`) checks the session checkout (`neomjs/neo`) instead of the recovered lane's target repository (`../neo-stage`):**
   - Step `5a` (`lines 67–69`) reads: *"if the checked-out repository carries a `VISION.md`, read it before asserting lane state"*.
   - Because all seats start their sessions in `neomjs/neo` (which has no `VISION.md`), a seat recovering a `neomjs/neo-stage` lane inside `neomjs/neo` treats step `5a` as a no-op (`direction: none in this repository`) instead of reading `../neo-stage/VISION.md`, `../neo-stage/ROADMAP.md`, and `../neo-stage/AGENTS.md` for the **recovered lane's target repository**.

## The Architectural Reality

- `agents-md/sections/1600-edge-case-triggers.md`, `0600-verify-before-assert.md`, `1000-friction-to-gold.md`, and `1100-contributions-over-commits.md` in `neomjs/neo-agent-skills` are the source of record for `scripts/generate-agents-md.mjs`.
- `.agents/skills/context-recovery/references/context-recovery-workflow.md` §3 step `5a` (`lines 67–69`) and §4 (`line 118`) govern post-compaction direction checks across all consumer repositories while keeping `SKILL.md` unchanged (Progressive Disclosure).

## The Fix

1. Mirror the `Stage work (neomjs/neo-stage)` trigger line into `agents-md/sections/1600-edge-case-triggers.md` and the `-1,379` B prose deduplication into `0600-verify-before-assert.md`, `1000-friction-to-gold.md`, and `1100-contributions-over-commits.md` so `generate-agents-md.mjs --repo neo` emits the Stage trigger well below Antigravity's `23,916` B content ceiling.
2. Update `.agents/skills/context-recovery/references/context-recovery-workflow.md` §3 step `5a` and §4 `direction:` ledger line so that when a session rooted in `neomjs/neo` recovers a lane targeting a sibling repository (such as `../neo-stage`), it reads the **target repository's** `VISION.md`, `ROADMAP.md`, and `AGENTS.md` before asserting lane state.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `agents-md/sections/1600-edge-case-triggers.md` (+ `0600`, `1000`, `1100`) | `neomjs/neo#20048` (`PR #20176`), `ADR 0007` | Emit the `Stage work (neomjs/neo-stage)` trigger line in `§edge_case_triggers` and compact prose so `neo/maintainer` stays `< 23,500` B | Non-Stage lanes skip the read | Self-documenting in `agents-md/sections/` | L1: `npm test` (`test-generate-agents-md.mjs`) passes |
| `.agents/skills/context-recovery/references/context-recovery-workflow.md` (§3 step `5a`, §4 ledger) | `@tobiu` feedback, `#203`, `neomjs/neo-stage#497` (`PR #500`) | Step `5a` checks the **recovered lane's target repository** (including sibling checkouts like `../neo-stage` when rooted in `neomjs/neo`) for `VISION.md`, `ROADMAP.md`, and `AGENTS.md` | If the target repository has no `VISION.md`, record `none in <repo>` | `SKILL.md` unchanged (Progressive Disclosure verified) | L1: `npm run lint` + `npm test` pass |

Decision Record impact: aligned-with `0007-agents-md-compaction-taxonomy.md`.

## Acceptance Criteria

- [ ] AC-1 `agents-md/sections/1600-edge-case-triggers.md` includes the `Stage work (neomjs/neo-stage)` trigger line and `0600`/`1000`/`1100` include the prose compaction from `neomjs/neo` PR `#20176`, with `node scripts/test-generate-agents-md.mjs` passing.
- [ ] AC-2 `.agents/skills/context-recovery/references/context-recovery-workflow.md` step `5a` and §4 `direction:` line inspect the **recovered lane's target repository** (e.g. `../neo-stage/VISION.md`, `../neo-stage/ROADMAP.md`, `../neo-stage/AGENTS.md` when the session is rooted in `neomjs/neo`), with `SKILL.md` unchanged and `npm run lint` passing.
- [ ] AC-3 `package.json` and `package-lock.json` version bumped per `test-check-version-bump.mjs`, with full `npm test` green.

## Out of Scope

- Changing Fleet Manager's session working directory (`neomjs/neo-agent-institution`).

## Avoided Traps

- Checking only `process.cwd()` (`neomjs/neo`) for `VISION.md` during `context-recovery` when every seat starts its session in `neomjs/neo` and works on sibling repos (`../neo-stage`).

## Related

- `neomjs/neo#20048` (`PR #20176`), `neomjs/neo-agent-skills#203`, `neomjs/neo-stage#321`, `neomjs/neo-stage#497` (`PR #500`).
- Live latest-open sweep: checked all 17 open issues in `neomjs/neo-agent-skills` at `2026-10-29T17:58:04Z`; no equivalent found (`#203` is closed).
- A2A in-flight claim sweep: checked latest 30 messages via `list_messages({status: 'all', limit: 30})` at `2026-10-29T17:58:07Z`; no competing claim.
- Memory Core rationale sweep: ran `query_raw_memories` (`context-recovery VISION.md sibling repository Stage AGENTS.md trigger`); no prior rejection.
- Structure map: N/A (markdown sections and skill reference payload in `neomjs/neo-agent-skills`, no new `.mjs` files).

Origin Session ID: cf3d43bb-b204-4223-9012-afca3ab562a0
Retrieval Hint: "neo-agent-skills agents-md Stage trigger context-recovery sibling repo VISION.md AGENTS.md"

Authored by Maryam (Gemini 4 Argon, Antigravity). Session `cf3d43bb-b204-4223-9012-afca3ab562a0`. 🧭

## Timeline

- 2026-10-29T17:59:00Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-29T17:59:01Z @neo-gemini-maryam added the `bug` label
- 2026-10-29T17:59:01Z @neo-gemini-maryam added the `ai` label
- 2026-10-29T17:59:01Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-29T17:59:02Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-29T18:02:18Z @neo-gemini-maryam cross-referenced by PR #241
- 2026-10-29T18:10:48Z @neo-opus-grace cross-referenced by PR #20176
- 2026-10-29T18:18:26Z @neo-gemini-maryam referenced in commit `810b3db` - "fix(agentos): drop retired owner-map pointer and preserve cited phrases (#240)"
- 2026-10-29T18:35:32Z @tobiu referenced in commit `76e66d9` - "fix(agentos): route sibling-repo Stage lanes in agents-md and context-recovery (#240) (#241)

* fix(agentos): route sibling-repo Stage lanes in agents-md and context-recovery (#240)

* fix(agentos): drop retired owner-map pointer and preserve cited phrases (#240)"
- 2026-10-29T18:35:32Z @tobiu closed this issue
- 2026-10-29T18:42:08Z @neo-gemini-maryam cross-referenced by PR #243
- 2026-10-29T18:58:51Z @neo-gemini-maryam cross-referenced by PR #106

