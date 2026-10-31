---
id: 250
title: 'Blog-post title test: the reasons the target reader would not click'
state: OPEN
labels:
  - enhancement
  - Blog Post
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-fable-clio
createdAt: '2026-10-30T23:49:12Z'
updatedAt: '2026-10-31T02:13:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/250'
author: neo-fable-clio
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
---
# Blog-post title test: the reasons the target reader would not click

## Context

Fri 2026-10-30 night, five title rounds on one ticket (neo #20184, the grid post for Showoff Saturday) in one hour — the product name as the headline's subject; a release digest; a superlative the linked benchmark report falsifies in its own scrolling table; an engineering statement; a scale number without the reader's pain; a competitor comparison — every one accurate, every one dead on the operator's stranger read. His rule: **challenge titles and subtitles the hardest possible way — find the reasons why the readers who SHOULD be the target audience will not click and will not read the first paragraph.** The earlier instance: *Multi-window web apps without syncing state* — the peers loved it, meant "the engine syncs for you, nested cross-window providers included", and strangers read "isolated windows, AI slop, state not even considered".

## The Problem

§3's title test is mechanical for **voice and identity** over-claims (flavors 4–5: the "Your AI…" frame, one-upmanship, bragging to a rival lab) and flavor 1 bans the *unsourced* superlative. A title passes all of it and creates zero traction: the product name as subject (nobody searches a name they have never heard), the author's-context parse ("without X" read as *lacking X*), a *sourced* superlative the source contradicts in another row, a true statement with no pull. Nothing in the guide asks **who the target reader is and why they would skip** — and the author cannot answer that alone: all five dead titles had passed the author's own read.

## The Architectural Reality

- `references/blog-authoring-guide.md` §3 (the five flavors, the claims test, the title test — 8,600 B) and §4 (the review bar: ≥ 2 reviews, one cross-family — the second reader already exists).
- #168 (closed) widened the skill to social posts through one load-on-demand outbound reference; the same test governs a post's title, a video's title and a Showoff blurb.
- Placement: §3 and §4 of the payload; no router change; `DISCIPLINE-ONLY`, with the second reader as the enforcement point.
- Budget: `maxPositiveDeltaBytes` 250 — the addition is offset by tightening §3's prose, or carries `[skill-growth-justified: …]`.

## The Fix

1. §3 gains **flavors 6–8, the traction failures** (one line each, with the Oct 30 examples): **6 the product-name headline** — accurate and invisible; **7 the author's-context title** — reads as its opposite to a stranger; "without X" only before a cost (jank, blocking), never before a capability (syncing, state); **8 the receipt-contradicted claim** — a superlative or number the linked source itself contradicts in another row; the first reader who opens the receipt finds it. Three dead forms as the examples: the engineering statement, the scale number without the reader's pain ("my HTML table does that too"), the competitor comparison ("marketing slop").
2. **The title test gains the hardest read:** (d) name the target reader and the strongest reason they would not click or read the first paragraph; answer it or rewrite; (e) what sparks the click is a gap the reader wants closed — a claim they disagree with at first sight, a paradox they want explained — never a brag; numbers go in the subtitle and paragraph one as proof, not in the title as pitch; (f) **the second reader of §4 runs the hostile pass on title, subtitle and first paragraph before the publish request**, and the publish request carries the answer — the author's own pass never counts.
3. The social-post reference (#168) points at the same test for titles, thumbnails and the first three seconds.
4. Empirical anchor: neo #20184 / #20185 (the five rounds, public), beside #14877's.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `blog-authoring-guide.md` §3 | this ticket; the operator's rule (Oct 31) | flavors 6–8 with examples; the title test (d)–(f) | a post with no identifiable target reader fails (d) by construction | §3 | the PR's load-effect audit: payload-only, delta ≤ 250 B or justified |
| `blog-authoring-guide.md` §4 | the review bar | the second reader's hostile pass named as a duty, before the publish request | — | §4 | the next post's review thread shows it |
| the outbound/social reference (#168) | the same | one pointer line to §3's test | — | that file | — |

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 §3 carries flavors 6–8 with the Oct 30 examples and the extended title test (d)–(f); §4 names the second reader's hostile pass; no router change.
- [ ] AC-2 The social-post reference points at the test.
- [ ] AC-3 Net `.agents/skills/**/*.md` delta ≤ 250 B or `[skill-growth-justified: …]` with the retirement condition (a title lint); `lint-skill-manifest.mjs` green; the version bump per the repository's rule.
- [ ] AC-4 (post-merge) The next public piece's publish request carries the answered hardest read (receipt: neo #20184's request).

## Out of Scope

A mechanical title lint; Medium or YouTube mechanics; the private operating page's publish gate (it already carries the field).

## Avoided Traps

- Restating the operator's words as a scolding: the flavors are patterns with examples, like 4–5.
- A hot-index memory rule only: the seat had the rule and still wrote five dead titles; the second reader is the enforcement.
- A new skill: this is the existing skill's §3/§4.

## Related

#168 · neo #14877 (flavors 4–5's anchor) · neo #13486 (flavors 1–3's) · neo #20184 · neo #20185.

Live latest-open sweep: the 20 latest open issues of this repository at 2026-10-30 23:46Z, none equivalent (#168 is the social widening, closed). A2A in-flight claim sweep: the latest 15 messages at 23:50Z, none on this scope. Memory Core rationale: Grace's title alternatives for the 13.2 post (Oct 12) and #14877's batch are the prior rounds; no prior decision on a traction test. Own-assignment sweep: #10, no overlap. Meta-skill sweep (§1b): `create-skill` guide read whole.

Origin Session ID: fc680704-8742-4651-964e-9e7a053b27e9

Retrieval Hint: "blog title test hardest read target reader will not click first paragraph traction flavors product-name headline worst parse"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session fc680704-8742-4651-964e-9e7a053b27e9

## Timeline

- 2026-10-30T23:49:12Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-30T23:49:14Z @neo-fable-clio added the `enhancement` label
- 2026-10-30T23:49:14Z @neo-fable-clio added the `Blog Post` label
- 2026-10-30T23:49:14Z @neo-fable-clio added the `ai` label
- 2026-10-30T23:49:14Z @neo-fable-clio added the `model-experience` label
- 2026-10-30T23:49:23Z @github-actions added the `needs-design-authority` label
### @neo-fable-clio - 2026-10-31T02:13:13Z

Two deltas from the operator's read of the grid post (Oct 31, ~00:1xZ), relayed by Grace by A2A after her comment write was blocked; folded here so the PR carries them.

1. **Statement titles, never questions** — interrogative clauses without a question mark included. The reason is not "Reddit removes them": r/javascript routes question posts to its Ask flair, and r/webdev's handling is unverified. The reason that holds everywhere: a statement reads the same in every venue the post reaches; a question needs a venue that tolerates it.
2. **The test unit is the preview, per venue.** The hostile read runs on what a stranger sees before the click, and that differs by venue: Medium shows the title, the subtitle and the cover; a Reddit link post shows the poster's own title, the cover as thumbnail and the domain, with no subtitle. The check names the venues the post will reach and runs on each preview.

Both land in the title test (d)–(f) and flavors 6–8 of the PR. Source: Grace's DMs of 00:28Z and 00:34Z; the operator's read in her session.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 8bd753e1-2606-4bbf-873f-66d0c0284de3


