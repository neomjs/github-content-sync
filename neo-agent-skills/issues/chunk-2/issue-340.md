---
id: 340
title: outbound-post.md still counts five flavors; §3 now lists eight
state: CLOSED
labels:
  - bug
  - documentation
  - ai
  - model-experience
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T02:33:54Z'
updatedAt: '2026-11-11T02:56:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/340'
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
closedAt: '2026-11-11T02:56:43Z'
---
# outbound-post.md still counts five flavors; §3 now lists eight

## Context

neomjs/neo-agent-skills#339 (`Resolves #250`, merged at `ea6ecc1`) grew `blog-authoring-guide.md` §3 from five flavors to eight: 1–3 factual, 4–5 tonal, 6–8 traction (product-name headline, author's-context title, receipt-contradicted claim). `outbound-post.md`, the social-post reference that sends drafters to that list, kept the old count in three places. Maryam's Round-1 approval of #339 (`5852790619`) flagged two of them as a non-blocking watch item; a grep at `ea6ecc1` found the third.

## The Problem

At `ea6ecc1`, `.agents/skills/blog-post/references/outbound-post.md`:

- `:40` (§1, boundary 11): *"all five over-claim flavors in blog-authoring-guide.md §3 apply equally to social posts and replies"*
- `:56` (§2): *"Kill all five over-claim flavors (§3):"*, followed by items 1–5
- `:131` (§4, what the reviewer checks): *"All 5 over-claim flavors in §2, and the title test of `blog-authoring-guide.md` §3 on each preview"*

A drafter who follows §1 and §2 never meets flavors 6–8. They surface only at §4's cross-family review, through the title test. #250 moved that check to authoring time for blog posts, and social drafting still skips it: a Reddit link-post title or a video thumbnail is drafted against five flavors and judged against eight.

## The Architectural Reality

Design authority: "Social posts are smaller blog posts with the same failure modes (`D#19500` OQ-3). Apply [blog-authoring-guide.md](./blog-authoring-guide.md) §2–§3 in full:" (`outbound-post.md:53`)

- `outbound-post.md` is a `blog-post` skill payload, read on demand when the skill fires; no turn-loaded surface changes.
- The three counts contradict the authority sentence above: "in full" is eight flavors, and the counts say five.
- Items 1–5 of §2 are not copies of the guide. They carry social readings, such as the Karpathy-loop axes under flavor 5, so they stay. The traction flavors have no social reading beyond the guide's own rule (f), *"The unit is each venue's preview"*, so §2 points to them instead of restating them.

## The Fix

Count-free wording, so the next flavor §3 gains cannot leave a stale count behind:

1. `:40` → *"every over-claim and traction flavor in blog-authoring-guide.md §3 applies equally to social posts and replies"*.
2. `:56` → *"Kill every flavor in §3: the traction flavors as §3 writes them, on each preview; the over-claim flavors as follows:"*, with items 1–5 unchanged.
3. `:131` → *"Every flavor in §2, and the title test of `blog-authoring-guide.md` §3 on each preview"*.
4. The version bump that `check-version-bump` requires (`package.json` and `package-lock.json`).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `outbound-post.md` §1 boundary 11, §2's flavor lead-in, §4's reviewer check | `blog-authoring-guide.md` §3, the single list of flavors | the drafter and the reviewer text cover every §3 flavor and name no count | none (docs only) | `outbound-post.md` itself | a grep over `.agents/skills/blog-post/` for "five over-claim" and "All 5 over-claim" finds nothing |

## Decision Record impact

none

## Acceptance Criteria

- [ ] `outbound-post.md` names no flavor count. §1 boundary 11, §2's lead-in and §4's reviewer check each cover every §3 flavor, the traction flavors included.
- [ ] Items 1–5 of §2 keep their social text unchanged.
- [ ] `npm run lint` and `npm test` pass at the head, and `package.json` carries the bump.

## Out of Scope

- Restating flavors 6–8 inside `outbound-post.md`. The guide's §3 and its title test are their single source, and §2 points there.
- Any change to `blog-authoring-guide.md`.
- Renumbering §1's 18 boundaries.

## Related

- #250 (resolved by #339; not reopened, per the fix-means-new-ticket rule)
- #339
- neomjs/neo#20726, an open blog-post lane that drafts against these files

Live latest-open + closed sweep: checked the latest 20 open and a closed search ("over-claim flavors outbound": #314, #250 and #168 read, none governs the count) at 2026-11-11T02:32Z; no equivalent or prior exclusion found. Backlog band: 32 open (orange); no owning ticket exists for `outbound-post.md`, and #336's stale count sits on another surface.
A2A in-flight claim sweep: the last 30 messages (all read states) carry no lane claim on `outbound-post.md` or the flavor list.
Memory Core rationale sweep: `query_raw_memories("outbound post over-claim flavors count stale after traction flavors added")` found no decision about the count; the hits predate #339 and cite five flavors.
Own-assignment sweep: #335, #289 and #258 are assigned to me; none touches `blog-post`.
Structure map: N/A. This is the Skills repository; the owning folder is `.agents/skills/blog-post/references/`, and the sibling precedent is #339.

Ownership: self-assigned (Grace). This is residue of my own #339, fixed in the same session, so it is already owned and not a `good first issue`.

Retrieval Hint: "outbound-post flavor count traction flavors drafter review gate"

Origin Session ID: 40a9201b-5e9e-4845-ab9f-dee27312077d

## Timeline

- 2026-11-11T02:33:54Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T02:33:56Z @neo-opus-grace added the `bug` label
- 2026-11-11T02:33:56Z @neo-opus-grace added the `documentation` label
- 2026-11-11T02:33:56Z @neo-opus-grace added the `ai` label
- 2026-11-11T02:33:56Z @neo-opus-grace added the `model-experience` label
- 2026-11-11T02:38:44Z @neo-opus-grace cross-referenced by PR #341
- 2026-11-11T02:56:43Z @tobiu referenced in commit `1795746` - "fix(blog-post): outbound drafters meet every §3 flavor, with no count to go stale (#340) (#341)

#339 grew blog-authoring-guide.md §3 to eight flavors and edited the
outbound reviewer line without its count. outbound-post.md still said
five in boundary 11, in §2's lead-in and in §4's reviewer check, so a
social drafter met the traction flavors only at review.

The three lines now name every flavor and no number. §2 keeps its
social readings of flavors 1-5 and points to §3 for the traction ones.
Version 1.3.6 -> 1.3.7."
- 2026-11-11T02:56:44Z @tobiu closed this issue

