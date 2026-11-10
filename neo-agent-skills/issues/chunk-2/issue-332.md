---
id: 332
title: 'blog-post: stage posts on dev a day early and keep Reddit bodies as focused angles'
state: CLOSED
labels:
  - documentation
  - enhancement
  - Blog Post
  - ai
  - model-experience
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-10T14:21:18Z'
updatedAt: '2026-11-10T15:23:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/332'
author: neo-gemini-maryam
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
closedAt: '2026-11-10T15:23:42Z'
---
# blog-post: stage posts on dev a day early and keep Reddit bodies as focused angles

## Context

Across recent blog post and outbound social publication cycles (`neomjs/neo#20185`, `neomjs/neo#20563`, `neomjs/neo#20718`, and `neomjs/neo-agent-skills#314` / PR `#316`), two recurring operational failure modes emerged in the `blog-post` skill:

1. **Same-day publication rush (`content-one-day-early`):** A dated publication depends on final text, cover art merged to `neomjs/pages` `main` (`resources_pub/website/blog/<slug>/`), portal registration in `apps/portal/resources/data/blog.json`, render verification of visuals and crops, one consolidated final-version fact check by a non-drafting seat (`#262`), and cross-family review (`blog-authoring-guide.md` §4–§5, `outbound-post.md` §4). Note: this skill leaf records the D-1 18:00 CET lock rule in `blog-authoring-guide.md` §5 and `outbound-post.md` §3; mechanical publish-gate / workflow enforcement remains open on the publish-gate surface.
2. **Reddit self-post body replacing the article (`reddit-body-replaces-article`):** `.agents/skills/blog-post/references/outbound-post.md` §0 item 3 states the general trailer rule (*"A trailer, not the article: the hook, the proof, and one thread only the click resolves. A body that carries the whole method replaces the article, and readers answer the summary."*), but does not call out the recorded Reddit failure: a short text-post body that readers answered instead of opening the article. Its remedy is to prefer a link post so the card renders (item 2), or in a text post, write a body that exists only to send the reader to the article, with the link in the first screen.

Design authority: `.agents/skills/blog-post/references/blog-authoring-guide.md` §4–§5 (`#13486`, `#262`) and `.agents/skills/blog-post/references/outbound-post.md` §0 item 3 and §3 (`neomjs/neo-agent-skills#314` / PR `#316`).

- **Live latest-open + closed sweep:** checked all 31 open issues in `neomjs/neo-agent-skills` (31 open — orange band; `#250` covers title traction flavors 6–8 and is assigned to `@neo-opus-grace`, non-overlapping) and ran closed search (`blog-post reddit early`, `outbound-post.md`, `#314` / `#262` `Out of Scope` and `Avoided Traps` read) at `2026-11-10T14:16:31Z`; no equivalent or prior exclusion found.
- **A2A in-flight claim sweep:** checked `list_messages({status: 'all', projection: 'compact', limit: 30})` at `2026-11-10T14:16:33Z`; no competing claim on `blog-post` day-early staging or Reddit body framing.
- **Memory Core rationale + own-assignment sweep:** ran `query_raw_memories({query: 'blog-post skill content-one-day-early reddit-body-replaces-article outbound-post.md', detail: 'summary', nResults: 5})` and `gh issue list --repo neomjs/neo-agent-skills --state open --assignee @me` (0 open issues); no prior exclusion found.
- **Meta-skill & Turn-Memory Pre-Flight sweep (§1b, §1c):** Read `.agents/skills/create-skill/SKILL.md`, `.agents/skills/turn-memory-pre-flight/SKILL.md`, and ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --root ai/scripts --files --loc`. Top-level `.agents/skills/blog-post/SKILL.md` router (`11` lines, `1,140` B) is untouched; edits are confined to on-demand payloads `references/blog-authoring-guide.md` and `references/outbound-post.md`, with stale pre-merge caveats in `outbound-post.md` retired so the net `.agents/skills/**/*.md` byte delta is `-152` B.

## The Problem

1. `.agents/skills/blog-post/references/blog-authoring-guide.md` §5 (`Mechanics`) and `.agents/skills/blog-post/references/outbound-post.md` §3 specify how blog posts and outbound `requestPublish` requests ship, but do not state the D-1 18:00 CET lock before the operator's publish click (nor distinguish landing on `dev` from portal publication, since `neomjs.com` serves the published npm package).
2. `.agents/skills/blog-post/references/outbound-post.md` §0 item 3 (`A trailer, not the article`) states that a body carrying the whole method replaces the article, without naming Reddit's recorded failure mode (a short text-post body answered instead of opening the article) and its remedy (prefer a link post so the card renders per item 2; in a text post, the body exists only to send the reader to the article, with the link in the first screen).
3. `.agents/skills/blog-post/references/outbound-post.md` lines 5–7 and 69 still carry `Runtime Implementation Status (Unmerged / Not Yet Deployed on Every Seat)` warnings for `neomjs/neo-agent-brain#1263`, `#1264`, `#1267` (closed as completed), and `neomjs/neo-agent-institution#673`, which have since landed.

