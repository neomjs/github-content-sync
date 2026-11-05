---
id: 279
title: 'Blog covers lose their edges to Medium''s 3:2 thumbnail crop'
state: CLOSED
labels:
  - enhancement
  - Blog Post
  - design
  - ai
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-05T14:48:31Z'
updatedAt: '2026-11-05T16:06:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/279'
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
closedAt: '2026-11-05T16:06:50Z'
---
# Blog covers lose their edges to Medium's 3:2 thumbnail crop

## Context

The operator, 2026-11-05, on the Medium copy of the persistent-memory post: "the blog cover image does not work well for medium at all. wrong dimensions. this needs a blog post skill update. obviously rendering nice inside our portal app matters too."

The cover is 2400×1260 (1.91:1). Medium's thumbnails cut its labels at both edges: "ion's context window" and "ules + memory index". The Medium copy was fixed by hand with a 3:2 variant, padded with the cover's own background. The rule that would have prevented it does not exist.

## The Problem

`blog-authoring-guide.md` says where images live and that every visual is render-verified. It says nothing about a cover's size or about where its content may sit. Every surface that crops a cover crops from the centre, and the surfaces disagree on the ratio:

| Surface | What it shows | Measured from |
|---|---|---|
| Medium feed and profile thumbnails | a centred 3:2 fill (`resize:fill:160:106`, `resize:fill:320:214`) | the story's image URLs on its author page, 2026-11-05 |
| Medium `og:image`, X `summary_large_image` | the whole image (`resize:fit:1200`), then the card's 1.91:1 | the story's meta tags |
| Portal Medium cards (`apps/portal/view/news/medium/List.mjs`), showing the record's own `image` from `medium_blog.json` | a centred `cover` fill. The card is 400 px tall from a 601 px wide, 401 px tall viewport, otherwise 200 px, and 100 % wide up to 800 px. The crop therefore runs from about 1.4:1 (581×400 at the breakpoint) to 4:1 (800×200 in short windows). Title and author sit bottom-left (`left: 0; right: 30 %`, 15 % on 200 px cards). Curated posts also carry a top-right "selected into" line | computed from `resources/scss/src/apps/portal/news/medium/List.scss`; the 800×200 case was rendered by the reviewer |
| The post's hero in the portal and Medium's header | the whole image | `<img width="100%">` in the post |

On a 2400×1260 cover, the 3:2 thumbnails lose 255 px from each side. Across every portal card size, only the centred 1760×600 survives.

## The Architectural Reality

- §5 Mechanics, bullet `Images`, covers the location (`neomjs/pages` → `resources_pub/website/blog/<slug>/`) and `alt` text.
- §1 says visuals are required and render-verified before merge. Its `Ship` bullet asks for a render receipt that names each visual.
- neomjs/neo#15699 (graduated from D#15673, unassigned) owns the wider cover system: profiles that require, allow or refuse a cover, and a cover brief and receipt with "safe-area variants". This ticket delivers only the measured safe area. The profiles, brief and provenance stay with neomjs/neo#15699.

Design authority: n/a. This adds a rule where the guide is silent; it calls no existing behavior wrong.

## The Fix

In `blog-authoring-guide.md` §5:
- **`Cover`:** 2400×1260 (1.91:1), so link cards and the hero show the whole image. Text and subject sit 60 px inside the centred 1890×1260, which is Medium's 3:2 thumbnail crop.
- **`Portal Medium card`:** a post listed in `medium_blog.json` gets a card `image` whose subject sits in the centred 1760×600 band and whose text stays clear of the bottom-left title and author.
- **`Ship`:** the render receipt shows the cover's 3:2 crop and, for a listed post, its card at 800×400 and 800×200.

The two surfaces are scoped separately on purpose. One safe area that held across all of them would be a 1760×600 strip, too thin for a cover.

Bump the patch version.

## Acceptance Criteria

- [ ] §5 states the cover size and the centred 3:2 safe area with its margin, each with the surface that imposes it.
- [ ] §5 states the portal Medium card's crop range, its centred band and its bottom-left overlay.
- [ ] The `Ship` bullet's render receipt names the 3:2 crop and, for a listed post, the card at 800×400 and 800×200.
- [ ] `SKILL.md` is unchanged (always-loaded delta 0), and the package version is bumped.
- [ ] This ticket cross-references neomjs/neo#15699, so the measured crops and this slice appear on that ticket's timeline.

## Out of Scope

- Cover profiles, briefs, provenance and hashes: neo #15699.
- `og:image` on neomjs.com's static pages, which is the SEO lane.
- Re-composing existing covers.

## Avoided Traps

- **A 3:2 master:** link cards (1.91:1) would then crop its top and bottom.
- **Per-surface variants by default:** three assets to keep in sync for crops that one safe area survives.
- **A vendor's "16:9 for Medium" figure:** unmeasured. The thumbnails' own URLs say 3:2.

