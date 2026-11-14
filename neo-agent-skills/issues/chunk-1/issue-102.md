---
id: 102
title: The cross-family test has no answer for a disclosed external AI author
state: OPEN
labels:
  - enhancement
  - contributor-experience
  - ai
  - architecture
assignees:
  - neo-opus-ada
createdAt: '2026-09-21T14:55:52Z'
updatedAt: '2026-11-14T14:08:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/102'
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
# The cross-family test has no answer for a disclosed external AI author

> **Resolved 2026-11-14 by operator ruling** (to `@neo-opus-ada`): *"Let us assume that currently all external PRs fully get created by LLMs (even if sometimes contributors claim otherwise). In this case, a claude PR should get reviewed by a Gemini or GPT peer. If no model is named, anyone can review it. If FM points a peer at a same-lab PR, it should get offered to a peer from a different lab."* This decides the open question below, against the recommendation this body first made (see Avoided Traps). neomjs/neo-agent-brain#1702 gives the Fleet rotation and the merge-readiness verdict the same rule; this ticket is the seat-side half.

## Context

Opened 2026-09-21 on a live instance. @wofiporia's [neomjs/neo#19032](https://github.com/neomjs/neo/pull/19032) disclosed *"prepared by an AI coding agent (Claude / Claude Code)"*, and a Claude seat held its review. The mandate gave no answer for an outside author.

Measured 2026-11-14 over the 64 outside PRs opened across `neomjs` from 11-01 to 11-14: 27 name a model family (19 Claude, 6 GPT, 2 DeepSeek) and 37 name none. 8 of the 17 merged Claude-named PRs carry only a Claude approval. The method and the list are on neomjs/neo-agent-brain#1702.

## The Problem

`pull-request/references/cross-family-mandate.md` is a difference test on `modelFamily`, and `pull-request-workflow.md` resolves the author's family *"from the §5 Social Name, with `author.login` fallback"*. An outside author has neither, so the test returns no answer. That is not `unknown`, which is a recorded value for an assessed seat; it is no answer at all. With no rule to check on arrival, seats reviewed and approved same-lab outside PRs, and the census shows how often.

## The Architectural Reality

- `.agents/skills` in a consumer repo is a symlink into `node_modules/neo-agent-skills/.agents/skills`, so the mandate governs every repo at once.
- The check has to fire when a review request or a Fleet triage wake arrives, not only when a seat is requested. It must be answerable from `gh pr view N --json body,commits`, with no roster lookup.

Design authority: `cross-family-mandate.md`'s difference test and `pull-request-workflow.md`'s author-family line, quoted above. The operator ruling extends them to outside authors.

## The Fix

In `pull-request/references/cross-family-mandate.md`, in the file's difference-test idiom:

1. An outside author's family is the family its PR names. A name counts in a disclosure line in the body (`Authored by …`, `Generated with …`, `Prepared with …`, an `AI assistance` line) or in a `Co-Authored-By` trailer on any commit. When a PR names several families, all of them count. A harness alone (Cursor, Copilot, OpenCode) names none, and a path or topic mention is not a disclosure.
2. If the PR names a family, the approval needs a seat of a family it does not name. A seat of a named family that is handed the PR offers it to a seat of another lab.
3. If the PR names none, any seat may review and approve.
4. `pull-request-workflow.md`'s author-family line points at this clause for outside authors. `pr-review`'s arrival check reads the body and the commit trailers.

## Acceptance Criteria

- [ ] AC-1: `cross-family-mandate.md` states the outside-author rule in the difference-test idiom, without a family list: a named family needs an approver from another family; nothing named means any seat.
- [ ] AC-2: It defines a disclosure: body disclosure lines and `Co-Authored-By` trailers on any commit. Harness-only names and path or topic mentions do not count.
- [ ] AC-3: It states the reviewer path: a same-lab seat that receives such a PR hands it to a seat of another lab, so no PR waits on a seat that cannot approve it.
- [ ] AC-4: `pr-review`'s arrival check covers outside PRs, answerable from `gh pr view N --json body,commits` with no roster lookup.
- [ ] AC-5: Net loaded bytes are accounted per the accretion rule. The clause replaces the undefined case; it does not add a parallel rule.

## Out of Scope

- The Fleet rotation and the merge-readiness verdict: neomjs/neo-agent-brain#1702.
- Seat-to-seat review: settled, not reopened.
- Detecting undisclosed AI authorship: permanently out of scope. An outside PR that names nothing is open to any seat.

## Avoided Traps

- **The first recommendation here** treated outside contributions as family-less. It argued that the correlation the mandate guards comes from our shared substrate, and that a binding rule punishes disclosure. The ruling rejects it: outside PRs are presumed model-written, and 8 of 17 merged Claude-named PRs got a Claude-only approval while this question stayed open.
- **Reading absence as `unknown`:** that turns "never assessed" into "assessed and differing" (caught by @neo-opus-grace, 2026-09-21).
- **Bare name matching:** `.gemini/` paths and topic sentences name no author (neo `#20789`, `#20798`).
- **Enumerating families:** the mandate stays a difference test.

## Decision Record impact

`none`: clarifies an existing rule's scope rather than amending an ADR.

## Related

- neomjs/neo-agent-brain#1702: the Fleet and merge-readiness half of the same ruling.
- [neomjs/neo#19032](https://github.com/neomjs/neo/pull/19032): the first instance.
- `#85`: the same shape for a different rule, authority living only in agent memory.

## Sweeps

- Live latest-open sweep: 20 open issues on this repo at 2026-09-21T14:54:44Z; no equivalent. `#85` is adjacent and a different subject.
- A2A in-flight claim sweep at 14:54Z: no `[lane-claim]` overlaps the cross-family mandate. @neo-opus-grace explicitly declined to claim it.
- Own-assignment sweep: `#76`, `#66`, `#63` are mine here; none touch review-family eligibility.

Origin Session ID: c54728f6-de9d-46a5-921f-aef7e79b91c8

Retrieval Hint: `query_raw_memories("external contributor discloses Claude authorship cross-family reviewer eligibility")`

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5 · Claude Code. Rewritten 2026-11-14 (Claude Opus 5.5) to carry the operator's ruling.


## Timeline

- 2026-09-21T14:55:52Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-21T14:55:53Z @neo-opus-ada added the `enhancement` label
- 2026-09-21T14:55:53Z @neo-opus-ada added the `contributor-experience` label
- 2026-09-21T14:55:54Z @neo-opus-ada added the `ai` label
- 2026-09-21T14:55:54Z @neo-opus-ada added the `architecture` label
- 2026-09-21T16:42:36Z @neo-opus-grace cross-referenced by PR #19032
- 2026-09-23T11:42:05Z @neo-opus-ada cross-referenced by #427
- 2026-09-23T13:47:39Z @neo-opus-ada cross-referenced by #109
- 2026-10-10T22:31:14Z @neo-opus-ada cross-referenced by #166
- 2026-10-19T12:52:54Z @neo-opus-ada cross-referenced by #194
- 2026-10-21T22:52:39Z @neo-gpt-emmy cross-referenced by #199
- 2026-10-29T00:10:58Z @neo-opus-ada cross-referenced by #234
- 2026-11-07T06:58:18Z @neo-gpt-euclid cross-referenced by #310
- 2026-11-12T01:29:21Z @neo-opus-ada cross-referenced by #366
- 2026-11-12T01:40:42Z @neo-opus-ada cross-referenced by #367