## The Architectural Reality

- `.agents/skills/blog-post/SKILL.md` (`11` lines, `1,140` B) is the Progressive Disclosure router and needs no change.
- `.agents/skills/blog-post/references/blog-authoring-guide.md` (`61` lines, `9,559` B) owns blog post authoring and §5 shipping mechanics.
- `.agents/skills/blog-post/references/outbound-post.md` (`160` lines, `23,115` B) owns outbound social post drafting (§0 `The Job` and §3 `The Request Form`).
- Per `AGENTS.md §self_evolving_systems` (Accretion Defense) and `scripts/check-substrate-size.mjs`, retiring the obsolete unmerged-PR caveats in `outbound-post.md` lines 5–7 and 69 offsets both additions and yields a `-152` B net byte reduction across `.agents/skills/**/*.md` (`32,674` B → `32,522` B across the two modified files).

## The Fix

1. **In `.agents/skills/blog-post/references/blog-authoring-guide.md` (§5 `Mechanics` — `Ship`) and `.agents/skills/blog-post/references/outbound-post.md` (§3 `The Request Form`):**
   - State the D-1 18:00 CET lock rule for every dated publication before the operator's publish click (with the exception that urgent incident or security notices follow operator direction), and clarify in `blog-authoring-guide.md` §5 that a blog post reaches that lock by landing on `dev` while `neomjs.com` serves the published npm package (`9,559` B → `9,918` B, `+359` B).
2. **In `.agents/skills/blog-post/references/outbound-post.md` (§0 item 3 `A trailer, not the article`):**
   - Add the Reddit remedy: prefer a link post so the card renders (item 2); in a text post, the body exists only to send the reader to the article, with the link in the first screen.
   - Retire the obsolete unmerged-PR status bullet at lines 5–7 and the `"Once ... is merged"` prefix at line 69 so `outbound-post.md` drops from `23,115` B → `22,604` B (`-511` B net) and the combined `.agents/skills/**/*.md` byte delta is `-152` B.
3. **Bump `package.json` and `package-lock.json` version** (`1.3.3` -> `1.3.4`) per `scripts/test-check-version-bump.mjs`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/blog-post/references/blog-authoring-guide.md` §5 & `outbound-post.md` §3 | this ticket; `blog-authoring-guide.md` §4–§5 | For any dated publication, lock final text, cover/media, and fact check / cross-family attestation by 18:00 CET on D-1 before the operator's publish click (`dev` merge for blog posts; `neomjs.com` serves the npm release) | Urgent incident or security notices follow operator direction | `blog-authoring-guide.md` §5, `outbound-post.md` §3 | `npm run lint` and `npm test` in `neo-agent-skills` |
| `.agents/skills/blog-post/references/outbound-post.md` §0 item 3 | this ticket; `neomjs/neo-agent-skills#314` | On Reddit, prefer a link post so the card renders (item 2); in a text post, the body exists only to send the reader to the article, with the link in the first screen; stale unmerged caveats retired | Applies to all Reddit link/text posts | `outbound-post.md` §0 | `npm run lint`, `npm test`, and `-152` B net byte delta |

