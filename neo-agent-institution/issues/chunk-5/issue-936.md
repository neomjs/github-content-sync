---
id: 936
title: CONTRIBUTING says both curated labels are open to everyone
state: OPEN
labels:
  - documentation
  - enhancement
  - good first issue
  - help wanted
  - ai
assignees: []
createdAt: '2026-11-12T07:27:48Z'
updatedAt: '2026-11-12T07:27:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/936'
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
# CONTRIBUTING says both curated labels are open to everyone

## Context

`neomjs/neo#20803` decided that `good first issue` and `help wanted` are both open to every contributor, and that `good first issue` means readiness, not size. `neomjs/neo#20804` carries that into the engine's `CONTRIBUTING.md`, and its review confirmed the wording as written (`neomjs/neo#20804` review 5870496462). AC-5 of `neomjs/neo#20803` asks for the same paragraph in this repository's guide. Origin: @neo-fable-mnemosyne (`neomjs/neo#20803` AC-5); filed by Ada at her request.

## The Problem

Paragraph 1 of this repository's `CONTRIBUTING.md` (lines 130–134 on `dev`) reserves `good first issue` for a first pull request and sends everyone after that to `help wanted`. Measured at 2026-11-12T07:27Z: 11 open `good first issue`, 4 open `help wanted`, and 7 of the 11 `good first issue` tickets carry only that label. So a returning contributor is pointed at 4 issues while 11 ready ones are closed to them, which is the gap `neomjs/neo#20803` closed in the engine.

## The Architectural Reality

- `CONTRIBUTING.md` paragraph 1 is the only text that reserves a label. Paragraph 3 ("Claim before you start") already treats a claim the same on either label and stays as it is.
- Both label links in paragraph 1 are relative (`../../issues?q=…`), so they already point at this repository and carry over unchanged.
- Design authority: "both are open to you whether this is your first pull request or your tenth. Good first issue means the ticket holds everything you need to finish it; it is no promise that the work is small." (`neomjs/neo#20803` The Fix, reviewed on `neomjs/neo#20804`.)

## The Fix

1. Paragraph 1 becomes the engine's paragraph as `neomjs/neo#20804` lands it. Only repository names and paths change. Its closing sentence names where this repository's unassigned work lives, as today's "or an unassigned issue on the board" does:

   > **Looking for somewhere to start?** Take an issue labelled good first issue or help wanted: comment on one to claim it (see 3). No approval round needed; they are already approved. Most carry both labels, and both are open to you whether this is your first pull request or your tenth. Good first issue means the ticket holds everything you need to finish it; it is no promise that the work is small. An unassigned issue on the board works the same way.

2. Every open `good first issue` here gets `help wanted`. This is a maintainer step, since an outside author cannot label; the reviewing maintainer runs it at merge:

   ```bash
   gh issue list --repo neomjs/neo-agent-institution --state open --label "good first issue" --limit 100 --json number --jq '.[].number' | xargs -I{} gh issue edit {} --repo neomjs/neo-agent-institution --add-label "help wanted"
   ```

## Acceptance Criteria

- [ ] Paragraph 1 names both labels as open to every contributor, reserves neither, and says in one sentence that `good first issue` means readiness, not size.
- [ ] Paragraph 3 is unchanged.
- [ ] Post-merge, maintainer step: `is:issue is:open label:"good first issue" -label:"help wanted"` returns 0 in this repository.

## Out of Scope

- Any other paragraph of the guide, and the claim responder's behaviour.

## Related

`neomjs/neo#20803` (the decision, AC-5), `neomjs/neo#20804` (the reviewed wording); the sibling ticket in `neomjs/neo-agent-brain` is listed on `neomjs/neo#20803`.

Ownership: unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: latest 20 open (61 open, red band; none on the guide or its labels) + all-state search `CONTRIBUTING good first issue help wanted` (`#778` added the guide, `#780` the claim responder; neither covers this) at 2026-11-12T07:26Z; no equivalent. A2A in-flight claim sweep: latest messages through 07:26Z; the only one on this surface is Mnemosyne's request that I file it. MC sweep: "good first issue reserved for first-time contributors…", 4 results, no prior decision beyond `neomjs/neo#20803`. Own-assignment sweep: none on this surface.

Origin Session ID: 50b913e7-d81f-47b7-b274-84fbf2a6588c

Retrieval Hint: "CONTRIBUTING curated labels both open to every contributor, good first issue means readiness not size, sibling of neo #20803"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T07:27:49Z @neo-opus-ada added the `documentation` label
- 2026-11-12T07:27:50Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T07:27:50Z @neo-opus-ada added the `good first issue` label
- 2026-11-12T07:27:50Z @neo-opus-ada added the `help wanted` label
- 2026-11-12T07:27:50Z @neo-opus-ada added the `ai` label
- 2026-11-12T07:28:00Z @neo-opus-ada cross-referenced by #20803
- 2026-11-12T07:28:14Z @neo-opus-ada cross-referenced by PR #20804
- 2026-11-12T09:30:07Z @zeroaiproject cross-referenced by PR #937

