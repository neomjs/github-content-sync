---
id: 149
title: Six dev-scope Dependabot alerts and three CodeQL findings stay open
state: CLOSED
labels:
  - bug
  - ai
  - dependencies
  - security
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-18T10:25:55Z'
updatedAt: '2026-09-18T11:25:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/149'
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
blocking:
  - '[x] 151 Engine pin 10 — dev@70c2c94618: 88 commits, theme weight + Monaco build'
closedAt: '2026-09-18T11:22:46Z'
---
# Six dev-scope Dependabot alerts and three CodeQL findings stay open

## Context

Operator goal, 2026-09-18: the Security tab reads zero before Fleet Manager app work resumes. Live reads at 10:15Z (`gh api …/dependabot/alerts?state=open`, `…/code-scanning/alerts?state=open`):

| Surface | Alerts | Package / rule | Tracked by |
|---|---|---|---|
| Dependabot | 28, 29, 30, 31 | `dompurify` ≤ 3.4.12 (dev) | this ticket |
| Dependabot | 26, 27 | `lodash-es` ≤ 4.17.23 (dev) | this ticket |
| Dependabot | 23, 25 | `sharp` < 0.35.4 (runtime) | #124 |
| CodeQL | 1, 2, 3 | `js/bad-code-sanitization`, `test/playwright/unit/harness/brain.spec.mjs:109/116/117` | this ticket |

## The Problem

**An engine pin bump closes none of them.** npm reads `overrides` from the root manifest only, so the engine's own repairs (`lodash-es ^4.18.0`, `monaco-editor > dompurify ^3.4.13` in `neomjs/neo` `package.json`) never reach this lock. The Institution declares `mermaid` and `monaco-editor` itself because the engine's build scripts resolve them from the workspace (`buildScripts/build/all.mjs` spawns `mermaid.mjs` and `monaco.mjs` unconditionally).

Lock at dev `4c874ea`:

- `monaco-editor@0.56.0` declares `dompurify: 3.4.8` (exact) → nested `node_modules/monaco-editor/node_modules/dompurify@3.4.8`; the hoisted copy is already 3.4.14 (mermaid asks `^3.4.12`).
- `chevrotain`, `@chevrotain/gast`, `@chevrotain/cst-dts-gen` each declare `lodash-es: 4.17.23` (exact) → three nested copies; the hoisted copy is already 4.18.1.

**CodeQL:** `writeFleetRootFixture` builds module *source* from `JSON.stringify(path.basename(root))` and writes it to disk. `JSON.stringify` is a data encoder, not a code sanitizer — that is the rule. The value is a `mkdtemp` basename inside a spec (alerts are classified `test`), so nothing is exploitable; the construction is still the pattern the rule names, and a fixture is where the next author copies it from.

## The Architectural Reality

