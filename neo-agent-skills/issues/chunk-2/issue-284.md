---
id: 284
title: A first-time contributor's review gives one link where pointers into Neo are owed
state: OPEN
labels:
  - enhancement
  - contributor-experience
  - ai
  - needs-design-authority
assignees:
  - neo-opus-vega
createdAt: '2026-11-05T21:49:45Z'
updatedAt: '2026-11-05T21:49:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/284'
author: neo-opus-vega
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
# A first-time contributor's review gives one link where pointers into Neo are owed

## Context

On 2026-11-05 the operator merged neo #20386, a first-time contributor's PR, and said: *"we should always give new guys some pointers about neo."* #97's own source quote, on 2026-09-19, asked for the same thing: *"giving some hints and input on the neo ecosystem, and definitely saying 'thank you!'"*.

#97 shipped the micro form's Opening slot as **thanks, then ONE entry point**. The full form's `Peer-Review Opening` is generic encouragement with no pointer at all.

## The Problem

The specimen is my #20386 micro-review (5792577246). It followed the slot exactly: thanks for one specific judgement, one setup link, one Introduction link. The operator found that insufficient. I then posted the orientation separately (6670413257): what Neo is, the part of Neo the PR touched, how to run it, and what to pick up next.

The instrument produced the thin opening. "ONE entry point" reads as a ceiling, so a reviewer who fills the slot faithfully stops at one link. A first-time contributor who lands one PR and leaves has learnt nothing about the project, and the first merge is the cheapest moment to orient them.

## The Architectural Reality

- `.agents/skills/pr-review/assets/pr-review-micro-review-template.md`, the `**Opening**` slot: "thanks for the specific judgement this PR shows, then ONE entry point", with a long example sentence.
- `.agents/skills/pr-review/assets/pr-review-template.md`: `**Peer-Review Opening:**`, generic encouragement only.
- `references/pr-review-guide.md` §1 already points the register rule at the opening slot (#97), so the slot's wording is where the behaviour is decided.
- Both assets fall under the corpus byte budgets that `scripts/lint-skill-corpus.mjs` enforces in CI.

## The Fix

1. **Micro form:** the Opening asks for thanks for one specific judgement, then a few pointers into Neo, each linked at its canonical owner: what Neo is, the part of Neo this PR touched, how to run it, and what to pick up next. The rule that setup is linked at the Engine's `CONTRIBUTING.md`, never as a command pair, stays, and so does the fact that `neo-agent-skills` is on npm. The example gets shorter, not longer.
2. **Full form:** the `Peer-Review Opening` carries the same rule for external and first-time contributors.
3. Bump the package version (published-package rule).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| micro-review template, `Opening` slot | #97 | thanks + four kinds of pointers into Neo, at canonical owners | omitted for maintainer PRs, as today | `pr-review-guide.md` §1 | the #20386 specimen |
| full template, `Peer-Review Opening` | existing | external and first-time contributors get the same pointers | maintainer PRs unchanged | self | — |
| `package.json` `version` | npm publish contract | bumped | none | n/a | — |

**Accretion disposition.** Net bytes about flat: the example sentence shrinks as the rule grows. Sunset as #97 set it: retire the slot when an external-contributor review path exists as its own skill, or when a body lint enforces the opening.

## Acceptance Criteria

- [ ] The micro template's Opening asks for pointers covering what Neo is, the part the PR touched, how to run it, and what's next, each linked at its canonical owner; the setup link and the npm fact stay.
- [ ] The full template's `Peer-Review Opening` states the same rule for external and first-time contributors.
- [ ] `npm run lint` passes in this repository, byte budgets included.
- [ ] The package version is bumped.

## Out of Scope

Review rigour, the claim-responder bot's text, and the Engine's `CONTRIBUTING.md`.

## Related

#97 (the slot this amends), neo #20386 (the specimen).

Live latest-open + closed sweep: checked the latest 20 open issues and a closed search (`contributor opening review template`) at 2026-11-05T21:48Z; #97 is the predecessor, read in full; no equivalent or exclusion found.
MC sweep: `new contributor merged PR got no orientation about Neo, review opening too thin`, 5 results, no prior decision beyond #97.
Own-assignment sweep: 2 open (#5, #6), none overlapping.

Origin Session ID: 63900c99-2a60-4254-99d3-2e5fd1893d70

Retrieval Hint: "first-time contributor review opening pointers into Neo"


## Timeline

- 2026-11-05T21:49:46Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-05T21:49:47Z @neo-opus-vega added the `enhancement` label
- 2026-11-05T21:49:47Z @neo-opus-vega added the `contributor-experience` label
- 2026-11-05T21:49:47Z @neo-opus-vega added the `ai` label
- 2026-11-05T21:49:56Z @github-actions added the `needs-design-authority` label

