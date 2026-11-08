---
id: 316
title: 'outbound-post: a hand-posted reply needs a cross-family check only when it makes a new claim'
state: CLOSED
labels:
  - enhancement
  - Blog Post
  - ai
  - model-experience
assignees:
  - neo-opus-grace
createdAt: '2026-11-08T03:40:32Z'
updatedAt: '2026-11-08T04:07:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/316'
author: neo-opus-grace
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
closedAt: '2026-11-08T04:07:04Z'
---
# outbound-post: a hand-posted reply needs a cross-family check only when it makes a new claim

## Context

The operator, 2026-11-08, after a reply went up before its review came back: *"do you REALLY think outbound comments need a review?"*

The record from the two nights of part 2 replies:
- Seven signed replies went through six cross-family review rounds.
- Every verdict was OUTBOUND_APPROVED, and no review asked for a change.
- Two replies were already live when their review returned.
- The operator's own read caught the vague phrase "my last turns" before posting.
- Minutes after this ticket was filed, the check of reply 8, posted before its review came back, found a factual error. The reply said hooks hand the agent its mailbox messages at the start and end of every turn. In fact the start-of-turn hook only claims the seat, and the Stop hook wakes the session with a digest of counts and subjects, not the bodies. The author had read the hook's name in the manifest, not its code.

So the seven clean replies restated what reviewed, published pieces already said: the article, `AGENTS.md`, the README. The one that failed made a new claim about how our system works.

## The Problem

`outbound-post.md` §1.7 reads: *"A reply is a post — every `link-reply` and `reply` is a `requestPublish` request governed by the exact same boundaries, review bar, and revision digest rules as a top-level post."* §4 opens: *"Every outbound post or reply is reviewed by a peer from a different model family than the author."*

For a reply that only restates reviewed, published claims, the check duplicates a gate that already exists: the operator's read before he posts. It costs a round trip on a thread whose attention decays within hours, and on such replies it caught nothing. A reply that makes a new claim about our system is different: the author's blind spot is exactly what the cross-family read caught. As written, the rule fits the executor path, where the Fleet publishes without a human read. For the replies the operator posts himself it should scale with the claims, and while X's reply classification is unresolved, every reply goes out that way (§5.2).

## The Architectural Reality

- Design authority: §1.7 and §4 as quoted above, from D#19500 §4 and ADR 0042 §1.2/§1.4/§3.2. The cross-family attestation is built for the executor's switch states, *"auto-approve after a cross-family review attestation"* (§4.2).
- The boundaries themselves are not in question: they hold for a reply exactly as for a post.
- Payload-only, in `references/outbound-post.md`; no router change.

## The Fix

1. §1.7 keeps the rule that a reply meets every boundary. The review clause scales with the reply's claims:
   - A hand-posted reply that only restates what a reviewed, published piece says goes straight to the operator after the author's self-check.
   - A reply that makes a new claim about how our system works keeps the cross-family check, and the author reads the code behind the claim, not its name.
   - The executor path keeps its attestation.
2. §4's opening line names both cases and points to §1.7.

## Acceptance Criteria

- [ ] AC-1 §1.7 and §4 state that a hand-posted reply restating reviewed, published claims needs no cross-family pre-review, while a reply with a new claim about our system keeps it. Boundaries and receipts still apply, and the executor path keeps its attestation.
- [ ] AC-2 `npm run lint` and `npm test` pass; the version is bumped.

## Out of Scope

- Top-level posts, blog posts and pitch copy keep the cross-family review. Their review has caught real defects, for example part 2's origin paragraph (`neomjs/neo#20574`).
- The executor path and its attestation (ADR 0042).

## Avoided Traps

- Dropping the boundaries for replies. Only the pre-review changes.
- Dropping the check for every reply. Reply 8 shows that a new mechanism claim needs a second reader who opens the code.

## Related

#314 · #168 · D#19500 · ADR 0042

Live latest-open + closed sweep: the latest 20 open issues plus a closed search for *reply review outbound* at 2026-11-08T03:40Z; nothing equivalent. #168, the outbound reference's origin, is closed. A2A in-flight sweep: the latest messages carry no claim on this scope. Memory Core: my 11-07 note held that a signed reply needs one cross-family check, which is the reading this ticket narrows. Own assignments: no overlap.

Origin Session ID: 6bb2b8dc-e7ad-48a9-83ff-f78b6afcfb62

Retrieval Hint: "hand-posted reply cross-family check only for new mechanism claims; restating published claims goes straight to the operator"


## Timeline

- 2026-11-08T03:40:33Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-08T03:40:34Z @neo-opus-grace added the `enhancement` label
- 2026-11-08T03:40:34Z @neo-opus-grace added the `Blog Post` label
- 2026-11-08T03:40:35Z @neo-opus-grace added the `ai` label
- 2026-11-08T03:40:35Z @neo-opus-grace added the `model-experience` label
- 2026-11-08T03:41:40Z @neo-opus-grace changed title from **outbound-post: a reply the operator posts by hand needs no cross-family pre-review** to **outbound-post: a hand-posted reply needs a cross-family check only when it makes a new claim**
- 2026-11-08T03:43:00Z @neo-opus-grace cross-referenced by PR #317
- 2026-11-08T03:57:12Z @neo-opus-grace referenced in commit `45b7b87` - "chore(release): 0.2.0, a new review rule is a minor change, not a patch (#316)

The operator refused 0.1.99: 95 releases had all been 0.1.x patches, rule additions included. A new or changed rule is new behavior, which SemVer counts as a minor change."
- 2026-11-08T03:58:18Z @neo-opus-grace referenced in commit `39e561b` - "chore(release): 1.0.0, the package declares the contract it has kept for 95 releases (#316)

The operator's ruling: the team has used this package as a stable contract for a long time and never declared v1; 95 releases were all 0.1.x patches, rule additions included. The README gains a Versioning section: the public contract (skills and paths, bins and flags, reusable workflows) and which part a change bumps."
- 2026-11-08T04:07:04Z @tobiu referenced in commit `256b8f9` - "feat(blog-post): release 1.0.0; a hand-posted reply needs a cross-family check only when it makes a new claim (#316) (#317)

* feat(blog-post): a hand-posted reply needs a cross-family check only when it makes a new claim (#316)

Seven replies restating reviewed, published claims passed six review rounds with no change; the one reply with a new mechanism claim (the wake hooks) carried a factual error the cross-family read caught. §1.7 scales the review with the claims; §4 names both cases.

[skill-growth-justified: replies scale their review with their claims instead of a blanket gate; retires with §0 into the requestPublish payload]

* chore(release): 0.2.0, a new review rule is a minor change, not a patch (#316)

The operator refused 0.1.99: 95 releases had all been 0.1.x patches, rule additions included. A new or changed rule is new behavior, which SemVer counts as a minor change.

* chore(release): 1.0.0, the package declares the contract it has kept for 95 releases (#316)

The operator's ruling: the team has used this package as a stable contract for a long time and never declared v1; 95 releases were all 0.1.x patches, rule additions included. The README gains a Versioning section: the public contract (skills and paths, bins and flags, reusable workflows) and which part a change bumps."
- 2026-11-08T04:07:04Z @tobiu closed this issue

