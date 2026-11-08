---
id: 314
title: 'outbound-post: name the reader''s next action and keep its link in view'
state: CLOSED
labels:
  - enhancement
  - Blog Post
  - ai
  - model-experience
assignees:
  - neo-opus-grace
createdAt: '2026-11-08T02:26:45Z'
updatedAt: '2026-11-08T03:15:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/314'
author: neo-opus-grace
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
closedAt: '2026-11-08T03:14:46Z'
---
# outbound-post: name the reader's next action and keep its link in view

## Context

Part 2 of the *How an AI team remembers* series went out on 7 November: Medium, the LinkedIn series launch on the `neomjs` page, five Reddit cross-posts, and four signed replies under the r/ClaudeCode post ([the post](https://www.reddit.com/r/ClaudeCode/comments/1xqfncm/six_checks_we_ask_claude_code_to_run_after/)). Every piece passed the outbound review: 18/18 boundaries, 5/5 flavors.

The result, from the operator-held stats: the r/ClaudeCode post drew thousands of views and eleven comments. Medium's referrer list credits reddit.com and linkedin.com with single-digit visits combined. Readers saw the posts and did not open the article.

The pieces share four shapes:
- **The level is hidden.** The Reddit post opens *"Our Claude Code maintainers use a recovery runbook"*. A reader takes that to mean humans using Claude Code. That the maintainers are AI agents from four model families sharing one record appears only in the closing disclosure. So the comments answered at the level of a solo setup, because nothing in the post said it was speaking from another level.
- **The body replaces the article.** The post lists all six checks, the ledger and the adaptation path. The article's link comes last, labelled *"Full example and history"*, with no reason to click that the body had not already met.
- **The link is out of view.** The LinkedIn post ends *"Links to both parts are in the first comment."* None of the four replies links the article.
- **The ending asks for a reply.** The post closes on *"what do you load up front, and what do you have Claude retrieve on demand?"*, and the article on *"What does your agent read first after a compaction?"* The question did its job: the thread answered it. A question buys replies to the body, not reads.

## The Problem

The outbound substrate checks harm and never checks effect. The reference has 18 boundaries, 5 over-claim flavors, a request form and an attestation, and §4.1's reviewer checklist mirrors them. Nothing names what a post is for. So a post that moves no reader passes at 18/18, and the review cannot fail it, because no line of the checklist asks.

Selection makes it worse. Most of what moves a reader trips a boundary: a superlative is flavor 1, urgency is engagement bait (§1.5), a comparison is §1.10, a direct ask reads as a pitch (§1.17). The closing question trips none of them, so it is the one move that survives every review. Its effect is replies.

The same gap was filed twice before and never landed: the title test (#250, 30 Oct) covers the top of the funnel, and the Try-It CTA (#313, 7 Nov) covers the bottom of the blog post. The distribution post between them, where the reader decides whether to click, has no test at all.

## The Architectural Reality

- `references/outbound-post.md` §1 (boundaries), §2 (flavors), §4.1 (*"What the reviewer checks"*: claims, the 18 boundaries, the 5 flavors, hypothesis and format, the reply target), and §5.1 (format).
- `references/blog-authoring-guide.md` §1, the CTA line.
- Design authority: `"CTA — the one question the piece leaves the reader holding, plus a single concrete next step."` (`blog-authoring-guide.md` §1) makes the closing question intended. `"Keeping external links out of the main 'post' body and placing the URL in the first self-reply (format: 'link-reply') is the link-reach hypothesis tested as a format variable in neomjs/neo#19575"` (`outbound-post.md` §5.1, from D#19500 §3) makes link-in-first-reply intended for cold X accounts, *"a measured hypothesis under test, never as a settled platform fact"*. This ticket narrows both to their evidence; it overturns neither.
- Placement: payload-only, in the two existing references; no `SKILL.md` router change. Ran `npm run ai:structure-map -- --files --loc` in `neo-agent-brain` (exit 0); it maps Brain code, and the owning surface is the existing `outbound-post.md`, so no new file.

## The Fix

1. **`outbound-post.md` gains §0, "The job", before §1.** The author fills it before drafting and the reviewer checks it:
   - one reader action (open the article, star the repository, run the install) and where its number shows. Read the first venue's number before copying the shape to the next venue;
   - the link in view: prefer link-post venues, which render the card; in a text post the link sits in the first screen, together with the reason to click;
   - a trailer, not the article: the hook, the proof, and one open thread that only the click resolves;
   - the level in the first line: who is speaking, with its receipt;
   - the ending is the action and its link. A question only when replies are the goal and someone will answer them;
   - a reply links the article where its answer continues.
2. **§4.1 adds the cold-reader check:** *"would a reader who sees only this post take the action?"* A post can pass all 18 boundaries and fail it.
3. **§5.1 scopes link-in-first-reply** to the cold-X test it came from. On other venues the link goes in the post.
4. **`blog-authoring-guide.md` §1:** the CTA is the reader's next action with its link, not a question.
5. **#313 folded in, with its author's consent:** §1 gains the Try-It test for the closing "How to run it today" section. That section leads with what runs today, follows the progressive path, and never opens with what does not work. §4 has one reviewer read as a stranger.

Growth: about 1.2 KB, carried as `[skill-growth-justified: …]`. Retirement trigger: §0 folds into the `requestPublish` payload once the payload carries the action and the Fleet publish executor records its outcome (ADR 0042, D#19500 analytics).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `outbound-post.md` §0, §4.1 | this ticket; the 7 Nov outcome | the six fills before drafting; the reviewer's cold-reader check | an announcement with no click target names "none" and says why | `outbound-post.md` | the next review request carries the fills |
| `outbound-post.md` §5.1 | D#19500 §3, `neomjs/neo#19575` | link-in-first-reply only on the cold-X test | any venue may still run it as a declared test with its number | same | — |
| `blog-authoring-guide.md` §1 CTA | this ticket | the reader's next action with its link | a post whose goal is replies says so | same | — |

Decision Record impact: `aligned-with` ADR 0008 (payload-only). D#19500 §3's hypothesis is kept and scoped to its own venue.

## Acceptance Criteria

- [ ] AC-1 `outbound-post.md` carries §0 with the six fills, §4.1 names the cold-reader check, and §5.1 scopes link-in-first-reply to the cold-X test.
- [ ] AC-2 `blog-authoring-guide.md` §1 defines the CTA as the reader's next action with its link.
- [ ] AC-3 `npm run lint` and `npm test` pass; the version is bumped; the growth carries `[skill-growth-justified: …]` with the retirement trigger.
- [ ] AC-4 (post-merge) The next outbound piece's review request carries the §0 fills, and the first venue's number is read before its shape is copied to another venue.
- [ ] AC-5 (from #313) `blog-authoring-guide.md` §1 carries the Try-It test (two questions, the progressive path, no anti-CTA opening), and §4 has one reviewer run it, together with §0's cold-reader check for outbound pieces.

## Out of Scope

- Editing published posts: they are snapshots, and an edit brings no new traffic.
- The title test (#250). It composes with this ticket and lands on its own.
- The `requestPublish` schema (ADR 0042).

## Avoided Traps

- Another boundary. The defect is that nothing asks for effect, so the fix is a slot the reviewer reads, not a seventh flavor.
- Dropping the boundaries. They held: nothing went out that needed a correction.
- A memory-only fix. Three tickets carried this lesson and every post since shipped through the same unchanged reference.

## Related

#250 · #313 (folded in with its author's consent; closed as superseded by this ticket) · #306 · `neomjs/neo#19575` · D#19500 · `neomjs/neo#20563` (part 2)

Live latest-open + closed sweep: the latest 20 open issues plus a closed search for *outbound post link reply CTA* at 2026-11-08T02:23Z. #250 and #313 are adjacent and compose with this ticket; nothing else is equivalent or excluded. A2A in-flight sweep: the latest 30 messages at 02:23Z, no claim on this scope. Memory Core sweep: the 7 Nov finding that a text-post body replaces the article, and the venue-card proposal for the publish request; no prior ruling. Own-assignment sweep: #289 and #258, no overlap. Meta-skill sweep: `create-skill` read; payload-only.

Origin Session ID: 6bb2b8dc-e7ad-48a9-83ff-f78b6afcfb62

Retrieval Hint: "outbound post passes 18 boundaries but moves no reader: hidden level, body replaces article, link out of view, closing question"


## Timeline

- 2026-11-08T02:26:46Z @neo-opus-grace added the `enhancement` label
- 2026-11-08T02:26:46Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-08T02:26:46Z @neo-opus-grace added the `Blog Post` label
- 2026-11-08T02:26:46Z @neo-opus-grace added the `ai` label
- 2026-11-08T02:26:47Z @neo-opus-grace added the `model-experience` label
- 2026-11-08T02:30:28Z @neo-opus-grace cross-referenced by PR #315
- 2026-11-08T02:39:01Z @neo-opus-grace referenced in commit `f599406` - "feat(blog-post): the closing section passes the Try-It test, one reviewer reads as a stranger (#314)

Folds #313 in with its author's consent: blog-authoring-guide.md §1 gains the Try-It test for the closing How-to-run-it-today section (what runs today first, the progressive path, no anti-CTA opening); §4 has one reviewer run it alongside outbound-post.md §0's cold-reader check."
- 2026-11-08T02:41:13Z @neo-opus-grace referenced in commit `74a553c` - "fix(blog-post): the reviewer's link-reply check is scoped to cold X accounts, like §5.1 (#314)

Julia's review note: §4.1 still checked link-in-first-reply unscoped after §5.1 was scoped."
- 2026-11-08T02:41:52Z @neo-gemini-julia cross-referenced by #313
- 2026-11-08T02:44:10Z @neo-opus-grace referenced in commit `3097900` - "feat(blog-post): the level line names its source, and the outbound is drafted with the post (#314)

§0.4 points the level line at learn/benefits/Introduction.md (accountable AI peers: a name, a memory, rival-lab review, the right to refuse); §1's pre-flight drafts the outbound's venues and §0 fills with the post and reviews them in the same PR."
- 2026-11-08T02:45:15Z @neo-opus-grace referenced in commit `9c5e74f` - "fix(blog-post): the level line links the Introduction by its canonical URL and the census records it (#314)"
- 2026-11-08T03:14:46Z @tobiu referenced in commit `50577d9` - "feat(blog-post): an outbound post names the reader's next action and keeps its link in view (#314) (#315)

* feat(blog-post): an outbound post names the reader's next action and keeps its link in view (#314)

outbound-post.md gains §0 The Job: six fills before drafting (the action and
its number, the link in view, a trailer not the article, the level in line
one, end on the action, a reply links where its answer continues). §4.1 reads
§0 as a cold reader; §5.1 scopes link-in-first-reply to cold X accounts.
blog-authoring-guide.md §1: the CTA is the reader's next action with its link.

[skill-growth-justified: the outbound gate had no slot for effect, so posts passed 18/18 boundaries and moved no reader; §0 retires into the requestPublish payload once it carries the action and the Fleet executor records the outcome]

* feat(blog-post): the closing section passes the Try-It test, one reviewer reads as a stranger (#314)

Folds #313 in with its author's consent: blog-authoring-guide.md §1 gains the Try-It test for the closing How-to-run-it-today section (what runs today first, the progressive path, no anti-CTA opening); §4 has one reviewer run it alongside outbound-post.md §0's cold-reader check.

* fix(blog-post): the reviewer's link-reply check is scoped to cold X accounts, like §5.1 (#314)

Julia's review note: §4.1 still checked link-in-first-reply unscoped after §5.1 was scoped.

* feat(blog-post): the level line names its source, and the outbound is drafted with the post (#314)

§0.4 points the level line at learn/benefits/Introduction.md (accountable AI peers: a name, a memory, rival-lab review, the right to refuse); §1's pre-flight drafts the outbound's venues and §0 fills with the post and reviews them in the same PR.

* fix(blog-post): the level line links the Introduction by its canonical URL and the census records it (#314)"
- 2026-11-08T03:14:46Z @tobiu closed this issue
### @neo-opus-grace - 2026-11-08T03:15:32Z

Post-merge validation of AC-4, first half: met. The first outbound batch written under §0, replies 5–7 to part 2's Reddit cross-posts, carried the §0 fills in its review request, and the cross-family check read it against §0 as a cold reader before posting. The operator's own catch on that batch is a wording one: name the exact record, never "my last turns".

AC-4's second half is open until the next venue: read the first venue's number (Medium's reddit.com referrer, baseline 2) before its shape is copied to another venue.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 6bb2b8dc-e7ad-48a9-83ff-f78b6afcfb62

- 2026-11-08T03:40:33Z @neo-opus-grace cross-referenced by #316

