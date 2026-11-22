---
id: 161
title: Add docs/public-manifest.json and publish lint for DevIndex guides
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gemini-julia
createdAt: '2026-11-22T00:25:57Z'
updatedAt: '2026-11-22T01:55:25Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/161'
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
  - '[ ] 158 Materialize pinned spoke learn manifests in updateNeoVersion.mjs'
closedAt: '2026-11-22T01:55:25Z'
---
# Add docs/public-manifest.json and publish lint for DevIndex guides

## Context

Graduated leaf of `neomjs/neo` Discussion #20920 (`A0 + A1` Multi-Repo Guide Restore Slice, `neomjs/neo#21847`). Before `neomjs/devindex` split out of `neomjs/neo` in commit `715d48d101` (`2026-08-20`), its `26` guides were published on `neomjs.com` under `/learn/guides/devindex/*` — including `/learn/guides/devindex/data-factory/Engine`, which earned Page-1 Google Search impressions (`pos 10.0`) and currently returns `HTTP 404`.

## The Problem

`neomjs/devindex` already maintains `learn/tree.json` (`26` leaf guides), but lacks the `A1` file-level default-deny `docs/public-manifest.json` and CI publish lint required for `neomjs/pages` and `neomjs/middleware-v2` to federate its guides back onto `neomjs.com` at their exact pre-split `/learn/guides/devindex/*` URLs.

## The Architectural Reality

- `neomjs/devindex/learn/tree.json` defines `26` leaf nodes (`isLeaf !== false`) whose relative `.md` files live under `learn/<id>.md` (e.g. `learn/data-factory/Engine.md`).
- In `neomjs/neo` `715d48d101^:learn/tree.json` (and `#20498`), those `26` leaves mount under `Guides` > `"DevIndex App"` at `guides/devindex/<id>`, corresponding to canonical URLs `/learn/guides/devindex/<id>`.
- `neomjs/devindex/buildScripts/` houses standalone Node verification scripts (`buildScripts/checkDataTracking.mjs`) invoked in `.github/workflows/ci.yml`, and `test/playwright/playwright.config.unit.mjs` runs unit specs in CI (`--project=unit --fail-on-flaky-tests`).

## The Fix

- **Pre-Flight (structural fast-path):** Authoring `buildScripts/lintPublicDocs.mjs` matches sibling pattern of `buildScripts/checkDataTracking.mjs` in `buildScripts/`; authoring `test/playwright/unit/buildScripts/LintPublicDocs.spec.mjs` matches sibling specs under `test/playwright/unit/`; sibling-file-lift applies; no novel directory choice.
1. Author `docs/public-manifest.json` (`schema: "public-manifest.v1"`, `spoke: "neomjs/devindex"`, `treeFile: "learn/tree.json"`, `treeMount: { parentId: "Guides", id: "guides/devindex", name: "DevIndex App" }`) enumerating all `26` `learn/tree.json` leaves with `{ "file": "learn/<id>.md", "route": "/learn/guides/devindex/<id>" }`.
2. Author `buildScripts/lintPublicDocs.mjs` (`npm run lint-public-docs`, zero ticket `#N` refs in source comments) and wire it into `.github/workflows/ci.yml`:
   - Verifies `1:1` set equality between `docs/public-manifest.json` entries and `learn/tree.json` leaves, that every `file` exists under `learn/`, and that every `route` starts with `/learn/guides/devindex/` with no duplicates or traversal segments.
   - Scans every manifest-listed Markdown file and fails closed on real host paths (`/Users/<user>`, `/home/<user>` — allowing generic placeholders like `/Users/<you>`), private-repo identifiers (`neomjs/business`, `neomjs/neo-stage`, `neomjs/neo-pro`), and broken relative `.md` links, while permitting public `@neo-*` handles.
3. Add unit spec `test/playwright/unit/buildScripts/LintPublicDocs.spec.mjs` covering clean pass + synthetic host-path, private-repo, and tree/manifest drift rejections.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `docs/public-manifest.json` | Discussion #20920 `A0 + A1` (`OQ-1`, `OQ-2`, `#discussioncomment-19803727`) | Default-deny JSON manifest mapping all `26` `learn/tree.json` leaves to `/learn/guides/devindex/<id>` (`Guides` > `"DevIndex App"`) | Build fails closed if manifest or target `.md` is missing | JSDoc in `buildScripts/lintPublicDocs.mjs` | Unit spec + CI |
| `buildScripts/lintPublicDocs.mjs` + `.github/workflows/ci.yml` | Discussion #20920 `OQ-1` | CLI + CI step rejecting manifest/tree drift, real host paths, private-repo refs, and broken relative links | Non-zero exit blocks PR CI | `package.json` script `lint-public-docs` | `npx playwright test -c test/playwright/playwright.config.unit.mjs --project=unit` |

## Decision Record & Discussion Criteria Mapping

- **Decision Record:** `Optional: neomjs/neo#20920`
- **Decision Record impact:** `aligned-with ADR 0040`

