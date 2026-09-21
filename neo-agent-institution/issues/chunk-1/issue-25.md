---
id: 25
title: Give Agent Institution its product front door
state: CLOSED
labels:
  - documentation
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-gpt
createdAt: '2026-08-27T11:12:43Z'
updatedAt: '2026-08-27T14:05:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/25'
author: neo-gpt
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
closedAt: '2026-08-27T13:33:52Z'
---
# Give Agent Institution its product front door

## Context

`neomjs/neo-agent-institution` now owns the Agent Institution application, Electron harness, visual assets, and 103-spec product test suite. The receive is merged at `dev@48fcc8e`, but the repository has **no root `README.md`**, no GitHub description/homepage/topics, and therefore no public product front door.

The shipped repository already contains the materials a truthful README needs: local Neo logo assets, a full-cockpit visual golden, `server-start`, isolated unit/component/E2E/visual commands, Institution CI, the browser app at `apps/agentos/index.html`, and an explicit cross-repository Brain contract. Memory Core prior art and D#10119 establish the operator app as the product wedge: the app is the starting point for an institution; the repository is its source. D#17247 makes **Agent Institution** the outward repository/category identity while keeping fleet vocabulary internal to the subsystem.

Structure map: N/A — this ticket creates only the root README and introduces no module or `.mjs` placement.

## The Problem

A cold visitor cannot determine:

- what the Institution product is or how it relates to the Neo.mjs organism;
- that the **Agent Institution app**, not a source checkout, is the intended starting point for operating an agent team;
- how Engine, Brain, Institution, DevIndex, and Skills divide ownership;
- which browser workflow works today;
- why ordinary contributors need no Neo maintainer credentials or Brain checkout for isolated development;
- which Brain-connected and Electron/package paths remain transitional.

The absence also hides the strongest product proof already in the repository: a visual, tested cockpit with roster, activity, account, task, memory, mailbox, wake, instance, and multi-window surfaces.

## The Architectural Reality

Institution is the operator-facing product layer:

- **Engine / Body** arrives through the immutable `neo.mjs` package dependency.
- **Brain / Agent OS** remains a sibling runtime reached through explicit transport and, for checkout/test/harness integration, an absolute `NEO_AGENTOS_RUNTIME_ROOT`; Brain source is never copied here or discovered through cwd/sibling guessing.
- **Agent Institution** is the application that starts, observes, and operates an institution. Its source currently lives at `apps/agentos` for compatibility; internal fleet module names do not define the outward product identity.
- **Skills** arrive through `neo-agent-skills` materialization; public contributors consume those rules without inheriting maintainer tokens, identities, or private service configuration.

Browser development and isolated CI are shipped. The native harness source is present, but post-split packaging/launcher completion belongs to #4. The README must not advertise a finished downloadable application or hide that boundary.

## The Fix

Create one root `README.md`:

1. Use the repository-local light/dark Neo logo and Institution CI / Node / license / contribution badges.
2. Show the repository-local full-cockpit visual golden as the product hero.
3. Preserve the parent Neo.mjs organism narrative while projecting the Agent Institution product: persistent identities, shared memory, cross-family review, and operator control for teams running on their own projects.
4. Add the five-repository bulleted map; each bullet starts with its repository link, and Institution alone ends with plain-text **← You are here** outside the link.
5. Explain shipped Agent Institution surfaces and the Engine ↔ Institution ↔ Brain boundary.
6. Document the current browser quickstart (`npm install`, `npm run server-start`, `apps/agentos/index.html`) and the isolated test commands.
7. Separate contributor mode (no Brain/credentials required for isolated checks) from maintainer/full-contract mode (explicit absolute Brain root and local credentials).
8. Link the harness documentation while stating #4 owns post-split packaging/launcher completion.
9. Link the canonical organization Introduction and the current Engine/Brain/Skills owners rather than duplicating their detailed documentation.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| README apex | canonical Introduction + D#10119 + D#17247 | Parent organism first; Agent Institution operator product second | Preserve exact apex wording where no segmentation is needed | root README | framing comparison |
| product hero | repository visual golden + local logo assets | Show the tested Fleet cockpit without cross-repo hotlinks | Omit an image only if the local golden is removed with an explicit successor | root README | local path + render check |
| organization map | live public repositories + operator direction | Five links, one Institution marker outside the link | None: missing sibling is incomplete | root README | URL/status check |
| browser quickstart | `package.json#server-start` + `apps/agentos/index.html` | Starts the current web product from a clean checkout | Missing command/route blocks | root README | clean-install + route check |
| Brain-connected mode | Institution CI + #4 | Absolute explicit Brain root; no copied Brain/cwd fallback; packaging still transitional | Isolated contributor workflow remains available | root README / harness README | CI contract + boundary audit |

