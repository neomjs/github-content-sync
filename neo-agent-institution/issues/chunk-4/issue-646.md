---
id: 646
title: 'The Institution tracks Brain and engine dev, not hand-bumped SHA pins'
state: OPEN
labels:
  - enhancement
  - ai
  - build
  - dependencies
assignees:
  - neo-opus-vega
createdAt: '2026-10-09T15:41:13Z'
updatedAt: '2026-10-09T15:41:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/646'
author: neo-opus-vega
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
# The Institution tracks Brain and engine dev, not hand-bumped SHA pins

## Context

The operator's standing order of 2026-09-21: an org repository that consumes another one tracks that repository's latest `dev`, resolved at run time. Provenance is recorded on the artifact; it is never a SHA a human must bump.

The Institution still pins both org dependencies, in three places:
- `package.json`: `neo-agent-brain` at `github:neomjs/neo-agent-brain#fb8c11ee…` and `neo.mjs` at `github:neomjs/neo#d75cc685…`;
- `package-lock.json`;
- `.github/workflows/ci.yml:77`, the Brain checkout `ref`.

On 2026-10-09 the operator chose to bump now and de-pin next. The bump carries Brain #952 in the usual way, as its own ticket; this ticket ends the pattern.

What the pattern costs:
- At least fifteen pin tickets since August: `#163`, `#195`, `#228`, `#269`, `#277`, `#300`, `#302`, `#380`, `#409`, `#442`, `#451`, `#460`, `#469`, `#484` and `#538`. Each is a ticket, a PR and a review that move one SHA.
- The Brain SHA lives in two hand-maintained places with no drift guard. This was raised in the review of `#307` (2026-09-28).

## The Problem

A merged Brain or engine fix reaches no installed walk until someone files a pin ticket. The lag is human attention, not test time.

The pin claims to protect against running an untested upstream. CI already gives that protection: a `dev` that breaks the Institution turns its run red.

## The Architectural Reality

- **The lock decides what installs.** `package.json` names both dependencies as `github:` refs with a SHA, and `package-lock.json` records each resolved SHA with its integrity. `npm ci` installs exactly the lock. Writing `#dev` in `package.json` alone still installs whatever SHA the lock last resolved, so tracking `dev` needs a resolve step at run time, not a string change.
- **CI is pinned too.** The `cross-repository` job checks the Brain out at the pinned `ref` and links the Institution's engine into it (`ci.yml:62–109`).
- **The visual freshness gate hashes the engine's lock entry** (`buildScripts/checkVisualBaselines.mjs:166–184`). An engine that moves at run time moves that digest. Unless the stamp model changes too, every engine merge turns `Check visual baseline freshness` red.
- **Builds already record their owners.** A packaged build writes them into `organism-build-info.json` (`harness/README.md:106`, `:307`), which is the natural home for the resolved SHAs.

## The Fix

The shape is settled in the PR.

- **Dependencies:** both name `dev`. Each CI job resolves them before its tests and prints the resolved SHAs in the job summary. The Brain checkout uses `ref: dev`.
- **Packaging:** the build resolves at cut time and writes both resolved SHAs into `organism-build-info.json`. The ROADMAP's `Row state:` candidate line reads them from there.
- **Visual stamp:** it keys on something a moving engine does not break. Either it records the engine SHA it was captured against and reports drift without failing, or the engine leaves the stamp's identity. The goldens' Darwin-local authority is unchanged either way.

## Acceptance Criteria

- [ ] AC-1: `package.json` and `ci.yml` contain no Brain or engine SHA that a human must bump.
- [ ] AC-2: every CI job resolves both dependencies at run time and names the resolved SHAs in its summary.
- [ ] AC-3: a packaged candidate's `organism-build-info.json` records both resolved SHAs.
- [ ] AC-4: fail-closed. A Brain or engine revision that breaks the Institution turns its CI red and names the resolved SHA. This is witnessed once against a deliberately broken ref.
- [ ] AC-5: an engine merge that changes no Institution input does not turn the visual freshness gate red.

## Out of Scope

- External and vendor dependencies; Dependabot keeps those.
- How other repositories consume each other, such as the Brain's own engine and skills ranges.

## Avoided Traps

- **`#dev` in `package.json` with an unchanged `npm ci`.** It reads as tracking but installs the old lock.
- **A scheduled bump PR.** It automates the hand step instead of removing it, and keeps both the lag and a review per bump.

## Related

#307 · #12 · #7

Live latest-open sweep: checked the latest 20 open issues at 15:39Z; no equivalent. A2A claim sweep (last 30, all states): no competing claim. Own-assignment sweep: #485 only, unrelated. MC sweep: the 2026-09-21 order; no recorded Institution exception. Structure map: N/A (dependency resolution, CI and build only; no `ai/` placement). Decision Record impact: none.

Origin Session ID: 9a84c569-02eb-4f7c-b87d-43ebcb24593d
Retrieval Hint: "Institution track Brain engine dev de-pin package.json ci.yml resolve at run time visual stamp engine digest"

## Timeline

- 2026-10-09T15:41:14Z @neo-opus-vega added the `enhancement` label
- 2026-10-09T15:41:15Z @neo-opus-vega added the `ai` label
- 2026-10-09T15:41:15Z @neo-opus-vega added the `build` label
- 2026-10-09T15:41:15Z @neo-opus-vega added the `dependencies` label
- 2026-10-09T15:41:28Z @neo-opus-vega cross-referenced by #647
- 2026-10-09T15:41:29Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-09T15:47:44Z @neo-opus-vega cross-referenced by PR #648

