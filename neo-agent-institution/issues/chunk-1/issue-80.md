---
id: 80
title: 'A roster record without avatarUrl renders a broken image: the card has no avatar fallback'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-02T12:24:08Z'
updatedAt: '2026-09-02T15:38:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/80'
author: neo-fable-clio
commentsCount: 0
parentIssue: 10
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-02T15:38:51Z'
---
# A roster record without avatarUrl renders a broken image: the card has no avatar fallback

Sub-issue of #10 (the cockpit UI/UX epic). Seen on 2026-09-02 while proving #76 on the bumped engine (PR #79); filed on the operator's word so it does not get lost.

## Context

The AgentCard composes its avatar as a `Neo.component.Image` (`apps/agentos/view/fleet/roster/card/Container.mjs:124-127`, `cls: ['fm-card-avatar']`, `reference: 'card-avatar'`) and binds it on every record sync: `me.getReference('card-avatar').set({… src: record.avatarUrl ?? null})` (`:477-479`). The engine's `Image#afterSetSrc` writes the `src` attribute through `changeVdomRootKey`, so a `null` URL renders an `<img alt="<displayName>">` **with no `src`** — and Chrome paints its broken-image glyph plus the alt text in the 80px slot.

## The Problem

Read on the live cockpit (Neural Link, three fixture rows without `avatarUrl`): each card shows Chrome's broken-image icon over the first letters of the alt text (`Ama`, `Cle`), in the avatar's circle. The card has **no fallback of its own** — no initials monogram, no family-colored placeholder: `grep -n "initials\|monogram\|fallback" Container.mjs` finds nothing; the `.fm-card-avatar` SCSS (`resources/scss/src/apps/agentos/fleet/roster/card/Container.scss:47, :330, :358`) sizes the image per width mode and nothing else.

Production rows carry the GitHub face, so the seed roster never shows it. It shows the moment a record has no face: an agent defined through the S5 form before its GitHub avatar resolves (#74's journey), a registry entry without a `githubUsername`, a fleet server that cannot reach GitHub — exactly the first-run moments the demo walks through.

The AgentCard synthesis witness (`AgentCardSynthesisRenderNL`) names "the operator avatar-keeper invariant: a visible avatar at every card width (real image slot)" and satisfies it with a data-URI SVG per row — the fixture supplies what the product does not.

## The Architectural Reality

- `Neo.component.Image` renders an `<img>` unconditionally; hiding it is the component's `hidden` config (`removeDom`), not a `src` trick.
- The card's identity column is card-owned (`@container` width modes: 32px avatar under 320px, the regular slot above); a fallback must render in the same slot at the same sizes, or the width modes drift.
- The family rail already carries a per-family ink (`FamilyRail`); a monogram on that ink is the visual vocabulary the card already speaks.

## The Fix

In the card (`roster/card/Container.mjs` + its SCSS), the avatar slot becomes a keeper:

1. When `record.avatarUrl` is empty: the `Image` is hidden (`hidden: true`) and a sibling monogram element in the same slot shows the display name's initials (two letters max, upper-cased) on the family ink, sized by the same width-mode rules as the image. When the URL is present: the image shows, the monogram is hidden. One reactive sync, both directions (a record may gain a face after registration).
2. The `<img>` keeps its `alt` for the loaded case; a src-less `<img>` never reaches the DOM.
3. Optional (if cheap): `Image` load failure (`error` event) falls back to the monogram too — a 404 face is the same defect from the other side.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| `card-avatar` reference (`Container.mjs:124`) | this ticket | hidden when `avatarUrl` is empty | — | class JSDoc (anatomy) | component/e2e arm |
| new monogram element in the avatar slot | this ticket; #15536 (the card's design) | initials on family ink, same width-mode sizes | empty name → the family glyph alone | same | same arm + the synthesis witness's width matrix |
| `AgentCardSynthesisRenderNL` avatar-keeper invariant | existing | unchanged: a visible avatar at every width — now true for src-less rows too | — | — | add one src-less row to the pathological roster; goldens re-captured with a design read |

## Acceptance Criteria

- [ ] A record without `avatarUrl` renders no `<img>` in its card and shows a two-letter monogram on the family ink, at every card width of the synthesis matrix (294 / 319 / 320 / 328 / 360 / 720), both themes. Red on `dev` (the `<img>` with no `src` is in the DOM today).
- [ ] A record that gains an `avatarUrl` after registration switches to the image; one that loses it switches back (store `set` on the live record; DOM read).
- [ ] `AgentCardSynthesisRenderNL` gains one src-less row; the twelve goldens are re-captured with a design read (the golden diff is the review surface, per PR #71's rule).
- [ ] Unit / visual / `check-app-file-sizes` green; `check-visual-baselines` stamped.

## Out of Scope

- Fetching or resolving GitHub faces (the registry's concern).
- The avatar's size tokens and the width modes (#15536 settled them).

## Related

Parent: #10. The card's design: #15536 (recompose for responsive hierarchy). The journey that shows it first: #74. Seen during #76 / PR #79.

Live latest-open sweep: checked the latest 20 open institution issues at 2026-09-02T12:21Z — none on the avatar or a card fallback; A2A claims of the last hour: none on the card; `ask_knowledge_base` (ticket) surfaced #15536 (the card's composition ticket) and no fallback leaf.

Origin Session ID: 91f83b9c-df95-4f72-a68f-d33f470792ac

Retrieval Hint: "AgentCard avatar fallback monogram src-less img broken image family ink"

📜 Clio

## Timeline

- 2026-09-02T12:24:09Z @neo-fable-clio added the `bug` label
- 2026-09-02T12:24:10Z @neo-fable-clio added the `agent-os` label
- 2026-09-02T12:24:10Z @neo-fable-clio added the `ai` label
- 2026-09-02T14:46:01Z @neo-fable-clio cross-referenced by #81
- 2026-09-02T14:57:27Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-02T15:28:16Z @neo-fable-clio referenced in commit `0fffedd` - "chore(test): refresh the visual baseline stamp over the staged keeper inputs (#80)

The stamp digests staged blob ids; the previous stamp was taken before the second commit's inputs were staged, so it recorded the prior index. Stage, then stamp, then commit — the order the checker's contract names."
- 2026-09-02T15:28:45Z @neo-fable-clio cross-referenced by PR #83
- 2026-09-02T15:38:51Z @tobiu referenced in commit `8bfca6b` - "Merge pull request #83 from neomjs/agent/80-avatar-monogram-fallback

fix(agentos): a faceless roster record shows a family-inked monogram — the avatar keeper never mounts a src-less <img> (#80)"
- 2026-09-02T15:38:52Z @tobiu closed this issue

