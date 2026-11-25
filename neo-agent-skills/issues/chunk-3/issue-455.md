---
id: 455
title: Plugin bundles keep fetchable links to other neomjs repositories' guides
state: CLOSED
labels:
  - bug
  - ai
  - build
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-24T23:46:35Z'
updatedAt: '2026-11-25T00:53:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/455'
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
blockedBy:
  - '[x] 453 Plugin bundles point Claude at fetchable skill files outside the bundle'
blocking: []
closedAt: '2026-11-25T00:53:26Z'
---
# Plugin bundles keep fetchable links to other neomjs repositories' guides

## Context

#453 / PR #454 stops plugin bundles from carrying fetchable URLs to skill files. The same Directory Policy rule (§2F: a plugin "must not direct Claude to dynamically pull behavioral instructions from external sources") also covers absolute links to other neomjs repositories' process guides, for example `https://github.com/neomjs/neo-agent-brain/blob/dev/learn/agentos/process/evidence-ladder.md`. The generator passes those links through unchanged. Julia raised this in her #454 review.

Measured on `dev` at 2026-11-24T23:46Z: 33 cross-repo `blob` / `raw` links across 10 skills the #446 magnets would bundle.

| Skill | Links |
|---|---|
| `pr-review` | 9 |
| `pull-request` | 4 |
| `epic-review` | 4 |
| `ticket-intake` | 3 |
| `create-skill` | 3 |
| `tech-debt-radar` | 2 |
| `turn-memory-pre-flight` | 2 |
| `ticket-create` | 2 |
| `epic-resolution` | 2 |
| `ideation-sandbox` | 2 |

`plugins/neo`'s three skills have 0, so the submitted bundle is not affected.

## The Problem

Each magnet leaf would otherwise handle these links on its own, four times over, and a link that slips through puts a submission at review risk.

## The Architectural Reality

- `rewriteSkillText()` in `scripts/generate-claude-plugin.mjs` rewrites only repo-relative `.agents/skills/` paths.
- `checkBundle()`'s `FETCHABLE_REF_RE`, added in #454, matches only `neo-agent-skills` blob and raw URLs.
- Design authority: Directory Policy §2F, already cited on #453.

## The Fix

1. Project every `https://github.com/neomjs/<repo>/(blob|raw)/<ref>/<path>` and `https://raw.githubusercontent.com/neomjs/<repo>/<ref>/<path>` link in a bundled file as the non-fetchable name `neomjs/<repo>:<path>`.
2. Widen `FETCHABLE_REF_RE` to the same family, so `--check` refuses any neomjs blob or raw URL left in a bundle.
3. Add a case to `scripts/test-claude-plugin.mjs`, regenerate, and make a patch bump.

## Acceptance Criteria

- [ ] No file under `plugins/` contains a `github.com/neomjs/*/blob|raw` or `raw.githubusercontent.com/neomjs/` URL, and `--check` fails if one appears.
- [ ] A cross-repo link projects as `neomjs/<repo>:<path>`, and the source `.agents/skills/**` is unchanged (+0 B).
- [ ] `npm run lint && npm test` exit 0, with a patch version bump.

## Out of Scope

- Non-neomjs URLs such as vendor documentation.
- Bundling referenced guides into a magnet: that is each leaf's call.

## Related

- Parent: #446
- Sibling: #453 (PR #454)

Live latest-open + closed sweep: checked the latest open issues (49 open, orange band) plus an open and closed search ("cross-repo blob link plugin fetchable learn") at 2026-11-24T23:46Z. No equivalent except sibling #453, which is limited to skill-file URLs and resolved by approved PR #454; this folds under owning epic #446.
A2A in-flight claim sweep: Julia's 23:44Z DM agreed this split (MESSAGE:11495afa).
MC sweep: this session's #453 records only.
Own-assignment sweep: #453 is the sibling; the others are orthogonal.
Decision Record impact: none.
Ownership: self-assigned.

Origin Session ID: 6eed0fb6-c3e1-4860-a3b9-d692df0c8753


## Timeline

- 2026-11-24T23:46:36Z @neo-opus-grace added the `bug` label
- 2026-11-24T23:46:36Z @neo-opus-grace added the `ai` label
- 2026-11-24T23:46:36Z @neo-opus-grace added the `build` label
- 2026-11-24T23:46:37Z @neo-opus-grace added the `model-experience` label
- 2026-11-24T23:46:44Z @github-actions added the `needs-design-authority` label
- 2026-11-24T23:46:46Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-24T23:46:50Z @neo-opus-grace added parent issue #446
- 2026-11-24T23:47:00Z @neo-opus-grace marked this issue as being blocked by #453
- 2026-11-24T23:51:07Z @neo-gemini-julia cross-referenced by #456
- 2026-11-24T23:51:13Z @neo-gemini-julia cross-referenced by #451
- 2026-11-24T23:51:15Z @neo-gemini-julia cross-referenced by #449
- 2026-11-24T23:51:19Z @neo-gemini-julia cross-referenced by #448
- 2026-11-24T23:51:22Z @neo-gemini-julia cross-referenced by #450
- 2026-11-24T23:53:27Z @neo-opus-grace cross-referenced by PR #457
- 2026-11-25T00:51:54Z @neo-opus-grace cross-referenced by PR #458
- 2026-11-25T00:53:26Z @tobiu referenced in commit `e86ec85` - "fix(claude): cross-repo neomjs guide links stay names in plugin bundles (#455) (#457)

Absolute github.com/neomjs/<repo>/(blob|raw) and raw.githubusercontent.com/neomjs links now project as neomjs/<repo>:<path>; an absolute link to a skill in this repository takes the skill-aware rewrite. --check refuses any neomjs blob or raw URL in a bundle (Directory Policy §2F). 2.7.2."
- 2026-11-25T00:53:28Z @tobiu closed this issue

