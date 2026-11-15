---
id: 102
title: The cross-family test has no answer for a disclosed external AI author
state: CLOSED
labels:
  - enhancement
  - contributor-experience
  - ai
  - architecture
assignees:
  - neo-opus-ada
createdAt: '2026-09-21T14:55:52Z'
updatedAt: '2026-11-15T04:26:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/102'
author: neo-opus-ada
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
closedAt: '2026-11-15T04:26:10Z'
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
- The check has to fire when a review request or a Fleet triage wake arrives, not only when a seat is requested. It must be answerable from the PR's body and every commit message, with no roster lookup. `gh pr view N --json body,commits` returns only the first 100 commits, so a longer PR stays unread until the rest is read (Euclid's review 5908852312 on #394).

Design authority: `cross-family-mandate.md`'s difference test and `pull-request-workflow.md`'s author-family line, quoted above. The operator ruling extends them to outside authors.

## The Fix

In `pull-request/references/cross-family-mandate.md`, in the file's difference-test idiom:

1. An outside author's family is the family its PR names. A name counts in a disclosure line in the body (`Authored by …`, `Generated with …`, `Prepared with …`, an `AI assistance` line) or in a `Co-Authored-By` trailer on any commit. When a PR names several families, all of them count. A harness alone (Cursor, Copilot, OpenCode) names none, and a path or topic mention is not a disclosure.
2. If the PR names a family, the approval needs a seat of a family it does not name. A seat of a named family that is handed the PR offers it to a seat of another lab.
3. If the PR names none, any seat may review and approve.
4. `pull-request-workflow.md`'s author-family line points at this clause for outside authors. The Review-Seat Gate (`post-review-pickup/references/pre-review-intake-lane-gate.md`), which `pr-review`'s guide runs as §2 step 1 at every review start, makes a seat of a disclosed family hand the PR on.

## Acceptance Criteria

- [ ] AC-1: `cross-family-mandate.md` states the outside-author rule in the difference-test idiom, without a family list: a named family needs an approver from another family; nothing named means any seat.
- [ ] AC-2: It defines a disclosure: body disclosure lines and `Co-Authored-By` trailers on any commit. Harness-only names and path or topic mentions do not count.
- [ ] AC-3: It states the reviewer path: a same-lab seat that receives such a PR hands it to a seat of another lab, so no PR waits on a seat that cannot approve it.
- [ ] AC-4: The Review-Seat Gate, which is `pr-review`'s arrival check (guide §2 step 1), covers outside PRs, answerable from the PR's body and every commit message with no roster lookup. A read that stops before the last commit is unread, not unnamed.
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
- Intake 2026-11-15 (the drift probe fired on `pr-review/` and `pull-request-workflow.md`): neomjs/neo-agent-brain#1705 (merged 2026-11-14, `e44ef997`) shipped the Fleet and merge-readiness half. Its `ai/services/graph/disclosedModelFamilies.mjs` is the parse, which the mandate cites rather than restates. `pr-review` has no arrival check of its own; its guide's §2 step 1 runs the Review-Seat Gate, so AC-4 lands there. Prescription checked: `cross-family-mandate.md` — owns the concern; `pr-review-guide.md` — better owner: `pre-review-intake-lane-gate.md` §Review-Seat Gate.

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
- 2026-11-15T00:43:11Z @neo-opus-ada cross-referenced by PR #394
- 2026-11-15T00:44:28Z @neo-opus-ada referenced in commit `102d22e` - "chore(release): bump neo-agent-skills to 1.9.7 (#102)"
- 2026-11-15T03:10:23Z @neo-opus-ada referenced in commit `0d8024c` - "feat(pull-request): an outside PR's named model rules out its lab (#102)

cross-family-mandate.md gains an Outside authors clause. An outside
author's family is what its PR discloses: body disclosure lines and
Co-Authored-By trailers on any commit, parsed by Brain's
disclosedModelFamilies.mjs. A named family needs an approver from
another family, so a seat of that family hands the PR to an eligible
seat; a PR naming none is open to any seat.

The author-family line in pull-request-workflow.md section 6.1 and the
Review-Seat Gate, which pr-review runs at every review start, point at
the clause. Trims in the same files offset it: rationale the atlas
already holds, a dated anecdote, and a stale pointer that sent readers
to the mandate for marker vocabulary its consumers hold. Net corpus
growth is 238 bytes."
- 2026-11-15T03:10:23Z @neo-opus-ada referenced in commit `5a85768` - "chore(release): bump neo-agent-skills to 1.9.8 (#102)"
### @neo-opus-ada - 2026-11-15T03:26:35Z

**Handover (sunset, owner Ada).** PR #394 is rebased onto `dev` after #396 merged (which shipped 1.9.7). The head is `5a85768`: the feature as `0d8024c`, plus the bump to 1.9.8 (check-version-bump: "a version npm does not have yet"). The body is updated.
- **Review:** Euclid has been requested since 00:49Z, and his seat reads dark (no activity in 4h+).
- **Pickup:**
  1. If he has not engaged by 04:49Z (the 4h SLA), reroute to a live non-Claude seat with a recorded reason.
  2. Re-run the CI-green gate on `5a85768` first.
  3. If another Skills PR merges and publishes 1.9.8 before this one, re-bump past npm.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-15T03:52:58Z @neo-opus-ada referenced in commit `db9821a` - "fix(pull-request): "none named" needs every commit read (#102)

gh pr view N --json body,commits returns at most 100 commits, so a disclosure on commit 101 reads as none and a seat of the named family takes the review. The atlas now names a complete read for a longer PR and keeps an unread disclosure apart from an unnamed one (Euclid, review 5908852312)."
- 2026-11-15T03:54:16Z @neo-opus-ada referenced in commit `493aeae` - "fix(pull-request): keep the atlas within the corpus growth cap (#102)

The completeness sentence took the corpus to +423 B against the 250 B cap. It is shorter now, and the unknown-family section drops a sentence that restated the one before it."
- 2026-11-15T04:26:10Z @tobiu referenced in commit `6329b15` - "feat(pull-request): an outside PR's named model rules out its lab (#102) (#394)

* feat(pull-request): an outside PR's named model rules out its lab (#102)

cross-family-mandate.md gains an Outside authors clause. An outside
author's family is what its PR discloses: body disclosure lines and
Co-Authored-By trailers on any commit, parsed by Brain's
disclosedModelFamilies.mjs. A named family needs an approver from
another family, so a seat of that family hands the PR to an eligible
seat; a PR naming none is open to any seat.

The author-family line in pull-request-workflow.md section 6.1 and the
Review-Seat Gate, which pr-review runs at every review start, point at
the clause. Trims in the same files offset it: rationale the atlas
already holds, a dated anecdote, and a stale pointer that sent readers
to the mandate for marker vocabulary its consumers hold. Net corpus
growth is 238 bytes.

* chore(release): bump neo-agent-skills to 1.9.8 (#102)

* fix(pull-request): "none named" needs every commit read (#102)

gh pr view N --json body,commits returns at most 100 commits, so a disclosure on commit 101 reads as none and a seat of the named family takes the review. The atlas now names a complete read for a longer PR and keeps an unread disclosure apart from an unnamed one (Euclid, review 5908852312).

* fix(pull-request): keep the atlas within the corpus growth cap (#102)

The completeness sentence took the corpus to +423 B against the 250 B cap. It is shorter now, and the unknown-family section drops a sentence that restated the one before it."
- 2026-11-15T04:26:12Z @tobiu closed this issue

