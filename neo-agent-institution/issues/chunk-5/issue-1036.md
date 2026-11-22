---
id: 1036
title: Add docs/public-manifest.json and publish lint for Institution guides
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
  - build
  - testing
assignees:
  - neo-gemini-julia
createdAt: '2026-11-22T00:26:00Z'
updatedAt: '2026-11-22T05:34:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1036'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 21847
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 158 Materialize pinned spoke learn manifests in updateNeoVersion.mjs'
closedAt: '2026-11-22T05:34:21Z'
---
# Add docs/public-manifest.json and publish lint for Institution guides

## Context

Graduated leaf of `neomjs/neo` Discussion #20920 (`A0 + A1` Multi-Repo Guide Restore Slice, `neomjs/neo#21847`). `neomjs/neo-agent-institution` contains `12` Markdown files under `learn/`: `7` product guides already structured in `learn/tree.json` (`README`, `Introduction`, `CockpitTour`, `CockpitStateCensus`, `CockpitStateWalkthrough`, `RunningTheInstitution`, `TheNativeVessel`) and `5` ADRs under `learn/agentos/decisions/*.md` (`0020`, `0032`, `0034`, `0037`, `0038`, whose relative paths overlap `neomjs/neo/learn/agentos/decisions/*`), none of which are currently federated onto `neomjs.com/learn/`.

## The Problem

Without a file-level default-deny `docs/public-manifest.json` and CI publish lint in `neomjs/neo-agent-institution`, `neomjs/pages` and `neomjs/middleware-v2` cannot publish the `7` Agent Institution product guides under `/learn/institution/*` (`AgentInterfaces` shelf in `#20498`, labeled `"Agent Institution"` per ADR 0018 OD-4) while keeping the `5` `learn/agentos/decisions/*.md` ADRs default-deny unlisted (`A1`).

## The Architectural Reality

- `neomjs/neo-agent-institution/learn/tree.json` already lists the `7` product leaves (`isLeaf !== false`) and excludes the `5` ADR files under `learn/agentos/decisions/`.
- Per Discussion #20920 `OQ-2`, `#20498`, and ADR 0018 OD-4 ([#discussioncomment-19803727](https://github.com/neomjs/neo/discussions/20920#discussioncomment-19803727), [#discussioncomment-19804194](https://github.com/neomjs/neo/discussions/20920#discussioncomment-19804194)), Institution's `7` product guides take `/learn/institution/<id>` and mount as one group under the hub's `AgentInterfaces` root (`treeMount: { parentId: "AgentInterfaces", id: "institution", name: "Agent Institution" }`). Question-level overlap with `neomjs/neo`'s two cockpit guides is reconciled via `#20527`'s one-owner-per-question table in `neomjs/neo#21848`.
- `buildScripts/checkAppFileSizes.mjs` provides the standalone guard script precedent wired into `.github/workflows/ci.yml` (`isolated` job).

## The Fix

- **Pre-Flight (structural fast-path):** Authoring `buildScripts/lintPublicDocs.mjs` matches sibling pattern of `buildScripts/checkAppFileSizes.mjs` in `buildScripts/`; authoring `test/playwright/unit/buildScripts/LintPublicDocs.spec.mjs` matches sibling specs under `test/playwright/unit/`; sibling-file-lift applies; no novel directory choice.
1. Author `docs/public-manifest.json` (`schema: "public-manifest.v1"`, `spoke: "neomjs/neo-agent-institution"`, `treeFile: "learn/tree.json"`, `treeMount: { parentId: "AgentInterfaces", id: "institution", name: "Agent Institution" }`) mapping the `7` `learn/tree.json` product guides to `/learn/institution/<id>` while leaving `learn/agentos/decisions/*.md` unlisted.
2. Author `buildScripts/lintPublicDocs.mjs` (`npm run lint-public-docs`, zero ticket `#N` refs in source comments) and wire it into `.github/workflows/ci.yml`:
   - Enforces `1:1` parity between `docs/public-manifest.json` and `learn/tree.json` leaves, file existence, `/learn/institution/*` route prefixing, and zero ADR/operational file leakage unless explicitly listed.
   - Fails closed on real host filesystem paths (`/Users/<user>`, `/home/<user>`), private-repo identifiers (`neomjs/business`, `neomjs/neo-stage`, `neomjs/neo-pro`), and broken relative `.md` links across manifest-admitted files.
3. Add Playwright unit spec `test/playwright/unit/buildScripts/LintPublicDocs.spec.mjs` verifying live manifest pass + synthetic host-path / private-repo / unlisted-tree-leaf failures.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `docs/public-manifest.json` | Discussion #20920 `A0 + A1` (`OQ-1`, `OQ-2`, `#discussioncomment-19804194`) | Default-deny manifest admitting `7` `learn/tree.json` product guides at `/learn/institution/<id>` (`parentId: "AgentInterfaces"`, `name: "Agent Institution"`, `5` ADRs excluded) | Build fails closed on missing file or parity drift | JSDoc in `buildScripts/lintPublicDocs.mjs` | Unit spec + CI |
| `buildScripts/lintPublicDocs.mjs` + `.github/workflows/ci.yml` | Discussion #20920 `OQ-1` | Spoke CI publish lint enforcing manifest↔tree parity and privacy/path hygiene | Non-zero exit blocks Institution CI | `package.json` `lint-public-docs` script | `npm run test-unit` |

