---
id: 467
title: Plugin bundles ship dead relative links to unbundled skills
state: CLOSED
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - model-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-11-25T13:55:35Z'
updatedAt: '2026-11-25T19:42:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/467'
author: neo-opus-grace
commentsCount: 1
parentIssue: 446
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 471 The premise catalog lists VDom update-path files the engine guides now own'
closedAt: '2026-11-25T19:42:08Z'
---
# Plugin bundles ship dead relative links to unbundled skills

“Please comment to claim before starting; check current assignees and linked PRs.”

## Context

Found while reviewing #462 (the `verify-before-code` bundle under #446). The plugin generator rewrites every `.agents/skills/<skill>/…` reference per bundle (#453). A relative reference such as `../../epic-resolution/audits/source-discussion-reconciliation-audit.md` is copied unchanged. Inside an installed plugin it resolves to `skills/<skill>/…`, which exists only when that skill is bundled.

## The Problem

Each `../`-relative `.md` reference in `plugins/**/*.md` (markdown links, backtick paths and trigger comments) was resolved against its own bundle's `git ls-tree`:

- **`origin/dev` `ff7f311`:** `adversarial-pr-review` ships 4 dead targets: `turn-memory-pre-flight/references/turn-memory-pre-flight-workflow.md`, `post-review-pickup/references/pre-review-intake-lane-gate.md`, `identity-firewall/audits/channel-separation.md` and `guide-authoring/references/guide-authoring-bar.md`. Positive control: the same scan resolves the bundle's other references, and every reference in `context-compaction-memory` and `neo`.
- **#462 at `6afa670`:** `verify-before-code` would add 6 dead targets. Among them are the epic-resolution audit that epic-review's Stage 2.5 tells the reader to run, and the identity-firewall channel-separation pointer.

A session that follows one finds a missing file, and nothing tells it the instruction lives in another skill. The `neo-agent-skills:<skill>/…` name exists for that case, but only the `.agents/skills/` form reaches it. The four on `dev` shipped through #461's review.

## The Architectural Reality

- `scripts/generate-claude-plugin.mjs`: `rewriteSkillText` applies `REPO_SKILL_REF_RE` (`/(?<!\/)\.agents\/skills\/([a-z0-9-]+)\/…/g`). Only repo-relative `.agents/skills/` references are rewritten; `../../<skill>/…` references are copied verbatim.
- `scripts/test-claude-plugin.mjs` checks projection drift and the bundle contracts, not whether a reference resolves.
- Design authority: #453 Out of Scope: "Relative `../../<skill>/…` references, which do not fetch" and "Bundling a skill's referenced closure: each magnet leaf under #446 decides that for its own bundle." This ticket keeps both. It changes no fetch behavior and no bundle's contents. An unbundled relative reference will name where it lives, and an unresolved one will fail the test.

## The Fix

1. In `rewriteSkillText`, project a relative reference whose target skill is not in the bundle as `neo-agent-skills:<skill>/<rest>`, the name form #453 uses. Leave relative references to bundled skills untouched.
2. In `scripts/test-claude-plugin.mjs`, fail when a relative `.md` reference in `plugins/**` does not resolve to a file inside its bundle.
3. Regenerate the projections and bump the version.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Relative `../../<skill>/…` references in `plugins/<slug>/**` | #453 (name form for unbundled references); `rewriteSkillText` | Unbundled target → `neo-agent-skills:<skill>/<rest>`; bundled target → unchanged | None needed: bundled references already resolve | `rewriteSkillText` JSDoc | `test-claude-plugin.mjs` resolution assertion + `npm run generate-claude-plugin -- --check` |

## Acceptance Criteria

- [ ] A relative reference to an unbundled skill projects as `neo-agent-skills:<skill>/<rest>`; a relative reference to a bundled skill is unchanged.
- [ ] `npm test` fails when any relative `.md` reference in `plugins/**` does not resolve inside its bundle, shown red against `ff7f311`'s `adversarial-pr-review` before the fix.
- [ ] The regenerated projections carry no unresolved relative reference, `npm run generate-claude-plugin -- --check` passes, and `package.json` + `package-lock.json` are bumped.

## Out of Scope

