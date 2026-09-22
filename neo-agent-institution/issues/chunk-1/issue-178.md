---
id: 178
title: Add a learn/ tree with the first Fleet Manager guides
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
assignees: []
createdAt: '2026-09-22T22:32:13Z'
updatedAt: '2026-09-22T22:32:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/178'
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
---
# Add a learn/ tree with the first Fleet Manager guides

## Context

Operator, 2026-09-23 (paired session): *"while `neomjs/neo-agent-institution` does not yet have a learn folder, this is something we want to add rather sooner than later."* Measured the same minute: this repository tracks eight markdown files outside `node_modules` — `README.md`, `harness/README.md`, `src/main/addon/README.md`, three design contracts under `apps/agentos/` (`CARD-CONTRACT.md`, `TOKENS.md`, `VisualSystem.md`), two vendored highlight.js files — and no `learn/`. The engine (`learn/tree.json` + 149 files) and devindex (`learn/tree.json` + 27 files) both carry one; the Brain carries `learn/` (117 files) without a tree manifest. `neomjs/neo#19047`, the v13.2 deployment epic, already counts three `learn/` trees for the portal's union; this repository is the fourth, and the union design should count it now rather than discover it later.

## The Problem

The Institution is the operator-facing product, and its only reader-facing prose is the README. What the cockpit shows (roster, activity, tasks, memories, mailbox, wake, catch-up, perspectives), how to run it against a Brain (the host Fleet transport, the plane binding, the `NEO_FLEET_*` variables), and what the native vessel adds (`harness/`) is spread over READMEs written for contributors, Brain-side guides (`learn/agentos/cloud-deployment/*`, `ai/scripts/lifecycle/local-agent-os/README.md`) and ticket bodies. An operator who downloads the app has no guide, the Knowledge Base has no Institution guide to ingest, and the portal's learn section cannot render what does not exist.

## The Architectural Reality

- The learn-tree contract is the engine's: `learn/tree.json` is `{"data": [{name, parentId, id, isLeaf?}]}`, where `id` is the markdown path without its extension (engine `learn/tree.json:1-4`; devindex mirrors the shape exactly). Its consumers today are the portal's learn section plus the engine's `buildScripts/docs/seo/generate.mjs` and `buildScripts/util/check-relative-links.mjs`. The cross-repository union of the trees is `neomjs/neo#19047` / [D#19051](https://github.com/neomjs/neo/discussions/19051), not this ticket.
- The Brain's `learn/` has no manifest and is read by agents and the Knowledge Base, not the portal. The Institution's tree is reader-facing product documentation, so it follows the engine/devindex shape, not the Brain's.
- The authoring bar is the `guide-authoring` skill: grounding by memory-mining and by using the subsystem's own tools before writing, render-verified TD Mermaid, conceptual-vs-reference separation, never hand-editing generated files.
- Seed material with authority today: `README.md` (what the Institution is, the organization map, what the cockpit operates), `harness/README.md` (the native vessel), the three `apps/agentos/*.md` design contracts (developer contracts — they stay where they are and get linked, never moved), and the Brain's local Agent OS README (the host Fleet transport section).

## The Fix

1. `learn/tree.json` and `learn/README.md` in the engine/devindex shape.
2. The first guides, each a leaf in the tree and each written through `guide-authoring`:
   - `Introduction.md` — what the Institution is and is not, from the README's "What is the Institution?" and the organization map; one TD Mermaid of Body · Brain · Institution.
   - `CockpitTour.md` — the cockpit's surfaces and what each state word means: roster cards, the activity header (`● streaming · quiet since …`), tasks, memories, mailbox, wake, catch-up, perspectives, instance switching, the connection banners — read from the running app, not inferred from code.
   - `RunningTheInstitution.md` — dev server vs the native vessel; the host Fleet transport binding (`NEO_FLEET_PLANE_BASE`, `NEO_FLEET_PLANE_BEARER`, `NEO_FLEET_CONTENT_ROOT`) with the Brain's README as the linked authority, one paragraph of orientation here.
   - `TheNativeVessel.md` — the `harness/` role from `harness/README.md`: what ships in the vessel and what deliberately does not.
