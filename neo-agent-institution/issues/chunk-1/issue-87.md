---
id: 87
title: 'js-yaml is a major behind the engine and the Brain, and npm outdated cannot see it'
state: CLOSED
labels:
  - bug
assignees:
  - neo-fable-clio
createdAt: '2026-09-02T19:13:32Z'
updatedAt: '2026-09-04T17:44:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/87'
author: neo-opus-grace
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
closedAt: '2026-09-04T17:44:31Z'
---
# js-yaml is a major behind the engine and the Brain, and npm outdated cannot see it

## Context

Dependency audit across the three org repos, 2026-09-02 (operator-requested: *"npm outdated, also checking for new major versions. i recommend engine, brain and agent-institution"*).

This repo's own advisory surface is the cleanest of the three — **one moderate (`qs`), fixable in range**. The finding worth a ticket is one no single-repo check could produce, and it only appeared because all three were read together.

## The Problem

**`js-yaml` is a full major behind its siblings, on a parser all three repos use for the same substrate files.**

| repo | installed | declared | latest |
|---|---|---|---|
| `neo` | 5.2.3 | — | 5.4.1 |
| `neo-agent-brain` | 5.4.0 | — | 5.4.1 |
| **`neo-agent-institution`** | **4.3.2** | `^4.1.1` | 5.4.1 |

Engine and Brain are on 5.x and drift only by patch. Institution is pinned to `^4`, so `npm outdated` here reports it as "wanted: 4.3.2" — current *for its range*, and invisible as a problem until the three are compared.

Why it matters more than a version number: these repos parse the same kinds of substrate (front-matter, workflow and config YAML) and increasingly move files between each other. Two majors of one parser across an org is a class of bug that shows up as "it worked in the other repo" — the hardest kind to attribute, because nothing is broken locally on either side.

`js-yaml` 5 is a real major with parsing and API changes, not a version-number bump, which is presumably why the pin exists. That makes this a small migration rather than a lockfile refresh, and it is why it gets its own ticket rather than riding along with the advisory fix.

## The Fix

1. Close the `qs` moderate with an in-range lockfile refresh. Independent of everything below; should not wait for it.
2. Move `js-yaml` to `^5`, aligning with `neo` and `neo-agent-brain`, and adjust for the 4 → 5 API and parsing changes at each call site.
3. Record the alignment so the next audit reads it as intentional rather than as drift that happened to resolve.

## Acceptance Criteria

- [ ] **AC-1** `npm audit` reports 0 vulnerabilities.
- [ ] **AC-2** `js-yaml` declared `^5` and installed at 5.x; every call site exercised by the suites, with the ones that changed behaviour named in the PR body rather than assumed equivalent.
- [ ] **AC-3** The full local suites are green — the parser reads substrate this repo depends on, so a green diff is not evidence.
- [ ] **AC-4** Any YAML this repo parses that another repo also parses is checked against the same input under both majors, or explicitly noted as not shared.

## Out of Scope — the majors, listed so they are not lost

All `devDependencies`, none security-driven, each wanting its own evidence:

- `inquirer` 13.4.3 → 14.2.0
- `monaco-editor` 0.50.0 → 0.56.0 — six 0.x minors, breaking by convention. **`neo` is on the same 0.50.0**, so these two are in sync and should probably move together rather than independently.
- `webpack` / `webpack-cli` — in range, covered by an ordinary update.

## Avoided Traps

- **Reading `npm outdated`'s "wanted" column as health.** It answers "am I current *for my declared range*", and a deliberately old range reports clean forever. The skew here is invisible to that column by construction.
- **Bundling the major into the advisory fix.** The `qs` refresh is lockfile-only and reviewable at a glance; a parser major is neither.
- **Aligning for tidiness.** The argument is shared substrate and file movement between repos, not symmetry. If a call site genuinely wants 4.x semantics, that is a finding to record, not a rule to satisfy.

## Related

- neomjs/neo#18135 — the engine's five advisories, all in range.
- neomjs/neo-agent-brain#300 — the `sharp`/libvips chain with no fix available.

Origin Session ID: fb58e855-74b2-4367-9f83-5877f151f024

Retrieval Hint: "institution js-yaml 4 while engine and brain run 5, cross-repo parser major skew invisible to npm outdated wanted column"


## Timeline

- 2026-09-02T19:13:33Z @neo-opus-grace added the `bug` label
- 2026-09-04T09:27:32Z @neo-fable-clio cross-referenced by #90
- 2026-09-04T17:02:51Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T17:06:42Z @neo-fable-clio cross-referenced by PR #106
- 2026-09-04T17:33:03Z @neo-fable-clio cross-referenced by #107
- 2026-09-04T17:34:37Z @neo-fable-clio referenced in commit `bf3ad1f` - "chore(institution): js-yaml ^5.2.3 aligned with the engine and the Brain; the qs moderate closed in range (#87)

One call site (test/playwright/unit/harness/pack.spec.mjs: namespace import, load() and dump() without options) — the migration guide's base case; the real input parses byte-identically under 4.3.2 and 5.4.1. The declared range matches the siblings' ^5.2.3 so the next cross-repo audit reads the alignment as intentional. npm audit fix closed qs in range (6.16.0): 0 vulnerabilities. check-visual-baselines passed unchanged: its engine digest hashes only the neo.mjs lock entry, and js-yaml and qs sit outside that axis."
- 2026-09-04T17:44:31Z @tobiu referenced in commit `0b0b33a` - "Merge pull request #106 from neomjs/agent/87-js-yaml-5

chore(institution): js-yaml ^5.2.3 aligned with the engine and the Brain; the qs moderate closed in range (#87)"
- 2026-09-04T17:44:31Z @tobiu closed this issue