- **A changed nested override is ignored while its stale lock entry exists** — `npm install` answers "up to date" and keeps 3.4.8 (measured here in a scratch copy; same finding in neo#18354). Remove the nested entry, then install: one `dompurify@3.4.14`, one `lodash-es@4.18.1`, and with #124's sharp entry `npm audit` reports 0 vulnerabilities.
- **Monaco vendors DOMPurify inside its tarball** (neo#18354). An override changes executed bytes only where Monaco is *built* from its ESM sources — the engine's `buildScripts/build/monaco.mjs` (neo#18717) resolves `dompurify` the way npm does. No Institution surface loads Monaco (`grep -ri monaco` outside `node_modules`: `package.json`, `.github/dependabot.yml`). The current engine pin predates that build step; the next pin carries it, so **this override has to land before the pin moves** — otherwise `build-all` bundles 3.4.8.
- **`lodash-es` 4.17.23 → 4.18.1 under chevrotain** was qualified on neo#18813: the import closure of chevrotain's lodash names meets the version diff in two comment-only files.

## The Fix

1. `package.json` `overrides`: `"lodash-es": "^4.18.0"` and `"monaco-editor": {"dompurify": "^3.4.13"}` — the engine's two entries, same ranges. #124's `sharp` entry rides the same PR (same file pair, same mechanism) — and with it the pack stage, which installs without a lock and has to repeat the overrides (finding recorded on #124).
2. `package-lock.json`: drop the stale nested entries, `npm install`, verify with a clean `npm ci`.
3. `brain.spec.mjs`: the fixture modules become constants that read the root name off their own location (`import.meta.url`). No value reaches source text, and an answer names the root the module was loaded from.

## Acceptance Criteria

- [ ] Clean `npm ci` from the PR lock resolves exactly one `dompurify` (≥ 3.4.13) and one `lodash-es` (≥ 4.18.0); `npm audit` lists neither package.
- [ ] The engine's mermaid bundle step builds against the deduplicated `lodash-es`.
- [ ] `brain.spec.mjs` interpolates no runtime value into module source; the Fleet-contract arm still proves per-root independence (each root answers its own name); mutation control: a wrong climb depth turns the arm red.
- [ ] Unit tier green locally and in both CI jobs.
- [ ] Post-merge: Dependabot 26–31 read `fixed` and CodeQL 1–3 read `fixed` on `dev`; none dismissed. Scanner delay is disclosed, not papered over.

## Out of Scope

The engine pin bump (next ticket, depends on this one). The Brain pin and the Brain's own lock (neomjs/neo-agent-brain#300). Adopting the `npm overrides` baseline check (neomjs/neo-agent-skills#88, unreleased).

## Avoided Traps

- **`npm audit fix --force`** proposes `monaco-editor@0.53.0` — a downgrade across the addon contract.
- **`npm update dompurify`** moves the hoisted copy and leaves the nested one (neo#18354).
- **Dismissing as "dev-scope" / "test code".** The mermaid bundle is a build output; the fixture is a template. Fix the cause.
- **Claiming the override patches Monaco's vendored bytes.** It does not; see Architectural Reality.

## Related

#124 · neo#18354 · neo#18717 · neo#18812 · neo#18813 · neomjs/neo-agent-brain#300 · neomjs/neo-agent-skills#88

Decision Record impact: none.

Live latest-open sweep: 19 open issues read at 2026-09-18T10:24Z plus an all-state search (`dompurify OR lodash OR CodeQL OR code-scanning OR dependabot`) — only #124 (sharp) and the closed #142 (version updates) surfaced. A2A sweep: latest 50 messages, no claim on this surface. Memory Core: problem-keyed query surfaced neo#18354 / neo#18813 / #124's filing turn — prior art folded in above. Own open assignments (#10, #127, #128, #129): no overlap.

Origin Session ID: 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59
Retrieval Hint: "Institution root overrides lodash-es monaco dompurify stale nested lock entry CodeQL bad-code-sanitization fixture tag as data"


## Timeline

- 2026-09-18T10:25:55Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T10:25:56Z @neo-fable-clio added the `bug` label
- 2026-09-18T10:25:56Z @neo-fable-clio added the `ai` label
- 2026-09-18T10:25:57Z @neo-fable-clio added the `dependencies` label
- 2026-09-18T10:25:57Z @neo-fable-clio added the `security` label
- 2026-09-18T10:25:57Z @neo-fable-clio added the `testing` label
- 2026-09-18T10:38:17Z @neo-fable-clio cross-referenced by #124
- 2026-09-18T10:39:43Z @neo-fable-clio cross-referenced by PR #150
- 2026-09-18T10:40:56Z @neo-fable-clio cross-referenced by #151
- 2026-09-18T11:22:46Z @tobiu referenced in commit `5404ca3` - "Merge pull request #150 from neomjs/agent/149-security-tab-zero

fix(deps): the Security tab's alerts close at the root lock and in the packaged stage (#149)"
- 2026-09-18T11:22:46Z @tobiu closed this issue
### @neo-fable-clio - 2026-09-18T11:25:48Z

Post-merge validation (AC-5, and #124's AC-4), read 2026-09-18T11:24Z after the merge at 11:22:44Z (`5404ca33c7`): Dependabot 23, 25–31 all `fixed` (11:22:49–50Z), none dismissed, 0 open · CodeQL 1–3 all `fixed` at 11:24:10Z — the analysis of the merge commit reports `results_count: 0`, the one before it (`4c874ea626`) reported 3. The code-scanning page lagged the merge by 86 s; nothing was dismissed to get there. Still open from the PR's checklist: the next `npm run dist` stage (Residual-Owner #7).

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59


