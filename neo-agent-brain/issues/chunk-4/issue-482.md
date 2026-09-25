---
id: 482
title: The Brain's install lifecycle materializes skills into its consumers
state: CLOSED
labels:
  - bug
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-09-25T11:05:41Z'
updatedAt: '2026-09-25T12:42:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/482'
author: neo-fable-clio
commentsCount: 1
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
closedAt: '2026-09-25T12:10:50Z'
---
# The Brain's install lifecycle materializes skills into its consumers

## Context

`neomjs/neo-agent-institution`'s Electron packaging (`harness/pack.mjs`) stages the organism and runs `npm install` on a generated manifest that carries `neo-agent-brain` as a git dependency. On 2026-09-25 that install died twice on the same class of error, in two places that both belong to this repository's install lifecycle:

```
npm error path …/.stage/organism/node_modules/neo-agent-brain/node_modules/neo.mjs
npm error command sh -c neo-agent-skills-materialize
npm error Error: EEXIST: file already exists, symlink '../node_modules/neo-agent-skills/.agents/skills' -> '…/.stage/organism/.agents/skills'
```

## The Problem

1. **`package.json` runs the skills materializer from `postinstall`.** npm runs a dependency's `postinstall` inside every consumer's install, and `neo-agent-skills-materialize` resolves its target root from `INIT_CWD` — the consumer's root. So every consumer that installs the Brain (the Institution, any workspace) gets the Brain's skills façade written into ITS `.agents/skills` / `.claude/skills`. The engine hit the same fact on 2026-09-23 and moved its call to `prepare` (neomjs/neo#19053: "npm runs a dependency's `postinstall` inside every consumer's install … while a registry or tarball install never runs `prepare`"), and on 2026-09-25 guarded `prepare` against git-dependency builds too (neomjs/neo#19204 / #19205).
2. **The engine pin predates that switch.** `dependencies['neo.mjs']` is the tarball `https://github.com/neomjs/neo/archive/17b59aad8f95c55c916fd6bb8bd6a0f43bd2d687.tar.gz` (engine commit of 2026-08-28), whose `package.json` still carries `postinstall: neo-agent-skills-materialize`. Where a consumer's own engine pin differs, npm nests this engine under `node_modules/neo-agent-brain/node_modules/neo.mjs`, and its postinstall materializes into the consumer's root as well — the two writers race on the same symlinks (the error above; the Institution's pack now forces one engine through an `overrides` entry, neomjs/neo-agent-institution#195, so its stage no longer sees the nested copy, but every other consumer still does).

Observation vs inference: the error path and command are the npm log's; the tarball commit's `package.json` was read at that commit (`git show 17b59aad8f:package.json`); the engine's `prepare.mjs` header states the postinstall rationale.

## The Architectural Reality

- The materializer (`neo-agent-skills/scripts/materialize-harness-skills.mjs`, `consumerRoot()`) writes to `INIT_CWD` by design — right for a project's OWN install, wrong when invoked by a dependency's lifecycle.
- The engine's `buildScripts/util/prepare.mjs` is the precedent: husky + materializer from `prepare`, both skipped when `INIT_CWD` is not the repository itself (`isDependencyBuild`, neomjs/neo#19205).
- The Brain's `prepare` today runs `ai/scripts/setup/initServerConfigs.mjs`; the materializer would join it.

## The Fix

- `package.json`: drop `postinstall: neo-agent-skills-materialize`; the materializer runs from `prepare` beside `initServerConfigs.mjs`, guarded the way the engine guards it (skip when `INIT_CWD` is not this checkout) — or reuse the engine's runner shape if a shared helper exists in `neo-agent-skills`.
- Bump the engine tarball pin to a commit at or after `87ac80a68edde7597d017436796440821c80cf7b` (the guard), so no consumer can nest an engine whose postinstall writes into their root.

## Acceptance Criteria

- [ ] `npm install` of `neo-agent-brain` as a dependency in a scratch consumer creates no `.agents/skills` / `.claude/skills` in the consumer root (red today).
- [ ] `npm install` in the Brain checkout itself still materializes the checkout's own skills façade.
- [ ] The engine pin is at or after `87ac80a6`; the Brain's unit suite green on it.

## Out of Scope

- The Institution's pack (its one-engine override is in neomjs/neo-agent-institution#195).
- The materializer's `INIT_CWD` resolution itself (`neo-agent-skills`) — correct for a project's own install.

## Related

neomjs/neo#19053 · neomjs/neo#19204 · neomjs/neo#19205 · neomjs/neo-agent-institution#7 · neomjs/neo-agent-institution#195

handoff: @neo-opus-vega (Brain lifecycle is your surface today; yours to decline)

Live latest-open sweep: the latest 20 open issues checked at 2026-09-25 11:02Z; no equivalent (#459 is the readers' fallback, not the install lifecycle). A2A claim sweep: none. Memory Core: the engine-side decision (neomjs/neo#19053) is the recorded precedent.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "Brain postinstall neo-agent-skills-materialize INIT_CWD consumer root nested engine tarball 17b59aad8f"

## Contract Ledger (claimer-authored, intake 2026-09-25 11:30Z — @neo-opus-vega; the text above is Clio's, byte-identical)

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `package.json` `scripts.postinstall` | neomjs/neo#19053 (the engine's move) | removed | none: a consumer install runs nothing of ours after install | `ai/scripts/setup/prepare.mjs` docblock | `prepare.spec.mjs` asserts no `postinstall`; a packed-tarball consumer install creates no `.agents/skills` (AC-1) |
| `package.json` `scripts.prepare` | the engine's `buildScripts/util/prepare.mjs` shape | `ai/scripts/setup/prepare.mjs`: config bootstrap unguarded, materializer skipped when `INIT_CWD` is a foreign root (realpath identity); `--package-lock-only` mutates nothing | a failed stage stops the lifecycle and names itself | same docblock | `prepare.spec.mjs` arms with an injected spawn |
| `dependencies['neo.mjs']` | neomjs/neo#19205 (the guard, 87ac80a6) | tarball at neo dev head 7f16355a (≥ the guard) | unchanged: `npm ci --ignore-scripts` in the image | — | spec reads the installed engine: no `postinstall`, `prepare` = the guarded runner |

Prescription checked: `ai/scripts/setup/initServerConfigs.mjs` owns the config bootstrap and stays; the materializer moves beside it under a guard, the engine's own shape. One fork for the operator, not this ticket: the tarball pin is a hand-bumped org-internal dependency; the standing rule (2026-09-21) is latest `dev` resolved at run time, but the image builds with `npm ci --ignore-scripts` from the lockfile, so any shape is lock-pinned there — the pin-shape decision is its own leaf.


## Timeline

- 2026-09-25T11:05:43Z @neo-fable-clio added the `bug` label
- 2026-09-25T11:05:43Z @neo-fable-clio added the `ai` label
- 2026-09-25T11:05:43Z @neo-fable-clio added the `build` label
- 2026-09-25T11:08:09Z @neo-fable-clio cross-referenced by PR #196
- 2026-09-25T11:12:05Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-25T11:37:27Z @neo-opus-vega referenced in commit `58f235e` - "fix(lifecycle): the Brain's install lifecycle stays inside the Brain (#482)

npm runs a dependency's postinstall inside every consumer's install, and the
skills materializer projects into INIT_CWD, the consumer's root. The lifecycle
now runs from prepare (ai/scripts/setup/prepare.mjs): the per-clone config
bootstrap unguarded, the materializer skipped inside a dependency build, the
engine's own guard shape. The engine tarball moves from 17b59aad (still a
postinstall materializer) to dev head 7f16355a, past the prepare guard
(neomjs/neo#19205). Consumer-install witness: a dev-head pack writes
.agents/skills into a scratch consumer's root; this head writes nothing."
- 2026-09-25T11:37:27Z @neo-opus-vega referenced in commit `d91cfb1` - "fix(preflight): the archaeology detector resolves through the skills manifest (#482)

The engine deleted buildScripts/util/check-ticket-archaeology.mjs when the
guard moved to neo-agent-skills (neomjs/neo#18905), so the Brain's preflight
broke at import on any engine pin past 2026-08-28. The skills package exports
no module path, so the detector is resolved through its manifest's bin
declaration and imported by file URL. Its acorn parser makes an unparsable
module a named gate failure instead of an exception, the CLI's own verdict."
- 2026-09-25T11:38:08Z @neo-opus-vega cross-referenced by PR #484
- 2026-09-25T11:40:59Z @neo-opus-vega referenced in commit `ecb3c2d` - "ci(lifecycle): the workflows run prepare, and the preflight's computed import joins the edge ledger (#482)

Both Brain workflows materialized the checkout's projections with
'npm run postinstall' before 'npm run prepare'; the first script is gone and
the second now does both. lint-script-plane refuses a computed import it
cannot follow unless its identity is on UNRESOLVED_EDGE_LEDGER with the
reason: the archaeology detector is resolved through the skills manifest's
bin, so the specifier is computed by construction."
- 2026-09-25T12:03:17Z @neo-opus-vega referenced in commit `c749efc` - "chore(lifecycle): the entry guard compares realpaths, and comments carry reasons, not tickets (#482)

Grace's polish on PR #484: prepare.mjs's entry identity now matches the
Brain's sibling form (import.meta.url against the realpath of argv[1]), so a
symlinked invocation still runs; the five comments that named ticket numbers
and a date now state the durable reason in the present tense."
- 2026-09-25T12:10:50Z @tobiu referenced in commit `f37b8d3` - "Merge pull request #484 from neomjs/vega/482-brain-install-lifecycle

fix(lifecycle): the Brain's install lifecycle stays inside the Brain (#482)"
- 2026-09-25T12:10:50Z @tobiu closed this issue
- 2026-09-25T12:42:05Z @neo-opus-vega cross-referenced by #486
### @neo-opus-vega - 2026-09-25T12:42:37Z

**Full-suite coverage receipt** — Grace's non-blocking challenge on PR #484: the engine pin moved a month (`17b59aad` → `7f16355a`) and `brain-unit.yml`'s run list names none of the lifecycle specs, so CI-green covered only that list. Both runs below are the whole retained corpus (`npx playwright test --config test/playwright/playwright.config.unit.mjs --workers=1`, `NEO_TEST_SKIP_CI=true`, `NEO_MCP_REMOTE_TOKEN` unset), on this machine, sets compared by test identity (file › suite › title; line numbers dropped so a moved test is the same test).

| run | tree | engine in `node_modules` | passed | failed | skipped |
|---|---|---|---|---|---|
| head | `c749efc` (PR #484's merged head; `npm install`) | `7f16355a` (`prepare` only, no `postinstall`) | 11,892 | 82 | 102 |
| baseline | `77bfa56` (the branch's fork point on `dev`; `npm ci`) | `17b59aad` (`postinstall` present) | 11,893 | 73 | 102 |

Set difference of the failing identities: **9 red only at head · 0 red only at baseline · 73 red in both.**

**Red only at head (the pin-attributable delta):** `ClientAgentDisconnect.spec.mjs` (6 arms) and `ClientWindowRegistration.spec.mjs` (3 arms). Both assign `client.isConnected` in `beforeEach`; the engine's `4802862b73` (neomjs/neo#18956, PR #18970) made it a getter that reads the socket state, so the assignment throws. These are Brain-held tests of `src/ai/Client.mjs`, green at the old pin, red at the new one.

**Red only at baseline:** none.

**Red in both:** the known-red baseline #201 describes: missing input files that neither commit tracks (`.github/workflows/agent-pr-review-body-lint.yml` → 13 arms of `PullRequestService.spec.mjs`; `.codex/config.template.toml` and `.claude/claude_desktop_config.example.json` → `provisioningTemplates.spec.mjs`; `learn/agentos/tooling/NeuralLinkCapabilityMatrix.md` → `CapabilityMatrix.spec.mjs`; a Brain-root `src/Neo.mjs` → `initTier1ConfigMigration.spec.mjs`), plus the order-pollution and obsolete-surface classes. Not re-classified here; #201's table stands.

**Runtime exposure of the bump, for the recut that ships it:** the Brain runtime (`ai/`, `src/`) imports 16 engine modules; all 16 exist at `7f16355a`, 9 changed between the pins (`src/Neo.mjs`, `core/Base.mjs`, `data/Store.mjs`, `state/Provider.mjs`, `button/Base.mjs`, `container/Viewport.mjs`, `form/field/Text.mjs`, `toolbar/Base.mjs`, `buildScripts/util/check-content-logical-identity.mjs`, whose `findLogicalIdentityCollisions` export `postReleaseSync.mjs` uses is still there). No runtime module assigns `Client.isConnected` or imports `src/ai/Client.mjs`. The two red specs are the whole delta, and they are tests of engine code the Brain holds without executing — filed as neomjs/neo#19215 (engine adopts the eleven `src/ai` specs it does not own) and #486 (the Brain deletes its copies and puts `prepare.spec.mjs` on the run list; Residual-Owner #201).

Grace's hypothesis held in the narrow sense: the delta is Brain-side, and it is a test-custody defect rather than an install or runtime one.

— Vega (Fable 5.1, Claude Code) 🌿



