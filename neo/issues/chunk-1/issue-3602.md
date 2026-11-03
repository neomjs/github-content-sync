---
id: 3602
title: Enhance create class build script
state: OPEN
labels:
  - enhancement
  - developer-experience
  - ai
  - build
assignees:
  - neo-gemini-maryam
createdAt: '2022-12-15T16:33:30Z'
updatedAt: '2026-11-03T17:48:00Z'
githubUrl: 'https://github.com/neomjs/neo/issues/3602'
author: Krytechs
commentsCount: 4
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
# Enhance create class build script

Is your feature request related to a problem? Please describe.
a strong dev tool helps new developers to see connections between different components of a framework

Describe the solution you'd like
When I use the create-class command line interface and want to create a store it should ask me whether I want to also implement a model.

The correct way of implementation depends on the driving part. What is the core in this store/model system which is the driving part? Depending on that this should be the master and requests if you want to add the slaves.

## Timeline

- 2022-12-15T16:33:30Z @Krytechs added the `enhancement` label
### @github-actions - 2024-08-30T02:27:37Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-30T02:27:37Z @github-actions added the `stale` label
### @github-actions - 2024-09-14T02:26:39Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-14T02:26:39Z @github-actions closed this issue
- 2026-10-09T20:23:50Z @neo-opus-grace reopened this issue
- 2026-10-09T20:23:52Z @neo-opus-grace removed the `stale` label
### @neo-opus-grace - 2026-10-09T20:23:53Z

#19489 set C · T4 · Grace · 2026-10-09: **reopened, valid but later.** Thank you @Krytechs. `create/class.mjs` still does not offer a model when generating a store, nor the reverse (#3255).

- 2026-10-09T20:26:18Z @neo-opus-grace cross-referenced by #19489
- 2026-11-03T17:47:52Z @neo-gemini-maryam added the `ai` label
### @neo-gemini-maryam - 2026-11-03T17:48:00Z

### Ticket Triage & Intake (`#3602`)

Triaged per `ticket-triage` skill. Applied: `ai`, `build`, `developer-experience` alongside `enhancement`. Stage retrospective passed. Self-assigned for implementation.

#### Validation Sweep & Prescription Check (`ticket-intake §0` / `§1`)
- **Prescription checked:** `buildScripts/create/class.mjs` — owns the concern.
- **Ticket age & stale-band:** `updatedAt: 2026-10-09T20:23:53Z` (reopened and validated by `@neo-opus-grace` in `#19489 set C · T4`, `IC_kwDODSospM8AAAABaulfBw`); `pre-stale` (`stale: false`, `no auto close: false`).
- **Currency & Successor-Risk Evidence:** `PR #20232` (`#3255`) added the `data.Model -> data.Store` companion prompt (`-m, --createStore <value>`). In the `data.Store -> data.Model` direction (`#3602`), `createClass()` (`buildScripts/create/class.mjs:546-585`) already links an existing `data.Model` file into a generated `data.Store`, but `create/class.mjs` still does not offer to create the matching `data.Model` when generating a `data.Store` whose model file does not yet exist.
- **Verdict:** `valid-as-written`.

#### Contract Ledger (`ticket-intake §1.7` sharpening)

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `buildScripts/create/class.mjs` (`-M, --createModel <value>` CLI option & interactive prompt when `baseClass === 'data.Store'`) | `#3602` + `@neo-opus-grace` `#19489 set C · T4` (`IC_kwDODSospM8AAAABaulfBw`) + sibling `#3255` (`PR #20232`) | When `baseClass === 'data.Store'` and the companion `data.Model` file does not yet exist on disk, check `-M, --createModel <value>` or prompt `Do you want to create a matching data.Model?` (when `!isDrop`). When `'yes'`, derive the singularized model name (`-ies` -> `-y`, trailing `-s` -> singular, or `<File>Model` when unchanged in the same folder) and `model` namespace (`store` -> `model` or `[...ns, 'model']`), generate the `data.Model` file, and wire `import <Model> from '<relativePath>';` + `model: <Model>` into the generated `data.Store` (including collision disambiguation when the model name collides with `Store` or the store class name). | When `'no'`, or when `-d/--drop` is used without `-M yes`, only the `data.Store` class is created (existing pre-created `data.Model` files on disk continue to be linked automatically). | CLI `--help` option description + JSDoc in `buildScripts/create/class.mjs` | Unit tests in `test/playwright/unit/buildScripts/createClassModelStore.spec.mjs` |

Authored by Maryam (Gemini 4 Argon, Antigravity). Session `f6c28a82-1400-4f2c-9d74-e81b65cb73d8`. 🧭

- 2026-11-03T17:53:48Z @neo-gemini-maryam cross-referenced by PR #20257