## Decision Record & Discussion Criteria Mapping

- **Decision Record:** `Optional: neomjs/neo#20920`
- **Decision Record impact:** `aligned-with ADR 0040` & `ADR 0018 OD-4`

<!-- binding-inventory: a0-a1-spokes -->
- `neomjs/neo-agent-institution` — `7` product guides at `/learn/institution/*` via `docs/public-manifest.json` + existing `learn/tree.json` (`5` ADRs excluded)
<!-- /binding-inventory -->

## Signal Ledger

- `gemini`: `AUTHOR_SIGNAL` by `@neo-gemini-julia` @ `body updatedAt 2026-11-22T00:56:38Z` (`DC_kwDODSospM4BLi_P`)
- `claude`: `APPROVED` (`@neo-fable-clio` `DC_kwDODSospM4BLjAi` & `@neo-opus-grace` `DC_kwDODSospM4BLjBN` @ `2026-11-22T00:56:38Z`)

## Unresolved Dissent

None.

## Unresolved Liveness

- `gpt` (`active`, `wake: unknown`) and `kimi` (`operator_benched`); cross-family PR review applies on the implementation PR.

## Acceptance Criteria

- [ ] `docs/public-manifest.json` maps all `7` `learn/tree.json` product guides (`README`, `Introduction`, `CockpitTour`, `CockpitStateCensus`, `CockpitStateWalkthrough`, `RunningTheInstitution`, `TheNativeVessel`) to `/learn/institution/<id>` (`parentId: "AgentInterfaces"`, `name: "Agent Institution"`) and excludes the `5` `learn/agentos/decisions/*.md` ADRs.
- [ ] `buildScripts/lintPublicDocs.mjs` (`npm run lint-public-docs`) runs in `.github/workflows/ci.yml` (`isolated` job) and passes on all `7` product guides while rejecting synthetic host-path, private-repo, and manifest/tree drift fixtures in unit tests.

## Out of Scope

- Publishing `learn/agentos/decisions/*.md` ADRs onto `neomjs.com`.
- `neomjs/pages` and `neomjs/middleware-v2` consumption (tracked in sibling leaves under `neomjs/neo#21847`).

## Avoided Traps

- **Whole-folder `learn/**` copy:** leaks `5` internal ADRs that collide with `neomjs/neo/learn/agentos/decisions/{0020,0032,0034,0037,0038}`.
- **Putting ticket/discussion `#N` citations inside source comments:** fails `Shared PR Baseline / Source comment archaeology`.

## Sweep Attestations & Provenance

- `Live latest-open + closed sweep`: checked latest 20 open (`59` open total — red backlog band) + open and closed search in `neomjs/neo-agent-institution` at `2026-11-22T00:17Z`; no duplicate found.
- `A2A in-flight claim sweep` + `MC sweep` + `Own-assignment sweep`: verified clean at `2026-11-22T00:17Z`.
- `unowned-rationale:` Open for peer self-select in the spoke wave (`@neo-gemini-julia` / `@neo-opus-grace`).

Related: `neomjs/neo#21847` · Source Discussion https://github.com/neomjs/neo/discussions/20920

Origin Session ID: 664ca229-4bbd-4e00-ae41-008707688181
Retrieval Hint: `neo-agent-institution docs/public-manifest.json 7 guides /learn/institution/ Agent Institution lintPublicDocs`

## Timeline

- 2026-11-22T00:26:02Z @neo-gemini-julia added the `documentation` label
- 2026-11-22T00:26:02Z @neo-gemini-julia added the `enhancement` label
- 2026-11-22T00:26:02Z @neo-gemini-julia added the `ai` label
- 2026-11-22T00:26:02Z @neo-gemini-julia added the `architecture` label
- 2026-11-22T00:26:03Z @neo-gemini-julia added the `build` label
- 2026-11-22T00:26:03Z @neo-gemini-julia added the `testing` label
- 2026-11-22T00:26:44Z @neo-gemini-julia added parent issue #21847
- 2026-11-22T00:28:34Z @neo-gemini-julia marked this issue as blocking #158
- 2026-11-22T01:28:33Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-22T01:34:21Z @neo-opus-grace cross-referenced by PR #162
- 2026-11-22T01:38:00Z @neo-gemini-julia cross-referenced by PR #1037
- 2026-11-22T01:43:21Z @neo-gemini-julia referenced in commit `bfdb2c0` - "fix(docs): catch file:// host paths and derive route prefix from treeMount.id (#1036)"
- 2026-11-22T02:03:58Z @neo-opus-grace cross-referenced by #434
- 2026-11-22T02:07:55Z @neo-gemini-julia referenced in commit `16d7813` - "docs(build): restore complete lintPublicDocs JSDoc and add treeMount.id drift spec (#1036)"
- 2026-11-22T05:34:21Z @tobiu referenced in commit `ff63776` - "feat(docs): add public-manifest.json and publish lint for Institution guides (#1036) (#1037)

* feat(docs): add public-manifest.json and publish lint for Institution guides (#1036)

* fix(docs): catch file:// host paths and derive route prefix from treeMount.id (#1036)

* docs(build): restore complete lintPublicDocs JSDoc and add treeMount.id drift spec (#1036)"
- 2026-11-22T05:34:22Z @tobiu closed this issue

