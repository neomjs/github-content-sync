---
id: 1891
title: 'Add Brain docs/public-manifest.json, learn/tree.json, and publish lint'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - testing
  - architecture
  - build
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-22T00:26:03Z'
updatedAt: '2026-11-22T05:35:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1891'
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
closedAt: '2026-11-22T05:35:10Z'
---
# Add Brain docs/public-manifest.json, learn/tree.json, and publish lint

## Context

Graduated leaf of `neomjs/neo` Discussion #20920 (`A0 + A1` Multi-Repo Guide Restore Slice, `neomjs/neo#21847`). `11` of the `14` pre-split `/learn/*` URLs currently returning `HTTP 404` in `sc-domain:neomjs.com` (including `7` Page-1 rankings at `pos 3.0–9.0` such as `/learn/agentos/tooling/Introduction`, `/learn/agentos/SharedDeployment`, `/learn/agentos/DeploymentCookbook`, and `/learn/agentos/cloud-deployment/PipelineWiring`, plus `6` `301` targets under `/learn/benefits/brain/*`) live in `neomjs/neo-agent-brain/learn/`.

## The Problem

`neomjs/neo-agent-brain/learn/` contains `125` Markdown files mixing multi-tenant product documentation with `50` operational records (`decisions/`, `measurements/`, `incidents/`), has **no `learn/tree.json`**, and contains real `/Users/<name>` paths in `2` product guides (`learn/agentos/CodeExecution.md` and `learn/agentos/OwnAgentTeam.md`). Per `@neo-fable-clio`'s and `@neo-opus-grace`'s reconciled count ([#discussioncomment-19803727](https://github.com/neomjs/neo/discussions/20920#discussioncomment-19803727), [#discussioncomment-19804237](https://github.com/neomjs/neo/discussions/20920#discussioncomment-19804237)):
1. Across every `neomjs/neo` `learn/tree.json` revision since `2026-06-01`, `67` former navigation leaves sit in `neomjs/neo-agent-brain/learn/`, of which `5` are internal records (`0001-cross-process-cache-coherence`, `0019-aiconfig-reactive-provider-ssot`, `0022-heavy-maintenance-scheduling-fairness`, `heartbeat-token-economy-2026-05`, `pr-review-baseline-2026-04`) excluded by `A1` default-deny and assigned `410` rows in `neomjs/middleware-v2#104`.
2. `13` of the `75` files outside `decisions/`, `measurements/`, and `incidents/` never held a pre-split nav entry (`process/` 4, `wake-substrate/` 3, `IdentitySchema`, `ModelStats`, `v13-path`, `tooling/MCPServerBaseClass`, `tooling/MultiTenantMigrationGuide`, `tooling/Tier2RevalidationSweep`) and stay unlisted in this `A0` restore slice as Stage-2 expansion.
3. That leaves **exactly `62` pre-split Brain product guides** (`23` Agent OS + `18` Tooling & Configuration + `15` Deploying the Agent OS + `6` `benefits/brain/*`) returning at their exact pre-split `/learn/agentos/*` and `/learn/benefits/brain/*` URLs (`0` redirects).

## The Architectural Reality

- `neomjs/neo-agent-brain/.github/workflows/brain-lint.yml` runs repo-local lint scripts under `ai/scripts/lint/` (e.g. `ai/scripts/lint/lint-identity-vocabulary.mjs`), guarded by `lintWorkflowScanRootParity.spec.mjs` (which asserts every path scanned by a `brain-lint.yml` step is covered by `pull_request.paths` and `push.paths` in equal sets).
- Adding a lint scanning `docs/public-manifest.json` and `learn/{agentos,benefits}/**` requires adding `'docs/public-manifest.json'`, `'learn/tree.json'`, `'learn/agentos/**'`, and `'learn/benefits/**'` identically to both `pull_request.paths` and `push.paths` in `.github/workflows/brain-lint.yml` so both `lintWorkflowScanRootParity.spec.mjs` and `OpenApiServiceParityTriggerReachability.spec.mjs` (`'learn/README.md'` negative control) stay green.

## The Fix

- **Pre-Flight (structural fast-path):** Authoring `ai/scripts/lint/lint-public-docs.mjs` matches sibling pattern of `ai/scripts/lint/lint-identity-vocabulary.mjs` in `ai/scripts/lint/` (confirmed via `node ai/scripts/diagnostics/structureMap.mjs --files --loc`); authoring unit spec under `test/playwright/unit/ai/scripts/lint/` matches sibling script-lint specs; sibling-file-lift applies; no novel directory choice.
1. Scrub the real `/Users/<name>` filesystem paths in `learn/agentos/CodeExecution.md` and `learn/agentos/OwnAgentTeam.md` to portable `<workspace>` / `$HOME` placeholders.
2. Seed `learn/tree.json` (`4` groups matching `#20498`: `"The Brain - Agent OS"` `6` under `Benefits`, plus `Agent OS` `23`, `Tooling & Configuration` `18`, and `Deploying the Agent OS` `15` under `AgentInterfaces`) and `docs/public-manifest.json` (`schema: "public-manifest.v1"`, `spoke: "neomjs/neo-agent-brain"`, `treeFile: "learn/tree.json"`, `treeMounts`, `excludedDirs`, `sourceLinkRoots: ["ai/"]`) admitting the exact **`62`** pre-split Brain product nav leaves at `/learn/agentos/*` and `/learn/benefits/brain/*` while excluding the `50` `decisions/`/`measurements/`/`incidents/` files and `13` net-new files.
3. Author `ai/scripts/lint/lint-public-docs.mjs` (zero ticket/discussion `#N` refs in source comments) and wire it into `.github/workflows/brain-lint.yml` (passing `NEO_PUBLIC_DOCS_FORBIDDEN_TERMS: ${{ secrets.NEO_PUBLIC_DOCS_FORBIDDEN_TERMS }}`), failing closed on manifest/`treeMounts`↔`learn/tree.json` drift, real host paths, private-repo identifiers (`business`, `neo-stage`, `neo-pro`), externally supplied client-name terms (reported by 1-based position without echoing the term, with checked term count printed on every run), and relative links outside admitted guides or committed `ai/**` paths.
4. Add unit test coverage verifying the live `62`-entry Brain manifest/tree pass and synthetic host-path, private-repo, positional forbidden-term (no echo), and manifest/tree/link drift rejections.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `learn/tree.json` + `docs/public-manifest.json` | Discussion #20920 `A0 + A1` (`OQ-1`, `OQ-2`, `#discussioncomment-19803727`) + `neomjs/neo-agent-skills#434` | Declares `treeMounts` (`4` folder mounts matching folder nodes in `learn/tree.json`), `excludedDirs` (`learn/agentos/{decisions,measurements,incidents}`), and `sourceLinkRoots: ["ai/"]`; restores exact `62` pre-split Brain product nav leaves (`23 + 18 + 15 + 6`) at `/learn/...` while excluding `50` ADR/measurement/incident + `13` net-new files | Downstream `pages` materializer fails closed on missing file, `excludedDirs` violation, or route collision | JSDoc in `ai/scripts/lint/lint-public-docs.mjs` | Unit spec + `brain-lint.yml` |
| `ai/scripts/lint/lint-public-docs.mjs` + `.github/workflows/brain-lint.yml` | Discussion #20920 `OQ-1` + `AGENTS.md §critical_gates` #9 | Install-free CI lint rejecting host paths, private-repo refs, `treeMounts`/tree drift, and relative links outside manifest-admitted guides or committed `ai/**` source paths (excluding gitignored `config.mjs` overlays); reads client-name terms from `NEO_PUBLIC_DOCS_FORBIDDEN_TERMS` (`secrets.NEO_PUBLIC_DOCS_FORBIDDEN_TERMS` in `brain-lint.yml`), prints the checked term count on every run, and reports matches by 1-based position without echoing the term | Non-zero exit reds `Brain Lint` | `package.json` `ai:lint-public-docs` | Unit spec + `lintWorkflowScanRootParity.spec.mjs` |

## Decision Record & Discussion Criteria Mapping

- **Decision Record:** `Optional: neomjs/neo#20920`
- **Decision Record impact:** `aligned-with ADR 0040` and `neomjs/neo-agent-brain#10`

<!-- binding-inventory: a0-a1-spokes -->
- `neomjs/neo-agent-brain` — `62` pre-split product guides at exact `/learn/agentos/*` & `/learn/benefits/brain/*` routes via `docs/public-manifest.json` + new `learn/tree.json` (`50` ADR/measurement/incident files & `13` net-new files excluded)
<!-- /binding-inventory -->

## Signal Ledger

- `gemini`: `AUTHOR_SIGNAL` by `@neo-gemini-julia` @ `body updatedAt 2026-11-22T00:56:38Z` (`DC_kwDODSospM4BLi_P`)
- `claude`: `APPROVED` (`@neo-fable-clio` `DC_kwDODSospM4BLjAi` & `@neo-opus-grace` `DC_kwDODSospM4BLjBN` @ `2026-11-22T00:56:38Z`)

## Unresolved Dissent

None.

## Unresolved Liveness

- `gpt` (`active`, `wake: unknown`) and `kimi` (`operator_benched`); cross-family PR review applies on the implementation PR.

## Acceptance Criteria

- [ ] `learn/agentos/CodeExecution.md` and `learn/agentos/OwnAgentTeam.md` contain zero real `/Users/<name>` host paths.
- [ ] `learn/tree.json` and `docs/public-manifest.json` admit the exact `62` pre-split Brain product nav leaves (`23` Agent OS + `18` Tooling & Configuration + `15` Deploying the Agent OS + `6` `benefits/brain/*`, covering all `11` exact-URL Brain GSC routes + all `6` `benefits/brain/*` `301` targets) while excluding the `50` `decisions/`/`measurements/`/`incidents/` files and `13` net-new files.
- [ ] `ai/scripts/lint/lint-public-docs.mjs` is wired into `.github/workflows/brain-lint.yml` (`pull_request.paths` == `push.paths`, `lintWorkflowScanRootParity.spec.mjs` green) and covered by Playwright unit tests.

## Out of Scope

- Modifying `ai/config*` (`ADR-0019` untouched) or Brain MCP runtime services.
- Publishing the `13` net-new Brain guides that never held a pre-split nav URL (deferred to Stage-2 expansion).

## Avoided Traps

- **Confusing `75` non-record files or `67` historical nav leaves with the `62` restore set:** `13` of the `75` are net-new expansion and `5` of the `67` are excluded ADRs/measurements (`410` in `middleware-v2#104`), leaving `62` ([#discussioncomment-19803727](https://github.com/neomjs/neo/discussions/20920#discussioncomment-19803727)).
- **Updating `brain-lint.yml` steps without updating `pull_request.paths` + `push.paths` identically:** breaks `lintWorkflowScanRootParity.spec.mjs`.
- **Putting ticket/discussion `#N` citations inside JSDoc/source comments:** fails `Shared PR Baseline / Source comment archaeology`.

## Sweep Attestations & Provenance

- `Live latest-open + closed sweep`: checked latest 20 open (`161` open total — red backlog band) + open and closed search in `neomjs/neo-agent-brain` at `2026-11-22T00:17Z`; no duplicate found.
- `A2A in-flight claim sweep` + `MC sweep` + `Own-assignment sweep` + `ai:structure-map`: verified clean at `2026-11-22T00:17Z`.
- `unowned-rationale:` Open for peer self-select in the spoke wave (`@neo-gemini-julia` / `@neo-opus-grace`).

Related: `neomjs/neo#21847` · `neomjs/neo-agent-brain#10` · Source Discussion https://github.com/neomjs/neo/discussions/20920

Origin Session ID: 664ca229-4bbd-4e00-ae41-008707688181
Retrieval Hint: `neo-agent-brain docs/public-manifest.json learn/tree.json 62 nav leaves lint-public-docs brain-lint.yml`

## Timeline

- 2026-11-22T00:26:04Z @neo-gemini-julia added the `documentation` label
- 2026-11-22T00:26:04Z @neo-gemini-julia added the `enhancement` label
- 2026-11-22T00:26:04Z @neo-gemini-julia added the `ai` label
- 2026-11-22T00:26:05Z @neo-gemini-julia added the `testing` label
- 2026-11-22T00:26:05Z @neo-gemini-julia added the `architecture` label
- 2026-11-22T00:26:05Z @neo-gemini-julia added the `build` label
- 2026-11-22T00:26:05Z @neo-gemini-julia added the `agent-os` label
- 2026-11-22T00:26:45Z @neo-gemini-julia added parent issue #21847
- 2026-11-22T00:28:35Z @neo-gemini-julia marked this issue as blocking #158
- 2026-11-22T01:34:21Z @neo-opus-grace cross-referenced by PR #162
- 2026-11-22T01:48:06Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-22T02:02:45Z @neo-gemini-julia cross-referenced by PR #1892
- 2026-11-22T02:03:58Z @neo-opus-grace cross-referenced by #434
- 2026-11-22T02:05:31Z @neo-opus-grace cross-referenced by PR #1037
- 2026-11-22T02:08:54Z @neo-gemini-julia referenced in commit `9e4696b` - "docs(lint): document helper signatures and options in lint-public-docs (#1891)"
- 2026-11-22T02:47:58Z @neo-gemini-julia referenced in commit `91a6115` - "fix(docs): fix IdentityRitualsCulture links and CI forbidden-term guard (#1891)"
- 2026-11-22T03:18:13Z @neo-opus-grace cross-referenced by #1893
- 2026-11-22T03:18:37Z @neo-opus-grace marked this issue as blocking #1893
- 2026-11-22T03:32:19Z @neo-opus-grace removed the block on #1893
- 2026-11-22T05:35:09Z @tobiu referenced in commit `23ce0b2` - "feat(docs): add Brain public-manifest.json, learn/tree.json, and publish lint (#1891) (#1892)

* feat(docs): add Brain public-manifest.json, learn/tree.json, and publish lint (#1891)

* docs(lint): document helper signatures and options in lint-public-docs (#1891)

* fix(docs): fix IdentityRitualsCulture links and CI forbidden-term guard (#1891)"
- 2026-11-22T05:35:11Z @tobiu closed this issue

