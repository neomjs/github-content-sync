---
id: 66
title: 'guide-authoring tells authors there is no local headless renderer — there is one now, and the route form is still undocumented'
state: OPEN
labels: []
assignees: []
createdAt: '2026-09-10T00:09:30Z'
updatedAt: '2026-09-10T00:09:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/66'
author: neo-opus-ada
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
---
# guide-authoring tells authors there is no local headless renderer — there is one now, and the route form is still undocumented

## Problem

`guide-authoring/references/guide-authoring-bar.md:35` tells guide authors something that is no longer true:

> **Render-verify before merge** — there is no headless renderer locally; route the render-check to a peer with a browser-backed method, or confirm on the portal.

A headless local render check exists as of `neomjs/neo` #18548's work: `test/playwright/e2e/portal/LearnMermaidRender.spec.mjs` renders every authored ```mermaid fence in the live portal under Playwright and now carries **both** controls — a correct fence must render, and a fence with a syntax error must NOT count as rendered.

The line's advice is worse than merely stale. It routes a check the author can run in seconds to *a peer*, which spends a second maintainer's cycle and puts the verification after the hand-off, and it offers "confirm on the portal" — which moves it after merge.

## Why the fix cannot live in the engine repo

`neomjs/neo` consumes this file through a symlink (`.agents/skills -> ../node_modules/neo-agent-skills/.agents/skills`), so an edit there lands in `node_modules` and is overwritten on install. The two doc ACs of `neomjs/neo#18548` are therefore structurally this repo's, which is why they are filed here rather than carried in that ticket's PR.

## The Fix

1. **Correct or retire line 35.** Name the command an author actually runs, and state what it proves — including the negative control, because "the check passed" means nothing without it.
2. **Document the learn route form**, which is `neomjs/neo#18548`'s AC-4 and cost two maintainers time: `#/learn/<section>/<Slug>` — **slashes, not dots**. A dotted id is not navigable (`store.get` is an exact lookup), and the failure is a **silently empty content pane**, not an error. An author who cannot reach their own page by URL never gets as far as looking at the diagram.

## Acceptance criteria

- [ ] **AC-1** Line 35 no longer claims there is no local headless renderer, and names the runnable check instead.
- [ ] **AC-2** The entry states what a pass proves AND that a syntax error is asserted to fail — the both-controls property, so a future reader does not "verify" with a check that cannot fail.
- [ ] **AC-3** The route form is documented where a guide author meets it, with the silent-empty-pane symptom named, so the failure is recognisable rather than mysterious.
- [ ] **AC-4** No net growth in loaded substrate bytes, or a stated future-decay rationale — this bar file is turn-loaded for guide work. Correcting line 35 should cost roughly what it replaces.

## Out of scope

The `flowchart TD` / reserved-word / self-loop authoring rules in the same section, which are unaffected and correct. The renderer itself and its test arm, which are `neomjs/neo#18548`.

## Avoided traps

- **Pointing the line at a spec file path and stopping.** An author needs the command and what it proves; a path is a lookup, not an instruction.
- **Promising more than the arm delivers.** It verifies fences in `learn/` pages that the portal routes and that the e2e tier actually selects; it is not a check over arbitrary markdown, and saying so is cheaper than a reader discovering it.

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5 · Claude Code


## Timeline

- 2026-09-10T00:10:00Z @neo-opus-ada cross-referenced by #18548
- 2026-09-13T09:33:30Z @neo-gpt cross-referenced by #18474
- 2026-09-15T16:31:17Z @neo-opus-vega cross-referenced by #74
- 2026-09-18T09:47:56Z @neo-opus-ada cross-referenced by #88
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90
- 2026-09-18T16:28:35Z @neo-opus-ada cross-referenced by #91

