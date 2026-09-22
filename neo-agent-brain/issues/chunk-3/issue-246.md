---
id: 246
title: Fleet content roots are hardwired checkout-relative — the pr-lane dies post-split without a symlink
state: CLOSED
labels: []
assignees:
  - neo-fable-clio
createdAt: '2026-08-30T00:16:16Z'
updatedAt: '2026-09-22T22:50:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/246'
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
closedAt: '2026-09-22T22:50:06Z'
---
# Fleet content roots are hardwired checkout-relative — the pr-lane dies post-split without a symlink

## Problem Scope

`ai/services/fleet/devFleetServer.mjs` resolves the activity feed's PR/lane content directories at both wiring sites (`:298-299` plane mode, `:312-313` in-process) as `path.resolve(AiConfig.projectRoot, 'resources/content/{issues,pulls}')` — checkout-relative, not declared. Post-split the Brain checkout carries no `resources/content/`, so the PR/lane slot degrades (`ENOENT: scandir <brain>/resources/content/issues`) unless a symlink points at the engine's frozen tree — measured live 2026-08-30 while bringing the FM app back online, and again on 2026-09-19 and 2026-09-22 as the cockpit's `● streaming · quiet since Aug 26` header. The slot reads pulls and issues in ONE try (`wireFleetActivityReadSource.mjs`, `makeReadPrLaneSnapshot`), so re-pointing one directory alone still degrades the slot on the other.

> Body refreshed 2026-09-22 with PR #410, on @neo-gpt's R1 preflight: the 2026-08-30 version named `import.meta.url`-relative paths (dev had since moved both sites to `AiConfig.projectRoot`) and only `pullsDir`; both corrected above.

## Intended Solution

One `fleet.contentRoot` AiConfig leaf (env `NEO_FLEET_CONTENT_ROOT`, type `string`) consumed at both wiring sites for BOTH directories: `issuesDir = <root>/issues`, `pullsDir = <root>/pulls`. The default is today's checkout-relative path, so a self-contained checkout changes nothing; a deployment names the corpus it materializes — e.g. the `neo/` tree of a `neomjs/github-content-sync` checkout, whose layout the readers already walk. Single-origin by design; the multi-origin walk (`<root>/<repoSlug>/…`, origin-qualified rows) is a follow-up leaf. This is the bridge for the content-sync relocation: a re-point is one env line, not another hardwired path migration.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `AiConfig.fleet.contentRoot` — new leaf, env `NEO_FLEET_CONTENT_ROOT`, type `string` | `ai/configBase.mjs` fleet block; ADR-0019 (declarative leaf, read at the use site, no formula, no `?.`) | resolves the activity feed's content root; default `path.resolve(projectRoot, 'resources/content')` | the default IS today's path — no behavior change for a self-contained checkout | leaf JSDoc; `ai/scripts/lifecycle/local-agent-os/README.md`, host Fleet transport section | `test/playwright/unit/ai/configBase.spec.mjs` › `fleet.contentRoot` (type / env / default arm, red-first); `ai/scripts/lint/config-leaf-parity.json` carries the path |
| `devFleetServer.mjs` wiring sites — `issuesDir` + `pullsDir`, plane mode and in-process | the leaf above | both sites derive `issues` and `pulls` under `AiConfig.fleet.contentRoot`; no content literal remains | — | the wiring-site comment | spec arm "both wiring sites read the leaf at the use site" (four reads, no `resources/content/` literal); host witness against a corpus checkout: PR #19049 (2026-09-22T16:35Z), 2,285 issue records, 413 ms |
| a missing or unreadable root | `wireFleetActivityReadSource.mjs` (unchanged) | the PR/lane slot degrades naming itself; the composite reports `degraded` — never a crash, never a silent empty | — | its JSDoc | existing arm "a CONFIGURED-but-unreadable pullsDir degrades the PR/lane slot" |

## Acceptance Criteria

- [ ] Both wiring sites resolve `issuesDir` AND `pullsDir` through the leaf; default = today's checkout-relative path.
- [ ] `NEO_FLEET_CONTENT_ROOT` re-points both; a missing directory still degrades honestly (the retained-reason path — never a crash, never silent).
- [ ] The symlink workaround leaves the operator recipe: the local Agent OS README names the env and what to point it at.

## Out of Scope

The content-sync pipeline and corpus repository themselves · multi-origin reading and origin-qualified rows (a follow-up leaf) · the plane's compose value for the env (`#213`) · other content consumers (the KB's corpus tenant, `#402`).

## Related

PR #410 delivers this. Sibling of `#213` (declarative deployment; the env's plane value) and `#402` (the KB's corpus tenant, the other Brain consumer). Design authority for the FM feed as a declared-leaf consumer: neomjs/neo#17846 §2.5 and §8.7a step 3. Surfaced with neomjs/neo-agent-institution#62 during the 2026-08-30 online session.

Authored by Clio (Fable 5, Claude Code). Origin Session ID: 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604 — refreshed in session cf6c8297-03b1-41af-8d76-cb19eb9aa9c4.

Retrieval Hint: `query_raw_memories("fleet content root pulls dir env leaf ENOENT")`


## Timeline

- 2026-08-30T00:16:30Z @neo-fable-clio cross-referenced by #184
- 2026-09-21T10:40:44Z @neo-opus-vega cross-referenced by #401
- 2026-09-21T11:14:33Z @neo-opus-vega cross-referenced by #402
- 2026-09-21T14:18:02Z @neo-opus-vega cross-referenced by #406
- 2026-09-22T22:00:47Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-22T22:14:38Z @neo-fable-clio cross-referenced by PR #410
- 2026-09-22T22:25:57Z @neo-opus-vega cross-referenced by #411
- 2026-09-22T22:26:39Z @neo-fable cross-referenced by #19055
- 2026-09-22T22:50:06Z @tobiu referenced in commit `2812f94` - "Merge pull request #410 from neomjs/agent/246-fleet-content-root

feat(fleet): the activity feed reads a declared content root (#246)"
- 2026-09-22T22:50:06Z @tobiu closed this issue

