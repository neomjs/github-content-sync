---
id: 453
title: Plugin bundles point Claude at fetchable skill files outside the bundle
state: CLOSED
labels:
  - bug
  - ai
  - build
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-24T23:08:39Z'
updatedAt: '2026-11-24T23:48:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/453'
author: neo-opus-grace
commentsCount: 0
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
  - '[x] 455 Plugin bundles keep fetchable links to other neomjs repositories'' guides'
closedAt: '2026-11-24T23:48:25Z'
---
# Plugin bundles point Claude at fetchable skill files outside the bundle

## Context

The Anthropic Software Directory Policy (support.claude.com/en/articles/13145358, §2F) says a plugin "must not direct Claude to dynamically pull behavioral instructions from external sources". `scripts/generate-claude-plugin.mjs` rewrites every `.agents/skills/<skill>/…` reference to a skill outside the bundle as `https://github.com/neomjs/neo-agent-skills/blob/dev/.agents/skills/<skill>/…`. Those target files are behavioral instructions, and the link invites Claude to fetch and follow them at run time.

Measured on dev (`9039405`):
- `plugins/neo` carries 1 such link. `skills/context-recovery/references/context-recovery-workflow.md:112` points at `pr-review`'s identity-claim audit.
- The bundles proposed under #446 have 1–3 distinct unbundled targets each.

`plugins/neo` 2.6.1 is already submitted to the directory, so the current bundle is exposed too, not only future ones.

## The Problem

A directory reviewer can read the HTTPS rewrite as §2F remote instructions and hold or reject the submission. Even when the bundle is accepted, a session that follows the link runs instructions from `dev`, which may differ from the version the user installed.

## The Architectural Reality

- `rewriteSkillText()` in `scripts/generate-claude-plugin.mjs` has three branches: own skill → `${CLAUDE_SKILL_DIR}`; bundled sibling → `${CLAUDE_PLUGIN_ROOT}/skills/<sibling>`; anything else → `BLOB_ROOT`. Only the third branch fetches.
- `checkBundle()` already fails on unrewritten `.agents/skills/` paths (`UNREWROTE_REF_RE`), so the check slot exists.
- `scripts/test-claude-plugin.mjs` case 2 pins the `BLOB_ROOT` branch, and case 3 pins its presence in a projected file.
- Design authority: the directory policy above. The HTTPS branch dates from #444, before the policy was read.

## The Fix

1. Rewrite an unbundled-skill reference to the non-fetchable token `neo-agent-skills:<skill>/<rest>`. That names its source without being a URL Claude can pull. Drop `BLOB_ROOT`.
2. In `checkBundle()`, fail on any `https://github.com/neomjs/neo-agent-skills/blob/` or `raw.githubusercontent.com/neomjs/neo-agent-skills/` URL inside a bundle.
3. Update test cases 2 and 3 to the new token, add the new check's red case, regenerate `plugins/neo`, and make a patch bump.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Unbundled-skill references inside `plugins/<slug>/**` | Directory Policy §2F | Rendered as `neo-agent-skills:<skill>/<rest>`, never as a fetchable URL | None needed: a bundled sibling still resolves through `${CLAUDE_PLUGIN_ROOT}` | the generator's JSDoc | `scripts/test-claude-plugin.mjs` + `--check` |

## Acceptance Criteria

- [ ] No file under `plugins/` contains a `neo-agent-skills` blob or raw URL, and `--check` fails if one appears.
- [ ] An unbundled reference projects as `neo-agent-skills:<skill>/<rest>`, and a bundled sibling still projects as `${CLAUDE_PLUGIN_ROOT}/skills/<sibling>/…`.
- [ ] `npm run lint && npm test` exit 0, with a patch version bump.

## Out of Scope

- Bundling a skill's referenced closure: each magnet leaf under #446 decides that for its own bundle.
- Relative `../../<skill>/…` references, which do not fetch.

## Related

- Parent: #446
- Generator: #447 (PR #452); HTTPS-branch origin: #443 (PR #444)

Live latest-open + closed sweep: checked the latest 20 open issues (49 open, orange band) plus an open and closed search ("plugin fetch OR blob OR rewrite OR runtime") at 2026-11-24T23:08Z. No equivalent found; this folds under owning epic #446.
A2A in-flight claim sweep: the latest 8 all-status messages show no claim on this scope.
MC sweep: `query_raw_memories` on unbundled skill links and directory policy found this session's own records only.
Own-assignment sweep: my open skills issues (#436, #335, #289, #258) are orthogonal.
Decision Record impact: none.
Ownership: self-assigned (generator owner after #447).

Origin Session ID: 5f4e1ff7-a2e0-4e4f-8196-23f4d94bc21f
Retrieval Hint: "generate-claude-plugin BLOB_ROOT runtime-fetched behavioral instructions directory policy 2F"


## Timeline

- 2026-11-24T23:08:41Z @neo-opus-grace added the `bug` label
- 2026-11-24T23:08:42Z @neo-opus-grace added the `ai` label
- 2026-11-24T23:08:42Z @neo-opus-grace added the `build` label
- 2026-11-24T23:08:42Z @neo-opus-grace added the `model-experience` label
- 2026-11-24T23:08:49Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-24T23:08:53Z @neo-opus-grace added parent issue #446
- 2026-11-24T23:08:59Z @github-actions added the `needs-design-authority` label
- 2026-11-24T23:13:08Z @neo-opus-grace cross-referenced by PR #454
- 2026-11-24T23:46:36Z @neo-opus-grace cross-referenced by #455
- 2026-11-24T23:47:00Z @neo-opus-grace marked this issue as blocking #455
- 2026-11-24T23:48:25Z @tobiu referenced in commit `1f05919` - "fix(claude): unbundled skill references stay names, never fetchable URLs (#453) (#454)

Directory Policy §2F forbids directing Claude to pull behavioral instructions from external sources. An unbundled reference now projects as neo-agent-skills:<skill>/<rest>, and --check fails on any neo-agent-skills blob or raw URL inside a bundle. plugins/neo regenerated (1 link). 2.7.1."
- 2026-11-24T23:48:26Z @tobiu closed this issue
- 2026-11-24T23:51:07Z @neo-gemini-julia cross-referenced by #456
- 2026-11-24T23:51:13Z @neo-gemini-julia cross-referenced by #451
- 2026-11-24T23:51:15Z @neo-gemini-julia cross-referenced by #449
- 2026-11-24T23:51:19Z @neo-gemini-julia cross-referenced by #448
- 2026-11-24T23:51:22Z @neo-gemini-julia cross-referenced by #450
- 2026-11-24T23:53:27Z @neo-opus-grace cross-referenced by PR #457
- 2026-11-25T00:51:54Z @neo-opus-grace cross-referenced by PR #458

