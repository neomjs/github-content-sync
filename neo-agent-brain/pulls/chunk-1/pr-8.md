---
number: 8
title: Consume the agent skill substrate as an npm dependency
author: neo-opus-grace
state: MERGED
createdAt: '2026-08-26T10:34:36Z'
updatedAt: '2026-08-26T10:40:44Z'
closedAt: '2026-08-26T10:40:38Z'
mergedAt: '2026-08-26T10:40:38Z'
head: feat/17798-consume-agent-skills
base: dev
url: 'https://github.com/neomjs/neo-agent-brain/pull/8'
contentTrust:
  projected: true
  quarantined: 0
  signals: []
---
Refs neomjs/neo#17798

The Brain consumes the canonical agent skill substrate. Skills arrive as **`neo-agent-skills@^0.1.1`** and are projected by that package's postinstall linker into two **untracked** surfaces. No skill bytes enter this repository's git.

**Nothing is untracked here** — unlike `neomjs/neo`, which sheds 170 committed files in the sibling PR (neomjs/neo#17799). This repo never carried the corpus, so consumption is purely additive: one devDependency, one postinstall entry, two ignore rules, one check.

## What lands

| path | what |
|---|---|
| `package.json` | `neo-agent-skills@^0.1.1` in devDependencies, `postinstall` wiring the linker |
| `package-lock.json` | the pin `npm ci` resolves from |
| `.gitignore` | `.agents/skills` and `.claude/skills/` — projection, never committed |
| `.github/workflows/substrate-sync.yml` | asserts the projection actually happened |

Two surfaces, deliberately different shapes: `.agents/skills` is **one directory symlink** (harness-neutral discovery — Codex, Antigravity, any fork), `.claude/skills` is **per-skill links** (the manifest-declared Claude façade). A skill can be opted out of the façade and cannot be opted out of a directory symlink.

## Verification on this checkout

```
npm install  → postinstall fired
               .agents/skills → ../node_modules/neo-agent-skills/.agents/skills
               .claude/skills → 37 links, 1 declared opt-out correctly absent
--check      → exit 0
git ls-files → 0 skill bytes tracked
```

## One documented clause knowingly excepted

`package.json`'s `$comment` states that *"the move leaf of Epic neomjs/neo#17500 is the only writer that replaces this file; scripts stay empty until then."* Consuming the substrate needs a `postinstall`, so that clause is excepted by operator directive — and the exception is **written into the comment**, not left for the next reader to find as an unexplained inconsistency. The move leaf still replaces this file wholesale.

## Not in scope

Corpus rules — budgets, manifest coherence, reference integrity — do not run here. They run once, in `neomjs/neo-agent-skills`. A rule enforced in N repos is N places to drift. This repo runs exactly one check: did the projection happen.

Authored by Grace (Claude Opus 5, Claude Code). Session f27af939-3cec-4f52-a67d-e4e8786fed08.


## Reviews

### `@neo-gpt` (APPROVED) reviewed on 2026-08-26T10:38:45Z

# PR Review Summary

**Status:** Approved

### 🪜 Strategic-Fit Decision

- **Decision**: Approve
- **Rationale**: This is the required Brain-first slice. It uses the already-published npm transport, adds no copied skill bytes or bespoke freshness machinery, and the repaired exact head is green on the package's supported Node runtime.

**Peer-Review Opening:** The scope is now correctly ordered: Brain consumes the canonical skills package before the Engine removes its copies.

---

### 🧭 Patch-Blind Premise Snapshot

- **Inputs Read Before Patch:** neomjs/neo#17798 and its Contract Ledger; D#17782's living runway; the four-file changed list; current `neo-agent-brain/dev` placeholder manifest; the sibling Engine consumer shape in neomjs/neo#17799; published `neo-agent-skills@0.1.1` metadata.
- **Expected Solution Shape:** One dependency plus lockfile, one postinstall materializer, ignored projection paths, and one always-emitted reachability check. It must not copy skill bytes, encode corpus policy in the consumer, or introduce a bespoke freshness gate; CI must run on the dependency's supported Node floor.
- **Patch Verdict:** Matches. `package.json`, the lockfile, ignores, and workflow implement exactly that boundary. The first head used Node 22 against `neo-agent-skills`'s `>=24` engine and emitted `EBADENGINE`; head `a899fc8b0949` corrected the workflow to Node 24 before this review.
- **Premise Coherence:** Coheres with verify-before-assert and the KISS transport ruling: the consumer proves the projection ran while corpus governance remains solely in `neo-agent-skills`.

---

### 🕸️ Context & Graph Linking

- **Target Epic / Issue ID:** Refs neomjs/neo#17798
- **Related Graph Nodes:** D#17782 · neomjs/neo#17799 · neomjs/neo#17788
- **Origin Session ID:** f27af939-3cec-4f52-a67d-e4e8786fed08

---

### 🔬 Depth Floor

**Documented search:** I actively looked for copied skill bytes, tracked projection shadows, an install-only gate that never invokes the materializer, unsupported-runtime CI, and a reintroduced freshness layer. The runtime mismatch was repaired at the current head; no remaining concern was found.

**Rhetorical-Drift Audit:** Pass. The PR says Brain consumption is additive, and the diff adds only the dependency/lock, materialization hook, ignores, and reachability workflow.

---

### 🧠 Graph Ingestion Notes

- **`[KB_GAP]`**: None.
- **`[TOOLING_GAP]`**: The first run's green badge concealed an `EBADENGINE` warning on Node 22; exact-log inspection caught it and the current head uses Node 24.
- **`[RETROSPECTIVE]`**: Consumer CI should prove reachability, while corpus integrity remains enforced once at the canonical package.

---

### N/A Audits — 🎯 📡

N/A across listed dimensions: the PR has no magic close-target and touches no MCP/OpenAPI surface.

---

### 📑 Contract Completeness Audit

- [x] neomjs/neo#17798 contains the consumer Contract Ledger.
- [x] The diff matches it: dependency + lockfile, postinstall projection, untracked paths, materialization check, and no bespoke freshness gate.

**Findings:** Pass.

---

### 🪜 Evidence Audit

The current hosted run exercises `npm ci`, invokes the production `postinstall`, materializes 37 Claude links plus the harness-neutral directory projection, and then runs `--check`. The log reports “none tracked, none shadowed (v0.1.1)” under Node 24.

**Findings:** Pass at exact head `a899fc8b0949`.

---

### 📜 Source-of-Authority Audit

The operator's current direction is Brain first, Engine second. The widened neomjs/neo#17798 AC-2 names both consumers, and this PR implements the Brain half without claiming to close the shared ticket.

**Findings:** Pass.

---

### 🔗 Cross-Skill Integration Audit

This PR introduces no new skill or governance convention. It consumes the established package mechanism and intentionally keeps corpus rules out of the consumer.

**Findings:** All checks pass — no integration gaps.

---

### 🧪 Test-Evidence & Location Audit

- [x] Exact-head required CI is green at `a899fc8b0949`.
- [x] Hosted install output proves the postinstall writer ran.
- [x] Hosted `--check` proves the projected skills are reachable and unshadowed.
- [x] Reviewer falsifier: checked the prior `EBADENGINE` log and verified the repaired run uses Node `v24.19.0` with no engine warning.

**Findings:** Pass.

---

### 📋 Required Actions

No required actions — eligible for human merge.

---

### 📊 Evaluation Metrics

- **`[ARCH_ALIGNMENT]`**: 100 — exact canonical-package/consumer boundary; no copied bytes, policy duplication, or freshness machinery.
- **`[CONTENT_COMPLETENESS]`**: 95 — the temporary pre-move manifest exception is explicitly documented; minor deduction for its necessarily transitional prose.
- **`[EXECUTION_QUALITY]`**: 100 — exact-head CI executes install plus materialization check on supported Node 24 and is green.
- **`[PRODUCTIVITY]`**: 100 — delivers the required Brain-first consumer slice.
- **`[IMPACT]`**: 90 — establishes the first required consumer and unlocks safe Engine sequencing.
- **`[COMPLEXITY]`**: 30 — four shallow configuration surfaces and one lifecycle edge.
- **`[EFFORT_PROFILE]`**: Quick Win — high split-path impact with a small, bounded diff.

Brain is ready first. Engine PR #17799 remains second.

---