## Decision Record impact

Aligned-with ADR 0018 identity governance, the operator-app-first product direction from D#10119 / ADR 0020, and the outward Agent Institution naming boundary in D#17247. No ADR change.

## Acceptance Criteria

- [ ] README uses repository-local light/dark logo assets and at least Institution CI, Node 24+, MIT, and contribution badges; no cross-repository image dependency.
- [ ] A repository-local full-cockpit screenshot is visible near the product opening with useful alt text.
- [ ] The opening explains Neo.mjs as the parent organism and Agent Institution as the operator product, including the trust/institution value for teams running on their own projects.
- [ ] Exactly five primary repositories are linked with accurate one-line ownership; every bullet starts with repository text, and Institution alone ends with plain-text **← You are here** outside its link.
- [ ] README distinguishes the app as the team starting point from the repository as source, and lists only capabilities evidenced by current source/tests.
- [ ] Browser quickstart commands and the `apps/agentos/index.html` route exist and are readable from a fresh checkout.
- [ ] Contributor-isolated mode, explicit Brain-connected mode, credential boundaries, and #4 Electron/package transition are separated without publishing secret values or claiming unfinished packaging.
- [ ] Unit, component, E2E, visual, and CI doors are documented; all local README targets exist and all new public targets resolve.
- [ ] Cross-family identity review completes before merge.

## Out of Scope

`package.json` identity metadata · GitHub description/topics/homepage · #2/#3 product bug fixes · #4 packaging/launcher implementation · application/source/test changes · Brain or Engine changes · secret/config materialization · Portal/Pages/SEO work · downloadable release claims.

## Avoided Traps

- Copying the Brain README and describing services instead of the operator product.
- Telling users to “start sessions from this repo” when the app is the intended control surface.
- Advertising Electron distribution before #4 closes the post-split pack/launcher boundary.
- Showing Neo maintainer tokens, identities, or private `.env` values as contributor prerequisites.
- Hotlinking the parent repository logo or an external screenshot when local canonical assets exist.

## Related

Receive baseline: #1  
Known product repairs: #2 · #3  
Packaging/launcher boundary: #4  
Engine source removal: neomjs/neo#17805  
Product direction: D#10119

Origin Session ID: a901c4fe-1816-4bc1-a1a2-7a983ec2a9cd

Retrieval Hint: `Agent Institution README product front door app starts institution browser quickstart explicit Brain root`

Authored by Euclid (OpenAI GPT-5.6 Sol, Codex Desktop). Session a901c4fe-1816-4bc1-a1a2-7a983ec2a9cd.


## Timeline

- 2026-08-27T11:12:44Z @neo-gpt added the `documentation` label
- 2026-08-27T11:12:45Z @neo-gpt added the `enhancement` label
- 2026-08-27T11:12:45Z @neo-gpt added the `agent-os` label
- 2026-08-27T11:12:45Z @neo-gpt added the `ai` label
- 2026-08-27T11:12:45Z @neo-gpt added the `design` label
- 2026-08-27T11:13:14Z @neo-gpt assigned to @neo-gpt
- 2026-08-27T11:19:47Z @neo-gpt changed title from **Give Fleet Manager its product front door** to **Give Agent Institution its product front door**
- 2026-08-27T11:25:25Z @tobiu cross-referenced by PR #26
- 2026-08-27T13:33:52Z @tobiu referenced in commit `e4498fb` - "Merge pull request #26 from neomjs/codex/25-agent-institution-readme

docs(readme): give Agent Institution its front door (#25)"
- 2026-08-27T13:33:52Z @tobiu closed this issue
- 2026-08-27T14:23:45Z @neo-gpt cross-referenced by #27
- 2026-08-27T14:28:29Z @tobiu cross-referenced by PR #28
- 2026-09-04T18:29:58Z @neo-fable-clio cross-referenced by #64
- 2026-09-04T18:31:47Z @neo-fable-clio cross-referenced by PR #110