Decision Record impact: `aligned-with` `neomjs/neo#adr:skill-anatomy-and-authoring-contract` (payload-only refinement with net byte reduction).

## Acceptance Criteria

- [ ] AC-1 `.agents/skills/blog-post/references/blog-authoring-guide.md` §5 and `.agents/skills/blog-post/references/outbound-post.md` §3 state the D-1 18:00 CET lock before the operator's publish click (including the urgent-notice fallback and the `dev` vs published npm portal distinction).
- [ ] AC-2 `.agents/skills/blog-post/references/outbound-post.md` §0 item 3 states that on Reddit, prefer a link post so the card renders (item 2), and in a text post, the body exists only to send the reader to the article, with the link in the first screen.
- [ ] AC-3 Net `.agents/skills/**/*.md` byte delta is `<= 0` (via retiring the stale unmerged caveats in `outbound-post.md`), `package.json` + `package-lock.json` version is bumped, and `npm run lint && npm test` pass.

## Out of Scope

- Changing `.agents/skills/blog-post/SKILL.md` router text.
- Mechanical publish-gate / workflow enforcement of the D-1 18:00 CET lock (the `content-one-day-early` register row stays open on its publish-gate surface until enforced there).
- Title traction flavors 6–8 (tracked separately in `neomjs/neo-agent-skills#250`).

## Avoided Traps

- **Adding rules to `SKILL.md` router:** Both rules govern authoring/review execution inside the loaded reference payloads (`blog-authoring-guide.md` and `outbound-post.md`), not trigger routing.
- **Recommending a question body on Reddit:** Contradicts `outbound-post.md` §0 item 5 (`#314`) and invites replies to the Reddit body instead of reads of the article.
- **Equating `on dev` with published on `neomjs.com`:** `neomjs.com` serves the published `neo.mjs` npm package, so a `dev` post reaches the portal on the next release.
- **Growing total skill corpus bytes without retiring stale text:** Retiring the obsolete `Unmerged / Not Yet Deployed` caveats in `outbound-post.md` keeps the corpus leaner (`-152` B net).

## Related

- `neomjs/neo-agent-skills#314` / PR `#316` (`outbound-post.md` §0 `The Job`)
- `neomjs/neo-agent-skills#262` (`blog-authoring-guide.md` §2 consolidated fact check)
- `neomjs/neo#20718` / `neomjs/neo#20563` / `neomjs/neo#20185`

Not `good first issue`: already-owned maintainer deliverable (`@neo-gemini-maryam`).

Origin Session ID: 985cc2dd-9f98-42ad-b4d3-660e4e5336e5
Retrieval Hint: "blog-post skill content-one-day-early reddit-body-replaces-article outbound-post.md blog-authoring-guide.md"

Authored by Maryam (Gemini 4 Argon, Antigravity). Session 985cc2dd-9f98-42ad-b4d3-660e4e5336e5. 🧭

## Timeline

- 2026-11-10T14:21:18Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-10T14:21:19Z @neo-gemini-maryam added the `documentation` label
- 2026-11-10T14:21:20Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-10T14:21:20Z @neo-gemini-maryam added the `Blog Post` label
- 2026-11-10T14:21:20Z @neo-gemini-maryam added the `ai` label
- 2026-11-10T14:21:20Z @neo-gemini-maryam added the `model-experience` label
- 2026-11-10T14:29:54Z @neo-gemini-maryam cross-referenced by PR #333
- 2026-11-10T14:45:44Z @neo-gemini-maryam referenced in commit `5f10b12` - "docs(blog-post): align Reddit card/body rule and D-1 18:00 CET lock (#332)"
- 2026-11-10T15:23:43Z @tobiu referenced in commit `a28885c` - "docs(blog-post): stage posts a day early and keep Reddit bodies as focused angles (#332) (#333)

* docs(blog-post): stage posts a day early and keep Reddit bodies as focused angles (#332)

* docs(blog-post): align Reddit card/body rule and D-1 18:00 CET lock (#332)"
- 2026-11-10T15:23:43Z @tobiu closed this issue

