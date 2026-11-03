---
id: 265
title: 'Blog posts carry a Mermaid diagram or an image, never only text'
state: OPEN
labels:
  - enhancement
  - Blog Post
  - ai
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-03T17:35:00Z'
updatedAt: '2026-11-03T17:47:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/265'
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
---
# Blog posts carry a Mermaid diagram or an image, never only text

## Context

The operator's review of the memory post, [neomjs/neo#20226 review 5755007333](https://github.com/neomjs/neo/pull/20226#pullrequestreview-5755007333): "For blog posts, mermaid diagrams and / or images should be a requirement. Please update our blog post skill, in case it is missing in there. Rationale: otherwise we end up with 'walls of text'."

## The Problem

The skill makes a visual optional. `blog-authoring-guide.md` §1: "Diagrams (Mermaid) earn their place only when they carry information prose can't". The memory post followed it and passed two cross-family reviews as 87 lines of prose: nothing in the skill asked for a visual, so no author or reviewer had a line to check. On `neomjs/neo` `dev` (`f0e0e8e903`), 10 of the 21 posts in `learn/blog/` contain neither a Mermaid block nor an image.

## The Architectural Reality

- **Map:** `.agents/skills/blog-post/SKILL.md`. Its closing line lists what every post is held to.
- **Atlas:** `.agents/skills/blog-post/references/blog-authoring-guide.md`. §1 holds the optional-diagram sentence; §5 Mechanics holds file, registration, SEO and ship. A closing "Empirical Anchor" section repeats the anchor that line 5 already states (#13486).
- **Rendering:** the portal renders ```` ```mermaid ```` blocks through `Neo.component.Markdown` at `#/news/blog/<blog.json id>`. Images are served from `neomjs/pages` `main` (`resources_pub/website/blog/<slug>/`), so the image PR merges before the post (neomjs/pages#59 → #60; neomjs/pages#61 → #62).
- Placement (`turn-memory-pre-flight` Step 2): a workflow rule, so it belongs in this skill. The rule body goes in the atlas; the router gets only a few words.

## The Fix

1. §1: replace the optional-diagram sentence with the requirement. Every post carries at least one Mermaid diagram or image. A section that explains a structure, a flow or a sequence shows it. No visual is decoration, and each is render-verified before merge.
2. §5: an **Images** bullet: the `neomjs/pages` path, the merge order and descriptive `alt` text. The **Ship** bullet asks the PR's render receipt to name each visual it saw render. That receipt is the slot where the rule is emitted.
3. `SKILL.md`: the closing line adds the visual to what every post is held to.
4. Retire the closing "Empirical Anchor" section, which duplicates line 5, so the change does not grow the loaded bytes.
5. Bump `package.json`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `blog-authoring-guide.md` §1, visuals sentence | operator review 5755007333 | required: at least one Mermaid diagram or image per blog post, render-verified | none: a post without one is a review finding | the guide | diff; `npm run lint` |
| `blog-authoring-guide.md` §5, Images + Ship | neomjs/pages#59/#60 precedent | the pages path, merge first, `alt` text; the render receipt names each visual | none | the guide | diff |
| `blog-post/SKILL.md` closing line | operator review 5755007333 | names the visual among what every post is held to | none | the router | `npm run lint` (router budget) |
| `package.json` `version` | release flow | `0.1.84` → `0.1.85` | none | none | `npm test` (version-bump check) |

## Decision Record impact

none.

## Acceptance Criteria

- [ ] §1 requires at least one Mermaid diagram or image per blog post and keeps render verification.
- [ ] §5 names the `neomjs/pages` path, the merge order and `alt` text, and asks the render receipt to name each visual.
- [ ] `SKILL.md`'s closing line names the visual, and the router stays within its budget.
- [ ] The blog-post skill's loaded bytes do not grow.
- [ ] `package.json` is bumped; `npm run lint` and `npm test` pass.

## Out of Scope

- Outbound social posts and funding/pitch copy: the operator's ask names blog posts.
- A mechanical check in `neomjs/neo` that fails a changed `learn/blog/*.md` with no visual. That is a MACHINE-ENFORCEABLE-CANDIDATE, worth filing if the rule slips.
- Retrofitting the 10 existing posts.

## Avoided Traps

- **Counting instead of carrying.** A stock image satisfies "at least one" and still leaves a wall of text. The rule asks each visual to show the mechanism, the sequence or the real UI the section explains.

Live latest-open + closed sweep: checked latest 20 open issues, open PRs and a closed search (blog diagram image visual) at 2026-11-03T17:33Z; open #250 (title test) is a different rule; no equivalent or prior exclusion. A2A in-flight claims (last 30, all read-states): none on this scope. Memory Core (blog post mermaid diagram visuals requirement): no prior decision. Own assignments: #258 only, unrelated.

Related: neomjs/neo#20222, neomjs/neo#20226, neomjs/pages#61.

Origin Session ID: dc201432-1fdb-486b-a749-bfb70ff2fd6d

Retrieval Hint: "blog post visuals required mermaid image wall of text"

🖖 Grace (Opus 5.5, Claude Code) · session dc201432-1fdb-486b-a749-bfb70ff2fd6d


## Timeline

- 2026-11-03T17:35:00Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-03T17:35:02Z @neo-opus-grace added the `enhancement` label
- 2026-11-03T17:35:02Z @neo-opus-grace added the `Blog Post` label
- 2026-11-03T17:35:02Z @neo-opus-grace added the `ai` label
- 2026-11-03T17:35:17Z @github-actions added the `needs-design-authority` label
- 2026-11-03T17:37:51Z @neo-opus-grace cross-referenced by PR #266
- 2026-11-03T17:39:55Z @neo-opus-grace cross-referenced by PR #20226
### @neo-opus-grace - 2026-11-03T17:47:18Z

**Handover (Grace, solo-refresh sunset, 2026-11-03 ~17:50Z).** Owner: Grace. PR #266 at `c121340`: CI green, version 0.1.85, net −6 bytes. Emmy (GPT) is the requested primary reviewer. Pickup: answer her review, then hand over for the human merge. Neo picks the rule up with the next release bump.

🖖 Grace (Opus 5.5, Claude Code) · session dc201432-1fdb-486b-a749-bfb70ff2fd6d


