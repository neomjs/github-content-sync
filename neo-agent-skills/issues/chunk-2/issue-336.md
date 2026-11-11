---
id: 336
title: The gates header says ten rules where four variants emit nine
state: CLOSED
labels:
  - bug
  - good first issue
  - ai
  - model-experience
assignees: []
createdAt: '2026-11-10T20:38:45Z'
updatedAt: '2026-11-11T21:05:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/336'
author: neo-opus-vega
commentsCount: 1
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
closedAt: '2026-11-11T21:05:22Z'
---
# The gates header says ten rules where four variants emit nine

## Context

`agents-md/sections/0400-critical-gates.md` line 9 opens the gate list with "These ten rules have **no conditional exceptions**". Gate 10 is its own section, `0410-critical-gate-10.md`, declared `repos: neo-agent-brain`.

Measured at `a28885c` with `node scripts/generate-agents-md.mjs --repo <r>`, counting the numbered items between `## §critical_gates` and `## §pre_commit_gates`:

| Variant | Gates emitted |
|---|---|
| `neo` | 9 |
| `neo-agent-institution` | 9 |
| `neo-agent-skills` | 9 |
| `devindex` | 9 |
| `neo-agent-brain`, or any set that contains it | 10 |

`neomjs/devindex` shows it today: its tracked `AGENTS.md` says ten and lists nine.

## The Problem

An agent told "ten rules" with nine in front of it either looks for a tenth that its variant never had, or learns that the count in a gate header is loose. The header is in the turn-loaded file of every maintainer session.

## The Architectural Reality

- The numeral is prose in section 0400. The generator numbers each gate from its `listNumber` and does not know the header.
- No other file counts the gates in prose (`grep -rn -i "ten rules\|ten gates\|nine rules" agents-md .agents/skills scripts` finds this one line).
- Design authority: "A numbered item renders with its DECLARED number, never its position. Dropping one rule from a repository must not renumber the rest" (`scripts/generate-agents-md.mjs`, the comment above `rendered`). The generator expects a variant to drop a numbered rule, so a fixed count cannot be right for every variant.

## The Fix

Drop the numeral: "These rules have **no conditional exceptions** …". One word in `0400-critical-gates.md`; no generator change.

## Acceptance Criteria

- [ ] The opening sentence of `§critical_gates` names no count.
- [ ] `node scripts/test-generate-agents-md.mjs` passes.
- [ ] The PR body shows that sentence for `--repo neo` and for `--repo neo-agent-brain`.

## Out of Scope

- Which repositories gate 10 is declared for.
- Regenerating a consumer's `AGENTS.md` (#335).

## Avoided Traps

- **Teaching the generator to substitute the count.** It adds a template mechanism for one word.

## Related

- #335
- #54

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: checked latest 20 open, created-descending, and closed search ("ten rules"; gate 10; critical gates) at 2026-11-10T20:36Z; no equivalent or prior exclusion found. 33 open (orange): no owning ticket or epic for this.
A2A in-flight sweep: latest 30 messages, all read states, at 20:36Z; no claim on this scope.
Memory Core rationale sweep: "tracked AGENTS.md stale after skills pin bump"; nothing on the header.
Own-assignment sweep: `#5`, `#6`, `#302`, `#334`; none on this surface.
Structure map: N/A. One word changes in an existing section.

Origin Session ID: b1fcc0fd-3006-429b-bb67-69b4591de962

## Timeline

- 2026-11-10T20:38:46Z @neo-opus-vega added the `bug` label
- 2026-11-10T20:38:47Z @neo-opus-vega added the `good first issue` label
- 2026-11-10T20:38:47Z @neo-opus-vega added the `ai` label
- 2026-11-10T20:38:47Z @neo-opus-vega added the `model-experience` label
- 2026-11-10T21:58:58Z @shounakjoshi88-a11y cross-referenced by PR #338
### @shounakjoshi88-a11y - 2026-11-10T21:59:16Z

Taking this one - the pull request is #338.

**Approach.** The numeral was prose in the section, so the fix is one word: the header now says "These rules" instead of "These ten rules". The generator is unchanged and still numbers each gate from its declared number, so a variant that drops a rule keeps the numbers it declares.

**Test plan.** The generated file is the evidence:

```bash
node scripts/generate-agents-md.mjs --repo neo
node scripts/generate-agents-md.mjs --repo neo-agent-brain
node scripts/test-generate-agents-md.mjs
npm run lint
```

The pull request body shows the emitted sentence for both variants and the numbered gates counted from the generated file: nine for `neo`, ten for `neo-agent-brain`.


- 2026-11-11T02:33:55Z @neo-opus-grace cross-referenced by #340
- 2026-11-11T03:01:01Z @neo-opus-grace cross-referenced by #342
- 2026-11-11T09:45:38Z @shounakjoshi88-a11y referenced in commit `23b8b64` - "chore(release): bump the package version for the #336 merge (#336)

`skill-corpus`'s Version bump step requires every non-Dependabot merge to
dev to publish a new version, so this carries the bump this PR will need
when it merges."
- 2026-11-11T10:10:30Z @github-actions cross-referenced by #47
- 2026-11-11T21:05:22Z @tobiu referenced in commit `c6f0d73` - "docs(critical-gates): drop the rule count from the gate header (#338)

* docs(critical-gates): drop the rule count from the gate header

The gate header said "These ten rules", but four of the five variants emit
nine gates. Only the variants that include neo-agent-brain, which declares
the tenth gate, emit ten.

An agent told "ten rules" with nine in front of it either looks for a tenth
that its variant never had, or learns that a count in a gate header is
loose. The header is in the turn-loaded file of every maintainer session.

The numeral was prose in the section, so the fix is one word: the header
now says "These rules". The generator is unchanged and still numbers each
gate from its declared number, so a variant that drops a rule keeps the
numbers it declares.

Signed-off-by: Shounak Joshi <shounakjoshi88@gmail.com>

* chore(release): bump the package version for the #336 merge (#336)

`skill-corpus`'s Version bump step requires every non-Dependabot merge to
dev to publish a new version, so this carries the bump this PR will need
when it merges.

---------

Signed-off-by: Shounak Joshi <shounakjoshi88@gmail.com>"
- 2026-11-11T21:05:23Z @tobiu closed this issue

