---
id: 432
title: Re-read PR diffs after compaction and add class-idiom review checks
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
assignees: []
createdAt: '2026-11-21T04:39:11Z'
updatedAt: '2026-11-21T04:42:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/432'
author: neo-gemini-julia
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
# Re-read PR diffs after compaction and add class-idiom review checks

## Context

During `neomjs/neo#21716` (`#21705`, `2026-11-21`), context compaction occurred between reading the PR diff and submitting the formal review (`#pullrequestreview-6002722463`). On resume, the reviewer ran `context-recovery` (`list_pull_requests({believedOpen})` + `get_conversation`), saw `live-state` green, and drafted `APPROVED` from the pre-compaction summary's compressed conclusions rather than re-reading `get_pull_request_diff`, missing three `.github/CODING_GUIDELINES.md` violations (`2` classes in `1` file, module-scope `const themeLabels` instead of `themeLabels_` with `merge: 'shallow'`, and repeated `this.` without `me = this`, plus eager static `import` of a lazy `menu/List` dependency).

## The Problem

1. **Compaction invalidates unsubmitted PR diff reads, but `context-recovery` only checks PR liveness:**
   - `.agents/skills/context-recovery/references/context-recovery-workflow.md` §3 step 5 (`live-state`, verified at `neo-agent-skills` `origin/dev@a73c2690725cd147d36bd15cedc4a121d39e0c0f`) checks whether a PR is still open/unmerged, and `.agents/skills/pr-review/references/pr-review-guide.md` §2.2 requires exact-head source inspection, **without** stating that a compaction boundary erases raw diff lines into lossy prose conclusions. When compaction bisects a review before `manage_pr_review`, scoring from the compaction summary manufactures a rubber-stamp approval.
2. **`pr-review/audits/core-idiom-audit.md` (`2,560 B` at `a73c269072`) omits four high-frequency Neo class idioms:**
   - Reviewers loading `core-idiom-audit.md` check batched `set()`, `Neo.getComponent`, reactive state, service lifecycle, and stateful owner shape, while walking past:
     - **One class per `.mjs` file** (`CODING_GUIDELINES.md` §3 `(11)` / §4 `(20)` — until every repo runs the AST guard),
     - **Consumer-extensible maps on `static config` (`merge: 'shallow'`)** instead of module-scope `const` lookup tables,
     - **`me = this` caching** when `this` is referenced `> 2` times in bundled runtime code (`CODING_GUIDELINES.md` §8 `(34)`: `src/`, `apps/`, `examples/`, `docs/app/`), and
     - **Preserving base-class lazy `import()` boundaries** (for example `Button#afterSetMenu`'s dynamic `import('../menu/List.mjs')` defeated by a subclass's static top-level menu import).

## The Architectural Reality

- Progressive Disclosure & `/turn-memory-pre-flight` placement evaluation (`turn-memory-pre-flight-workflow.md` Step 2 — lifecycle-scoped skill payloads, **zero** `AGENTS.md` or `SKILL.md` router byte growth):
  - `context-recovery-workflow.md` (`8,840 B` at `a73c269072`, well under its `25,000 B` cap): owns post-compaction recovery before resuming an in-flight review.
  - `core-idiom-audit.md` (`2,560 B` at `a73c269072`, conditional load via `pr-review-guide.md` §7.5.1): owns Neo class/instance idiom checks during `/pr-review` without growing `pr-review-guide.md` (`32,503 B`, combined-budget gated with `review-cost-circuit-breaker.md`).
- Accretion defense (`AGENTS.md §self_evolving_systems` & `scripts/lint-skill-corpus.mjs` `maxPositiveDeltaBytes`): keep `SKILL.md` routers at `0 B` delta and `pr-review-guide.md` `<= 0 B` net delta by tightening adjacent prose in `context-recovery-workflow.md` / `core-idiom-audit.md` (or declaring sunset triggers where mechanical lints retire individual bullets).
- Design authority: `"AGENTS.md §verify_before_assert: (1) Code, runtime, architecture — re-run every time; a check is spent the turn it ran (neo #15780)" and "neomjs/neo .github/CODING_GUIDELINES.md §3 (11), §4 (20), and §8 (34) (me = this when this is used > 2 times in bundled runtime code)"`.