<!-- binding-inventory: a0-a1-spokes -->
- `neomjs/devindex` — `26` pre-split guides at `/learn/guides/devindex/*` via `docs/public-manifest.json` + existing `learn/tree.json`
<!-- /binding-inventory -->

## Signal Ledger

- `gemini`: `AUTHOR_SIGNAL` by `@neo-gemini-julia` @ `body updatedAt 2026-11-22T00:56:38Z` (`DC_kwDODSospM4BLi_P`)
- `claude`: `APPROVED` (`@neo-fable-clio` `DC_kwDODSospM4BLjAi` & `@neo-opus-grace` `DC_kwDODSospM4BLjBN` @ `2026-11-22T00:56:38Z`)

## Unresolved Dissent

None.

## Unresolved Liveness

- `gpt` (`active`, `wake: unknown`) and `kimi` (`operator_benched`); cross-family PR review applies on `neomjs/devindex#162`.

## Acceptance Criteria

- [ ] `docs/public-manifest.json` lists all `26` `learn/tree.json` leaves mapped to their exact pre-split `/learn/guides/devindex/<id>` routes (including `/learn/guides/devindex/data-factory/Engine`, `treeMount.name: "DevIndex App"`).
- [ ] `buildScripts/lintPublicDocs.mjs` (`npm run lint-public-docs`) is wired into `.github/workflows/ci.yml` without path-filter starvation and enforces manifest↔`learn/tree.json` parity, file existence, route hygiene, host-path / private-repo rejection, and relative-link validity.
- [ ] Playwright unit spec verifies both the live `neomjs/devindex` manifest pass and synthetic rejection fixtures (`host path`, `private repo`, `unlisted tree leaf`).

## Out of Scope

- `neomjs/pages` and `neomjs/middleware-v2` consumption of `docs/public-manifest.json` (tracked in sibling leaves under `neomjs/neo#21847`).
- Modifying `/devindex/` app UI or data-sync workflows.

## Avoided Traps

- **Mounting DevIndex guides at `/learn/devindex/*` instead of `/learn/guides/devindex/*`:** forces unnecessary `301` re-consolidation on `/learn/guides/devindex/data-factory/Engine` (`pos 10.0` in GSC).
- **Putting ticket/discussion `#N` citations inside source comments:** fails `Shared PR Baseline / Source comment archaeology`.

## Sweep Attestations & Provenance

- `Live latest-open + closed sweep`: checked all `3` open issues (`#135`, `#61`, `#9`) + open and closed search in `neomjs/devindex` at `2026-11-22T00:17Z` (green backlog band: `3` open); no duplicate found.
- `A2A in-flight claim sweep` + `MC sweep` + `Own-assignment sweep`: verified clean at `2026-11-22T00:17Z`.
- Ownership: self-assigned to `@neo-gemini-julia` (PR `neomjs/devindex#162`).

Related: `neomjs/neo#21847` · Source Discussion https://github.com/neomjs/neo/discussions/20920

Origin Session ID: 664ca229-4bbd-4e00-ae41-008707688181
Retrieval Hint: `devindex docs/public-manifest.json 26 guides /learn/guides/devindex/ 715d48d101 lintPublicDocs`

## Timeline

- 2026-11-22T00:25:58Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-22T00:26:00Z @neo-gemini-julia added the `documentation` label
- 2026-11-22T00:26:00Z @neo-gemini-julia added the `enhancement` label
- 2026-11-22T00:26:00Z @neo-gemini-julia added the `ai` label
- 2026-11-22T00:26:00Z @neo-gemini-julia added the `architecture` label
- 2026-11-22T00:26:43Z @neo-gemini-julia added parent issue #21847
- 2026-11-22T00:28:34Z @neo-gemini-julia marked this issue as blocking #158
- 2026-11-22T00:35:04Z @neo-gemini-julia cross-referenced by PR #162
- 2026-11-22T00:46:31Z @neo-gemini-julia referenced in commit `8ff3411` - "docs(buildScripts): remove tracking refs from lintPublicDocs JSDoc (#161)"
- 2026-11-22T00:57:32Z @neo-gemini-julia referenced in commit `a2d61b4` - "feat(docs): align treeMount name to DevIndex App (#161)"
- 2026-11-22T01:42:24Z @neo-gemini-julia referenced in commit `463267a` - "fix(docs): enforce manifest-admitted link targets and file:// host-path rejection (#161)"
- 2026-11-22T01:55:25Z @tobiu referenced in commit `fd98ff5` - "feat(docs): add public-manifest.json and publish lint for DevIndex guides (#161) (#162)

* feat(docs): add public-manifest.json and publish lint for DevIndex guides (#161)

* docs(buildScripts): remove tracking refs from lintPublicDocs JSDoc (#161)

* feat(docs): align treeMount name to DevIndex App (#161)

* fix(docs): enforce manifest-admitted link targets and file:// host-path rejection (#161)"
- 2026-11-22T01:55:26Z @tobiu closed this issue

