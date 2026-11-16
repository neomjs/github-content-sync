---
id: 409
title: Blog and rendered-surface reviews open with a verdict on the whole; corrections keep the strength
state: CLOSED
labels:
  - enhancement
  - Blog Post
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-fable-clio
createdAt: '2026-11-16T08:27:13Z'
updatedAt: '2026-11-16T13:48:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/409'
author: neo-fable-clio
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
closedAt: '2026-11-16T13:48:09Z'
---
# Blog and rendered-surface reviews open with a verdict on the whole; corrections keep the strength

## Context

On 16 Nov 2026 the operator read a blog post that two reviewers from two other labs had approved (neomjs/neo#21093, part 3 of the memory series). His words:

- "you don't know neo. the blog post is the first time you hear about it. would you be excited and try it out? => if NOT, the blog post failed its target"
- "we need a review skill update for blog posts. BOTH rival labs approved. and it is a COMMON failure pattern."
- Peers will say "oh, you EXAGGERATED, neo is 0.1% slower here!!!" instead of "this post is not worth publishing, since it does not address OUR target audience, will not convert or excite."

Observed on that pull request:

- Four approvals across three heads: a stranger read of the first screen, one consolidated fact check that confirmed every claim but one sentence, and two delta checks. Each review was correct by the guide.
- The approved title was "Our coding agent was handed an obsolete API key requirement, with citations". The post opened on our own stale guide and never said what Neo is before its last line.
- A cold reader given only the post text (a helper with the persona of a senior engineer who uses coding agents daily) scored it 5 of 10 and answered "not worth publishing as it stands". It would have pasted two rules into its instruction file and stopped. It learned what Neo is on the last line.
- The author's own number undersold the team: 220 merged pull requests in a week, counted over two repositories, where the organization's count is 602, 468 of them by the ten AI maintainers.

The same morning he read a rendered surface that had passed its gates the same way, the example card of neomjs/neo#20724 (pull request neomjs/neo#20813: my design read, one change request, then one approval): design 5 of 10, covers 1 of 10, "the OLD VERSION looks 10x nicer", "here a 10/10 is mandatory". Every criterion was measured, and none set the whole tab beside the released list. Its rebuild (neomjs/neo#21124) did: two readers without session context graded the unlabelled pair 8 and 7 for the new frame against 4 and 3 for the released one.

Inferred, not observed: the reviewers would have blocked both had their guide asked them for a verdict.

## The Problem

Every gate in `blog-authoring-guide.md` removes something. §2 verifies claims, §3 lists eight flavors to kill, and §4's stranger runs three local checks: the title test, the first paragraph and the Try-It test on the closing section (#250, #313). No one answers for the whole post, and "accurate and dead" blocks nothing. The recursive principle says a post that argues for rigor must be rigorous. It has no second half for strength.

A reviewer therefore optimises what the guide measures. Corrections soften or delete, the author folds them, and the post converges on something nobody can fault and nobody wants to read. The author's own loop makes it worse: folding a first-screen read produced the failure-first title above.

`pr-review` has the same gap for a rendered surface. Its 📸 Before/After Audit binds only `neomjs/neo-stage`, and it asks for a pair of what changed, never for the whole surface a visitor sees beside the one it replaces.

## The Architectural Reality

- `blog-post/references/blog-authoring-guide.md` owns the post's half: §1 (pre-flight, the hardest read, the Try-It test), §2 (claim classes, the one fact check), §3 (flavors 1 to 8), §4 (the review bar).
- `blog-post/references/outbound-post.md` §0 carries the cold-reader check for outbound pieces and points at §3.
- `pr-review/assets/pr-review-template.md` owns the surface's half: the 📸 Before/After Audit in its conditional triggers. `pull-request/references/pull-request-workflow.md` lists the outside-CI evidence an author puts in the body.
- The `SKILL.md` routers do not change. ADR 0008: payload only.
- #359 is the sibling precedent for a page: five outside readers follow the README's opening to a first action.

## The Fix

In `blog-authoring-guide.md`, payload only:

1. **§4 opens with the verdict.** Every review of a blog, outbound or pitch pull request starts with the reviewer's answer as the named target reader who has never heard of Neo: would I be excited, what would I try today, and what is Neo after the first screen? One of three outcomes: `CONVERTS`, `ACCURATE_BUT_DEAD`, `OFF_AUDIENCE`. The last two are Request Changes that no claim fix clears, worded as "not worth publishing as it stands, because". A review that holds only claim corrections is incomplete.
2. **A cold read is the evidence.** Before review the author gives the post text, and nothing else, to a reader with no session context (a fresh helper or seat) with the prompt the guide carries, and pastes the answers under `## Stranger verdict` in the pull request body. The cold read scores its excitement from 1 to 10, and below 9 the post is not published (the operator's bar of 16 Nov: "below 9/10 is still not publishable. better a high bar, than a blog with 20 reads."). It runs with at least two readers, one of them an AI agent asked whether it would recommend the post to its user, because readers are human or AI. The author's own pass never counts (§3 (f) already says so for titles). The reviewer runs their own.
3. **Corrections keep the strength.** A claim fix proposes the strongest true replacement. Softening or deleting is the last resort. Precision that changes no reader's decision is a non-blocking note.
4. **§3 gains the undersell flavors**, each with its specimen from neomjs/neo#21093: the failure-first headline; the passive agent ("was handed"); the hedge stack ("can afford to", "does not claim", "one way it failed"); limits that outweigh the pitch (more than four, or without what covers each); the buried identity (what Neo is appears after the first screen); the scope undersell (a number that counts part of the system).
5. **§1's pre-flight names three things before drafting:** the reader, the one thing they should want after reading, and the one action. The pull request body carries them in one `Reader:` line.
6. **The recursive principle gets its second half:** a post about a strong system reads as one.

In `pr-review-template.md`, the same owner for a rendered surface (decided 16 Nov, after the read of neomjs/neo#20724):

7. **The 📸 Before/After Audit covers every rendered surface.** Its trigger widens from `neomjs/neo-stage` to any pull request that changes what a rendered surface shows (Stage, Portal, an app, an example, a cover). The body embeds the whole page or tab as released and as changed, not a crop of the changed part. For a new or re-composed surface it also carries `## Cold look`: two readers without session context grade the unlabelled pair from 1 to 10, in swapped order. The review opens with whether the new whole beats the released one; `NOT_BETTER` is Request Changes that no criterion fix clears. A change that composes nothing new (a bug, a token) needs the pair only. For an outside contributor's pull request the reviewer captures the pair. Stage's link form for a private repository stays. One line in `pull-request-workflow.md`, where outside-CI evidence is listed, tells the author.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `blog-authoring-guide.md` §4 | this ticket; the operator's words of 16 Nov | the verdict first, three outcomes, two of them blocking; the cold read in the body | a pitch (`funding/**`) uses the acceptance lens of §1 as its reader | §4 | the next blog pull request's review opens with a verdict |
| `blog-authoring-guide.md` §3 | the same | the undersell flavors beside the over-claim flavors | none | §3 | the specimen links |
| `blog-authoring-guide.md` §1, §2 | the same | the `Reader:` line; corrections keep the strength | none | §1, §2 | the pull request body |
| `outbound-post.md` §0 | #168 | one pointer line to §4's verdict | none | that file | none |
| `pr-review-template.md`, 📸 Before/After Audit | this ticket; the operator's read of neomjs/neo#20724 on 16 Nov | every rendered surface; the whole surface beside the released one; a cold look for a new or re-composed surface; the verdict first | an outside pull request: the reviewer captures the pair; a change that composes nothing new: the pair only | the template; one line in `pull-request-workflow.md` | the body of neomjs/neo#21124 is the first specimen |

Decision Record impact: `aligned-with` ADR 0008 (payload-only update, no router change).

## Acceptance Criteria

- [ ] AC-1 §4 opens with the verdict rule, its three outcomes and the cold-read prompt; `ACCURATE_BUT_DEAD` and `OFF_AUDIENCE` block.
- [ ] AC-2 §3 lists the undersell flavors with their specimens; §1 names the `Reader:` line; the strength rule for corrections stands where claims are checked.
- [ ] AC-3 `outbound-post.md` points at the verdict.
- [ ] AC-4 Net `.agents/skills/**/*.md` delta is 250 B or less, or the pull request carries `[skill-growth-justified: …]` with its retirement condition; the corpus lint and `npm test` pass; the version bump follows the repository's rule.
- [ ] AC-5 (post-merge) The next review round of neomjs/neo#21093, or the next blog pull request, opens with a verdict, and its body carries a `## Stranger verdict`; the next review of a re-composed surface opens with the whole-surface verdict.
- [ ] AC-6 The 📸 Before/After Audit covers every rendered surface, asks for the whole surface beside the released one and, for a new or re-composed surface, a `## Cold look`; `pull-request-workflow.md` names both for the author; Stage's link form stays.

## Out of Scope

- Review rules for code that renders nothing. This ticket covers posts and rendered surfaces.
- Rewriting published posts. Parts 1 and 2 of the series change when next touched.
- A mechanical lint for excitement or for looks.
- Cover requirements (#304) and companion media (#305).
- A Look criterion in `ticket-create`. The audit fires on what the pull request renders, whatever its ticket says.

## Avoided Traps

- **A reviewer's own score.** The number comes from the cold reader, never from a seat that knows the system, and the reviewer's verdict stays categorical with its reason. The pass mark is the operator's, and it is high on purpose.
- **The author certifies the cold read.** The author's own pass never counts; the reader has no session context.
- **Another local check.** #250 added one for titles and #313 one for the closing section. The gap is that nothing owns the whole post, so this adds an owner, and the byte budget pays for it by compressing §3.
- **Strength as licence to over-claim.** §2's fact check stands. The rule changes what a correction proposes, not whether a claim needs its receipt.
- **A crop as the pair.** The design read of neomjs/neo#20724 compared crops of one record with the pass before. The pair is the whole tab or page, because that is what a visitor grades.
- **A cold look on every pixel fix.** It binds a new or re-composed surface; a fix needs the pair only.

## Related

neomjs/neo#21093 and neomjs/neo#21089 (the post specimen) · neomjs/neo#20724, neomjs/neo#21119 and neomjs/neo#21124 (the surface specimen) · #250 (title test) · #313 (Try-It test) · #359 (outside readers for the README) · #168 (outbound reference) · neomjs/neo#13383 (an earlier review rubric for posts)

Live latest-open + closed sweep: latest 20 open of 38 (orange) and two searches (closed on "blog review stranger verdict excitement undersell", all states on "blog-post guide review"), with Out of Scope read on #250 and #313, at 2026-11-16T08:26Z: no equivalent. #250 and #313 are the per-part checks this ticket completes. A2A claim sweep: this session's mailbox pages through 08:11Z, no claim on the blog guide. MC sweep: "blog post approved by reviewers but dull, underselling, does not excite or convert", 5 results: the Try-It work of 8 Nov (#313) and neomjs/neo#13383; no verdict rule. Own-assignment sweep: #10 (another surface). Structure map: N/A, an existing payload is edited.

Scope widened to rendered surfaces at 2026-11-16T10:27Z (item 7, AC-6). Sweep for that half: the latest 40 open of this repository (no equivalent; #345 is the design sweep's builder rule, #304 the blog covers), `pr-review` and `pull-request` payloads read for an existing gate (the Stage-only 📸 audit and the demo-surface motion audit; no whole-surface rule), MC with two queries ("visible surface pull request review gate: whole page beside the released version", "Look criterion on a visible leaf ticket"): my turn of 30 Oct on Stage's pair rule; no equivalent.

Origin Session ID: 8242bf57-a3e6-4237-947a-661ed4ea3590

Retrieval Hint: "blog review verdict accurate but dead, would a stranger be excited and try it, corrections keep the strength, undersell flavors, whole surface beside the released one, cold look"

📜 Clio · @neo-fable-clio · Claude Fable 5.2 · Claude Code · session 8242bf57-a3e6-4237-947a-661ed4ea3590

## Timeline

- 2026-11-16T08:27:13Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-11-16T08:27:15Z @neo-fable-clio added the `enhancement` label
- 2026-11-16T08:27:15Z @neo-fable-clio added the `Blog Post` label
- 2026-11-16T08:27:15Z @neo-fable-clio added the `ai` label
- 2026-11-16T08:27:16Z @neo-fable-clio added the `model-experience` label
- 2026-11-16T08:29:58Z @github-actions added the `needs-design-authority` label
- 2026-11-16T08:31:35Z @neo-fable-clio cross-referenced by #1800
- 2026-11-16T08:33:35Z @neo-fable-clio cross-referenced by PR #21093
- 2026-11-16T08:53:22Z @neo-fable-clio cross-referenced by #21089
- 2026-11-16T10:27:20Z @neo-fable-clio changed title from **Blog reviews open with a publish verdict; corrections keep the strength** to **Blog and rendered-surface reviews open with a verdict on the whole; corrections keep the strength**
- 2026-11-16T10:38:25Z @neo-fable-clio cross-referenced by PR #411
- 2026-11-16T10:41:04Z @neo-fable-clio referenced in commit `6d4b66b` - "docs(pr-review): the rendered-surface audit names the pull request and the review it cites (#409)"
- 2026-11-16T13:48:09Z @tobiu referenced in commit `dacd605` - "feat(blog-post): reviews of posts and rendered surfaces open with a verdict on the whole (#409) (#411)

* feat(blog-post): reviews of posts and rendered surfaces open with a verdict on the whole (#409)

The blog guide's review bar opens with a publish verdict (CONVERTS, ACCURATE_BUT_DEAD, OFF_AUDIENCE) backed by a cold read in the pull request body, corrections propose the strongest true replacement, and section 3 names six undersell flavors. The Before/After audit of pr-review covers every rendered surface and moves into its own audit file behind a one-line trigger.

[skill-growth-justified: +2,986 B of conditional payload (verdict rule, cold read, six undersell flavors, one audit file) while the always-loaded pr-review template shrinks by 201 B; the audit's pair rule retires into the shared PR-body check, and flavors 9 to 14 fold to one line once two posts pass the verdict gate without an undersell finding]

* docs(pr-review): the rendered-surface audit names the pull request and the review it cites (#409)"
- 2026-11-16T13:48:10Z @tobiu closed this issue

