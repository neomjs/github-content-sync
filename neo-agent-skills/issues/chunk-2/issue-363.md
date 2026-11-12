---
id: 363
title: The README's two skill counts go stale with the next skill
state: OPEN
labels:
  - documentation
  - enhancement
  - good first issue
  - ai
assignees: []
createdAt: '2026-11-11T23:48:50Z'
updatedAt: '2026-11-11T23:48:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/363'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 356
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
# The README's two skill counts go stale with the next skill

## Context

#360 put two counts into the README's opening: "40 skills" and "28 of the 40 skills name a tool of our MCP servers". Both were measured at 1.4.4 (#360, receipts C and D). Its review (5866849367) names the gap: nothing checks them, so the next skill makes both wrong.

## The Problem

The second number is the page's plain statement to an outside reader about what they get. A stale count misstates it, and a reader who counts the folders finds a README that disagrees with the package.

## The Architectural Reality

- `scripts/lint-skill-corpus.mjs` already reads `.agents/skills/skills.manifest.json`; `npm run lint` ends with "40 skills, coherent with the manifest".
- The first count is the manifest's number of skills.
- The second cannot be recomputed in this repository: the list of the servers' tool names lives in neomjs/neo-agent-brain. #360's receipt D gives the method and names the twelve skills without such a tool.

Design authority: "Pasting a number into prose with no SSOT link (it will rot)." (`.agents/skills/neo-identity-update/references/update-protocol.md`, Anti-patterns)

## The Fix

1. `npm run lint` fails when the "N skills" of the README's opening differs from the manifest's count, and names the file and the line.
2. The second count is either dated in the text ("counted at 1.4.4") or tied to the same check through its total ("28 of the N"), so that a new skill forces a recount. The author picks one and says why.

## Acceptance Criteria

- [ ] With a manifest of 41 skills and an unchanged README, `npm run lint` fails and names `README.md` and the line. `scripts/test-lint-skill-corpus.mjs` builds that case.
- [ ] The count of skills that name a server tool is dated in the text or tied to the same check.
- [ ] `npm run lint` and `npm test` pass on the head.

## Out of Scope

- Recomputing the 28 in CI.
- Other numbers in the README.

## Avoided Traps

- **Removing the numbers.** The opening owes an outside reader the number (#356).

## Related

#360 and its review 5866849367 · #356 · #359

Decision Record impact: none.

unowned-rationale: Ready for a contributor; maintainers provide planning and review. It needs no private tool: one check, one test case, one line of the README. Leave `version` in `package.json` as it is; #358 explains why a fork does not bump it.

Live latest-open + closed sweep: the latest 20 open by creation date (38 open in all, orange band; no ticket or epic owns the README's counts) and a search over all states (`README skills count`) at 2026-11-11T23:48Z; no equivalent or prior exclusion found. A2A in-flight claim sweep: the unread and the latest read messages at 23:46Z; no claim on this scope. MC sweep: covered by #356's sweep of 21:14Z on the README; no prior decision on its numbers. Own-assignment sweep: 3 open here (#356, #359, #191), not overlapping. Structure map: N/A, no file is placed.

Origin Session ID: 593bf3c2-8f38-4a45-a034-cabdc74fe0f1

Retrieval Hint: "README opening skill counts 40 and 28 of 40: lint guard against the manifest, dated second count"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 593bf3c2-8f38-4a45-a034-cabdc74fe0f1

## Timeline

- 2026-11-11T23:48:51Z @neo-fable-mnemosyne added the `documentation` label
- 2026-11-11T23:48:51Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-11T23:48:51Z @neo-fable-mnemosyne added the `good first issue` label
- 2026-11-11T23:48:51Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-11T23:49:03Z @neo-fable-mnemosyne added parent issue #356
- 2026-11-11T23:49:20Z @neo-fable-mnemosyne cross-referenced by PR #360
- 2026-11-12T00:44:49Z @neo-fable-mnemosyne cross-referenced by #356

