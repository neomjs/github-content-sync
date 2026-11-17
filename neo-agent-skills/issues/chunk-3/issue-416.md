---
id: 416
title: Seven skill lines name the SCSS theme sources that 14.0 removes
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - needs-design-authority
assignees:
  - neo-opus-vega
createdAt: '2026-11-17T10:31:25Z'
updatedAt: '2026-11-17T18:37:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/416'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 21217 Theme sources become plain CSS, and the theme build copies them'
blocking: []
closedAt: '2026-11-17T18:37:42Z'
---
# Seven skill lines name the SCSS theme sources that 14.0 removes

## Context

neomjs/neo#21217 (PR neomjs/neo#21225) turns the engine's theme sources into plain CSS: `resources/scss` becomes `resources/css` (`{src,theme-*}/**.css`), `build-themes` copies instead of compiling, and Sass leaves the engine. neomjs/neo#21233 carries the six guides in neo's `learn/`, neomjs/create-app#61 the generated workspace. The skills are the third reader, and nothing tracks them yet; Maryam's review of neomjs/neo#21225 (RA-2) asked for this leaf.

Live latest-open + closed sweep: checked latest 20 open + open and closed search (`scss`, `plain CSS`, `resources/css`, `styles stay`; Out of Scope read on closed #211 and #279) at 2026-11-17T10:31Z; no equivalent or prior exclusion found. Backlog band: 38 open (orange); no owning epic here, the engine epic is neomjs/neo#21213. A2A claim sweep (last 20 messages) and Memory Core (`skills instruct SCSS after the plain-CSS cut`): no in-flight or earlier leaf.

## The Problem

After neomjs/neo#21225 merges, an agent that follows these lines writes `resources/scss/...` files the theme build never reads (`buildScripts/build/themes.mjs` walks `resources/css/` only), and two of the lines are materialized into neo's `AGENTS.md:178` and `.claude/CLAUDE.md:178`, which every neo session loads.

## The Architectural Reality and The Fix

`git grep -n -i "scss\|sass"` on `dev` (`3c1096d`) finds seven lines, all instructions:

| Line | Now | After |
|---|---|---|
| `.agents/skills/neo-app-authoring/SKILL.md:3` and `.agents/skills/skills.manifest.json:102` | "…state providers, SCSS." | "…state providers, CSS." |
| `.agents/skills/neo-app-authoring/references/app-authoring-guide.md:60` | "Styles stay in SCSS. Structure goes under `resources/scss/src/apps/<app>/` … `resources/scss/theme-*/apps/<app>/`" | plain CSS under `resources/css/src/apps/<app>/` and `resources/css/theme-*/apps/<app>/`, native nesting allowed |
| `.agents/skills/pr-review/audits/demo-surface-motion-audit.md:9` | `resources/scss/_motion.scss` | `resources/css/src/Global.css` (the motion vocabulary's one declaration since neomjs/neo#21225) |
| `.agents/skills/blog-post/references/blog-authoring-guide.md:71` | `news/medium/List.scss` | `news/medium/List.css` |
| `agents-md/sections/0901-app-orientation.md:12` | "`npm run build-themes` compiles `resources/scss/`" | "copies `resources/css/`" |
| `agents-md/sections/1600-edge-case-triggers.md:14` | "styles stay in SCSS" | "styles stay in CSS" |

Regenerate what is generated from these (`skills.manifest.json`, the AGENTS.md output) with the repository's own scripts; the PR carries the next release version.

## Acceptance Criteria

- [ ] The seven lines describe plain CSS under `resources/css`, matching the Contract Ledger of neomjs/neo#21217.
- [ ] On the PR head, `git grep -n -i "scss\|sass"` returns no instruction to author, build or read SCSS.
- [ ] The manifest and the generated AGENTS.md come from their sources; `npm test` passes.
- [ ] It merges after neomjs/neo#21225, the same day or the next, so no skills release describes a layout `dev` does not have.
- [ ] Post-merge: after neo's next skills bump, neo's `AGENTS.md:178` and `.claude/CLAUDE.md:178` say CSS.

## Out of Scope

neo's `learn/` guides (neomjs/neo#21233) · the create-app scaffold (neomjs/create-app#61) · the theme build itself · the Institution's and the private package's conversions (neomjs/neo#21217 AC-7).

## Related

neomjs/neo#21217 (blocks this) · PR neomjs/neo#21225 · neomjs/neo#21233 · neomjs/create-app#61 · Epic neomjs/neo#21213 · Discussion neomjs/neo#18182 · #211 (introduced the "styles stay in SCSS" rule).

Ownership: self-assigned (Vega); not a good first issue because it waits on an unmerged engine change and ships paired with it.

Retrieval Hint: "plain CSS theme sources skills SCSS mandate 14.0"

## Timeline

- 2026-11-17T10:31:25Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-17T10:31:26Z @neo-opus-vega added the `documentation` label
- 2026-11-17T10:31:27Z @neo-opus-vega added the `enhancement` label
- 2026-11-17T10:31:27Z @neo-opus-vega added the `ai` label
- 2026-11-17T10:31:32Z @neo-opus-vega marked this issue as being blocked by #21217
- 2026-11-17T10:32:14Z @github-actions added the `needs-design-authority` label
- 2026-11-17T10:49:45Z @neo-opus-vega cross-referenced by PR #21225
- 2026-11-17T14:18:03Z @neo-opus-vega referenced in commit `ec56867` - "docs(skills): the styles rule keeps its build sentence, so the change nets fewer bytes (#416)"
- 2026-11-17T14:18:22Z @neo-opus-vega cross-referenced by PR #419
- 2026-11-17T16:40:49Z @neo-opus-vega referenced in commit `e8c42af` - "chore(release): the plain-CSS lines ship as 2.0.0, so neo-app 13.4 workspaces keep 1.10.3 (#416)

neo-app 13.4.1 scaffolds neo-agent-skills ^1.2.0 beside neo.mjs ^13.4.0, and
publish.yml releases every maintainer merge to dev. A 1.x release would hand those
workspaces resources/css/ lines that the 13.4 theme build never reads. 1.10.3 stays
the last 1.x for them; create-app #61 moves the range to ^2.0.0 with neo.mjs 14."
- 2026-11-17T18:37:42Z @tobiu referenced in commit `2aabbf7` - "docs(skills): seven lines name the plain CSS sources neo 14.0 builds (#416) (#419)

* docs(skills): seven lines name the plain CSS sources neo 14.0 builds (#416)

neomjs/neo#21225 removed resources/scss: build-themes now copies
resources/css. The neo-app-authoring description and guide, the motion
audit's token source, the blog guide's Medium card sheet and the two
AGENTS.md sections say so. Version 1.10.4.

* docs(skills): the styles rule keeps its build sentence, so the change nets fewer bytes (#416)

* chore(release): the plain-CSS lines ship as 2.0.0, so neo-app 13.4 workspaces keep 1.10.3 (#416)

neo-app 13.4.1 scaffolds neo-agent-skills ^1.2.0 beside neo.mjs ^13.4.0, and
publish.yml releases every maintainer merge to dev. A 1.x release would hand those
workspaces resources/css/ lines that the 13.4 theme build never reads. 1.10.3 stays
the last 1.x for them; create-app #61 moves the range to ^2.0.0 with neo.mjs 14."
- 2026-11-17T18:37:43Z @tobiu closed this issue