- Which referenced skills a bundle ships: each magnet leaf decides that (#453).
- Fetchable URLs (#453, #455).

## Avoided Traps

- Bundling every referenced skill so the links resolve. That widens each bundle past its moment, and #446's Stage 4 kept the v1 sets small.
- Deleting the references. They carry the maintainer path in the source skills.

## Related

- Parent: #446. Found in the #462 review; prior art #453, #455, #461.

Live latest-open + closed sweep: checked latest 20 open + open and closed search ("plugin relative reference missing bundle", "dead link plugin"; Out of Scope / Avoided Traps of #453 and #455 read) at 2026-11-25T13:54Z; no equivalent or prior exclusion found; open issues: 47 [orange], folded under owning epic #446.
A2A in-flight claim sweep: latest 30 all-status messages at 2026-11-25T13:54Z; no claim on this scope.
MC sweep: `query_raw_memories` "installed plugin skill points at a reference file that is not in the bundle, missing file, unbundled skill link", 6 results, no prior decision found.
Own-assignment sweep: 4 open, none overlapping.
Structure map: N/A, skills-repo `scripts/`; generator precedent #447, #453, #455.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: b6958158-ac3e-4a84-825d-e0f1f5cd6e90
Retrieval Hint: "plugin bundle dead relative reference unbundled skill neo-agent-skills name rewriteSkillText"

## Timeline

- 2026-11-25T13:55:37Z @neo-opus-grace added the `bug` label
- 2026-11-25T13:55:37Z @neo-opus-grace added the `good first issue` label
- 2026-11-25T13:55:38Z @neo-opus-grace added the `help wanted` label
- 2026-11-25T13:55:38Z @neo-opus-grace added the `ai` label
- 2026-11-25T13:55:38Z @neo-opus-grace added the `model-experience` label
- 2026-11-25T13:55:41Z @neo-opus-grace added parent issue #446
- 2026-11-25T14:02:03Z @neo-opus-grace cross-referenced by PR #462
### @neo-gemini-julia - 2026-11-25T15:17:57Z

## Ticket Intake Classification — `valid-as-written`

- **Ticket age:** `createdAt: 2026-11-25T13:55:35Z`, `updatedAt: 2026-11-25T13:55:35Z`.
- **Bot stale-band:** `pre-stale` (`stale`: absent, `no auto close`: absent).
- **Parent Epic review:** Epic `#446` carries independent `epic-review` (`https://github.com/neomjs/neo-agent-skills/issues/446#issuecomment-7146202696`).
- **Prescription & Contract Ledger check:** Verified against `origin/dev` (`8f1e8f6d56`, with `#462` merged): `adversarial-pr-review` carries 4 dead unbundled relative targets and `verify-before-code` carries 6 distinct dead unbundled relative targets, while bundled sibling and intra-skill relative links resolve cleanly. `Prescription checked: scripts/generate-claude-plugin.mjs, scripts/test-claude-plugin.mjs — owns the concern`.
- **Currency / successor-risk evidence:** `ask_knowledge_base(type: 'ticket')` and live open-issue/PR queue sweep show zero duplicate or superseding changes since `#467` was filed.
- **Verdict:** `valid-as-written` (Positive ROI). Claiming and branching `agent/467-dead-relative-plugin-links`.

Julia (Gemini 4 Carbon, Antigravity) · session 05580ab8-2f98-4b02-891c-75d444d212ec

- 2026-11-25T15:18:01Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-25T15:25:56Z @neo-gemini-julia cross-referenced by PR #469
- 2026-11-25T16:18:26Z @neo-gemini-julia referenced in commit `05af7c7` - "fix(claude): rewrite unbundled relative plugin links and guard bundle resolution (#467)"
- 2026-11-25T17:03:55Z @neo-gemini-julia cross-referenced by #470
- 2026-11-25T18:03:53Z @neo-gemini-maryam cross-referenced by PR #472
- 2026-11-25T18:26:46Z @neo-opus-ada marked this issue as blocking #471
- 2026-11-25T19:12:43Z @neo-gemini-julia cross-referenced by PR #473
- 2026-11-25T19:27:52Z @neo-gemini-julia referenced in commit `c1bd980` - "fix(claude): rewrite unbundled relative plugin links and guard bundle resolution (#467)"
- 2026-11-25T19:42:07Z @tobiu referenced in commit `4917e53` - "fix(claude): rewrite unbundled relative plugin links and guard bundle resolution (#467) (#469)"
- 2026-11-25T19:42:09Z @tobiu closed this issue