## Related

neomjs/neo#15699 (cover system) · neomjs/neo#20226, neomjs/neo#20321 (the post) · #266 (visuals required)

Sweeps (2026-11-05T14:43Z), with no equivalent found; neo #15699 is the wider owner, see above:
- the latest 20 open issues here;
- an all-state search on cover / hero image / Medium / thumbnail / og:image / aspect ratio;
- neo issues on blog cover requirements;
- A2A claims since 14:20Z;
- Memory Core on the problem nouns (one precedent: a centre-safe Devpost thumbnail, July);
- my own assignments.

Origin Session ID: 8545cff9-f4d7-4731-b93e-7dfad1347c8a
Retrieval Hint: "blog cover Medium 3:2 thumbnail crop safe area 2400x1260 portal medium card overlay"



## Timeline

- 2026-11-05T14:48:31Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-05T14:48:33Z @neo-opus-grace added the `enhancement` label
- 2026-11-05T14:48:33Z @neo-opus-grace added the `Blog Post` label
- 2026-11-05T14:48:33Z @neo-opus-grace added the `design` label
- 2026-11-05T14:48:33Z @neo-opus-grace added the `ai` label
### @neo-opus-grace - 2026-11-05T14:49:02Z

**Contract Ledger** (skill-change PR gate, `create-skill` "PR-Open Gates"). Corrected at 98054b5 after Emmy's RA-1 on #280.

| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `blog-authoring-guide.md` §5 `Cover` | Operator, 2026-11-05, plus Medium's crop | The cover is 2400×1260. Text and subject sit 60 px inside the centred 1890×1260, the 3:2 crop | A cover that cannot fit ships a 3:2 copy for Medium, padded with its own background (the persistent-memory post did) | The guide | Medium image URLs `resize:fill:160:106` and `320:214` |
| `blog-authoring-guide.md` §5 `Portal Medium card` | `news/medium/List.scss` | A post listed in `medium_blog.json` gets a card `image`. Its subject sits in the centred 1760×600 band, and its text stays clear of the bottom-left title and author | No `image` change for posts that are not listed | The guide | Computed from the SCSS: 400 or 200 px tall, ≤ 800 px wide, so the crop runs from 1.4:1 to 4:1. The reviewer rendered 800×200 |
| `blog-authoring-guide.md` §5 `Ship` render receipt | §1, which requires every visual to be render-verified (#266) | The receipt shows the cover's 3:2 crop and, for a listed post, its card at 800×400 and 800×200 | Posts without a cover are unchanged | The guide | Diff |
| Package version | `check-version-bump` CI | 0.1.88 → 0.1.89 | None | None | CI |

The always-loaded delta is 0: `SKILL.md` is unchanged, and the rule lives in the conditional payload that `blog-post` loads.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 8545cff9-f4d7-4731-b93e-7dfad1347c8a


- 2026-11-05T14:50:49Z @github-actions added the `needs-design-authority` label
- 2026-11-05T14:52:16Z @neo-opus-grace cross-referenced by PR #280
- 2026-11-05T15:39:21Z @neo-opus-grace referenced in commit `98054b5` - "feat(blog-post): the portal's Medium card gets its own crop band and overlay rule (#279)

Review RA-1: the portal card crops between 1.4:1 and 4:1 (400 or 200 px tall, up to 800 px wide), not 2:1 to 3:2, and its title and author sit bottom-left. A centred 3:2 safe area covers Medium only; the card's image keeps its subject in the centred 1760x600 band, and the receipt shows the card at 800x400 and 800x200."
- 2026-11-05T16:06:50Z @tobiu referenced in commit `3a2d7bc` - "feat(blog-post): covers keep their content inside the centred 3:2 that Medium and the portal crop to (#279) (#280)

* feat(blog-post): covers keep their content inside the centred 3:2 that Medium and the portal crop to (#279)

Medium's feed and profile thumbnails fill 3:2 and the portal's Medium cards
fill 2:1 to 3:2, both from the centre, so a 2400x1260 cover loses 255 px at
each side there. The rule keeps text and subject inside the centred
1890x1260, and the render receipt shows that crop.

[skill-growth-justified: operator-requested cover geometry, measured on Medium and the portal; it folds into neo #15699's cover receipt when that lands]

* feat(blog-post): the portal's Medium card gets its own crop band and overlay rule (#279)

Review RA-1: the portal card crops between 1.4:1 and 4:1 (400 or 200 px tall, up to 800 px wide), not 2:1 to 3:2, and its title and author sit bottom-left. A centred 3:2 safe area covers Medium only; the card's image keeps its subject in the centred 1760x600 band, and the receipt shows the card at 800x400 and 800x200."
- 2026-11-05T16:06:51Z @tobiu closed this issue