## The Fix

1. **`context-recovery-workflow.md` §3 step 5 (`Live substrate check`):**
   - Add one compact clause: if compaction occurred while a PR review or code-audit verdict was still unsubmitted, the pre-compaction diff read is spent — re-fetch `get_pull_request_diff` at `headRefOid` in the active window before `manage_pr_review`; never score or approve from a compaction summary (`DISCIPLINE-ONLY` until review tooling binds diff-read turn provenance).
2. **`pr-review/audits/core-idiom-audit.md` (`## The checks`):**
   - Add concise, high-signal sub-checks covering: (a) one class per `.mjs` file matching `className` ↔ path, (b) consumer-extensible label/option maps declared as `static config` descriptors (`merge: 'shallow'`) rather than module-scope `const` objects, (c) `me = this` when `this` is used `> 2` times in bundled runtime code (`src/`, `apps/`, `examples/`, `docs/app/`), and (d) preserving base-class lazy `import()` boundaries (for example no top-level static import of an optional heavy dependency the base class lazy-loads).
   - Include explicit retirement trigger: retire check (a) once `check-class-module-scope` enforces single-class files across repos (`neomjs/neo#21737`).
3. **Budget & corpus verification:**
   - Keep all `SKILL.md` routers untouched (`0 B` delta), keep `pr-review-guide.md` untouched or net `<= 0 B`, and pass `npm test` + `npm run lint` (`scripts/lint-skill-corpus.mjs`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/context-recovery/references/context-recovery-workflow.md` §3.5 (`8,840 B` at `a73c269072`) | `AGENTS.md §verify_before_assert` (`neo #15780`), `#203` | Invalidates unsubmitted PR diff reads across compaction; requires fresh `get_pull_request_diff` in the resumed window before `manage_pr_review` | Completed/submitted reviews need only `list_pull_requests({believedOpen})` liveness check | Inline payload (`SKILL.md` `0 B` delta) | `npm run lint` (`lint-skill-corpus.mjs`) + `npm test` |
| `.agents/skills/pr-review/audits/core-idiom-audit.md` (`2,560 B` at `a73c269072`) | `neomjs/neo` `.github/CODING_GUIDELINES.md` §3 `(11)`, §4 `(20)`, §8 `(34)` | Checks 1-class-per-file, `static config` (`merge: 'shallow'`) over module-scope consumer maps, `me = this` (`> 2` uses in bundled trees), and lazy `import()` preservation | Unbundled Node/Agent OS trees (`ai/`, `buildScripts/`) remain exempt from `me = this` per `CODING_GUIDELINES.md:469` | Inline audit payload (`pr-review-guide.md` `<= 0 B` delta) | `npm run lint` + `npm test` |

Decision Record impact: aligned-with `neomjs/neo` ADR 0007 (`0007-agents-md-compaction-taxonomy.md`) and ADR 0008 (`0008-skill-anatomy-and-authoring-contract.md`).

## Acceptance Criteria

- [ ] `context-recovery-workflow.md` §3.5 explicitly invalidates unsubmitted PR diff reads across compaction and requires re-reading `get_pull_request_diff` in the active window before submitting a review.
- [ ] `pr-review/audits/core-idiom-audit.md` adds checks for one-class-per-file, `static config` (`merge: 'shallow'`) vs module-scope consumer maps, bundled-tree `me = this` (`> 2` uses, exempting `ai/` and `buildScripts/`), and preserving base-class lazy `import()` boundaries, with a retirement trigger pointing to `neomjs/neo#21737`.
- [ ] Zero `SKILL.md` router growth, zero `COMBINED_BUDGETS` breach on `pr-review-guide.md`, and `npm run lint` + `npm test` pass in `neomjs/neo-agent-skills`.

## Out of Scope

- Mechanical AST enforcement in `neomjs/neo` (`buildScripts/util/check-class-module-scope.mjs`), tracked in sibling Engine ticket `neomjs/neo#21737`.
- Modifying `AGENTS.md` or any `SKILL.md` router (`turn-memory-pre-flight` Step 2 keeps this strictly inside conditional skill payloads).

## Avoided Traps

- **Adding rules to `AGENTS.md` or `pr-review-guide.md` (`32,503 B`):** `context-recovery` already fires on every post-compaction resume and `core-idiom-audit.md` already fires on every Neo class/instance diff (`§7.5.1`), so placing both checks in those conditional payloads pays zero always-loaded token tax and respects `COMBINED_BUDGETS`.
- **Requiring `me = this` in `ai/` or `buildScripts/`:** `CODING_GUIDELINES.md:469-470` explicitly scopes `me = this` to bundled runtime-engine code (`src/`, `apps/`, `examples/`, `docs/app/`) and prefers direct `this` in unbundled Node/Agent OS code.

## Related

- Originated from `Friction -> Gold` Typed Calibration Loop overturn on `neomjs/neo#21716` (`#21705`, `MESSAGE:8bb7aeef` & `defect-note` `MESSAGE:d177aba1`).
- Sibling mechanical AST guard: `neomjs/neo#21737` (*Enforce one-class-per-file in check-class-module-scope and docs*).
- Adjacent prior art: `#166` (core-idiom fixture scope, parked) and `#203` (`context-recovery` `VISION.md` step 5a).

unowned-rationale: Queued for post-Engine-14-cut skills batch so tonight's 20-runner org pool stays dedicated to Engine 14 PRs (`neomjs/neo` D#21725).

Meta-skill pre-flight (§1b): verified against `create-skill/SKILL.md` + `skill-authoring-guide.md` and `turn-memory-pre-flight-workflow.md` (`SKILL.md` routers untouched, conditional payload placement, `.claude/CLAUDE.md -> ../AGENTS.md` and `.codex/hooks.json` verified).
Structure-map gate (§1c): `npm run ai:structure-map -- --files --loc` executed cleanly in `neo-agent-brain`; target files live in `neomjs/neo-agent-skills` `.agents/skills/{context-recovery,pr-review}/`.
Live latest-open + closed sweep: checked latest 20 open (`39` open total, orange band — no existing open `context-recovery` or `core-idiom-audit` owner) + open and closed search (`compaction`, `context-recovery`, `core-idiom-audit`; read `#166` and `#203` bodies and `Out of Scope` in full) at `2026-11-21T04:37Z`; no equivalent or prior exclusion found.
A2A in-flight sweep: scanned latest 30 messages across all read states (`04:11Z`–`04:37Z`); zero competing claim (promotes own `defect-note` `MESSAGE:d177aba1`).
MC sweep: `query_raw_memories` (`"two classes in one file lookup table module scope rubber stamp review after compaction"`, 6 results) + `ask_knowledge_base` (`type: 'ticket'`, 5 refs); no prior decision found.
Own-assignment sweep: `0` open assigned to `@neo-gemini-julia` in `neo-agent-skills` (`#423` & `#387` authored, disjoint).

Origin Session ID: 664ca229-4bbd-4e00-ae41-008707688181

Retrieval Hint: "context-recovery compaction unsubmitted PR diff re-read core-idiom-audit one class per file merge shallow me = this lazy import"

## Timeline

- 2026-11-21T04:39:13Z @neo-gemini-julia added the `enhancement` label
- 2026-11-21T04:39:13Z @neo-gemini-julia added the `ai` label
- 2026-11-21T04:39:13Z @neo-gemini-julia added the `architecture` label
- 2026-11-21T04:39:14Z @neo-gemini-julia added the `model-experience` label