3. `README.md` links the tree; relative links pass.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `learn/tree.json` | the engine's `learn/tree.json` shape | rows `{name, parentId, id}`; every `id` resolves to a markdown file | none (new surface) | this ticket | JSON parses; a resolve check over every `id` |
| `learn/*.md` | `guide-authoring` skill | reader-facing guides; Mermaid TD render-verified | — | the guides themselves | relative-link check passes |
| `README.md` | this repository | links the learn tree | — | README | the link resolves |

## Decision Record impact

none — documentation; the cross-repository union's authority is `neomjs/neo#19047` / D#19051.

## Acceptance Criteria

- [ ] `learn/tree.json` exists in the engine/devindex shape and every `id` resolves to a markdown file under `learn/`.
- [ ] The four guides above exist, each authored through `guide-authoring` (grounded in the live app or the owning tool, not written from memory), with every Mermaid diagram render-verified.
- [ ] Nothing moves out of `apps/agentos/*.md` or `harness/README.md`; the guides link to them wherever a developer contract is the authority.
- [ ] `README.md` links the learn tree, and a relative-link check over `learn/**` passes (the engine's `check-relative-links` pointed at this tree, or this repository's equivalent if one exists by then).
- [ ] Post-merge only: `neomjs/neo#19047` / D#19051 count this tree in the portal union — their acceptance criterion, referenced there by comment when this ticket is filed.

## Out of Scope

The cross-repository union and how the portal renders four trees (`neomjs/neo#19047`, D#19051) · a first-run or setup guide before [D#18965](https://github.com/neomjs/neo/discussions/18965) ships its path · moving the design contracts · Knowledge Base admission of this repository's `learn/**` (the tenant extraction profile under `neomjs/neo-agent-brain#402` owns it).

## Avoided Traps

- Copying the Brain's cloud-deployment prose into this tree: two copies drift within a week; link the authority, keep one paragraph of orientation.
- Writing the cockpit tour from the source: the `guide-authoring` bar is the live app (dev server or Neural Link), so state words and banners are read from the running surface.
- A README-only documentation story: the portal, the Knowledge Base and the release film all read trees, not READMEs.

## Related

`neomjs/neo#19047` (v13.2 deployment epic, the `learn/` union) · [D#19051](https://github.com/neomjs/neo/discussions/19051) (what the portal renders after the split) · #10 (cockpit UI/UX epic) · #12 (native shell UX specification) · #19 (release video; the tour guide and the screenplay share one source) · [D#18965](https://github.com/neomjs/neo/discussions/18965) (first-run journey).

Live latest-open sweep: checked the latest 20 open issues of this repository at 2026-09-22T22:29Z; no equivalent. A2A in-flight sweep (last 30 messages, all read-states): no claim on an Institution `learn/` lane; the learn-folder MERGE was named unowned by @neo-opus-vega at 22:01Z and lives inside `neomjs/neo#19047` / D#19051 — a different object. Memory sweep (`query_raw_memories` on the problem's nouns): no prior decision. Own-assignment sweep: #10 only, no overlap. Structure map: N/A (documentation, no `.mjs` file).

unowned-rationale: four guides through `guide-authoring` is an Opus- or GPT-sized lane this week (the Fable pair reviews little and drives short lanes); the FM lead (@neo-fable-clio) answers content and design questions here and takes the lane in a fresh session if nobody claims it before the v13.2 cut.

Authored by Clio (Claude Fable 5.1, Claude Code).
Origin Session ID: cf6c8297-03b1-41af-8d76-cb19eb9aa9c4
Retrieval Hint: `query_raw_memories("Institution learn tree first Fleet Manager guides tree.json")`


## Timeline

- 2026-09-22T22:32:14Z @neo-fable-clio added the `documentation` label
- 2026-09-22T22:32:14Z @neo-fable-clio added the `enhancement` label
- 2026-09-22T22:32:15Z @neo-fable-clio added the `ai` label
- 2026-09-22T22:32:30Z @neo-fable-clio cross-referenced by #19047

