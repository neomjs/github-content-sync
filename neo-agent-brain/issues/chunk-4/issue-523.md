---
id: 523
title: 'neo-agent-brain calls no PR baseline, so five shipped guards never run'
state: OPEN
labels:
  - enhancement
  - ai
  - build
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-03T16:31:15Z'
updatedAt: '2026-09-25T22:05:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/523'
author: neo-opus-grace
commentsCount: 0
parentIssue: 14
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
# neo-agent-brain calls no PR baseline, so five shipped guards never run

> **Transferred 2026-09-25 from `neomjs/neo-agent-skills#40`.** Consumer leaves live in their consumer repository (`devindex#11`, Institution's `#189`), and the PR-body lint needs a same-repo close target. Bare `#N` refs below are neo-agent-skills numbers, and "this repository" means neo-agent-skills. Since filing, the baseline has grown from five jobs to nine, and AC-4 covers all nine. Delivered by PR #524.

## Context

`#18` shipped source-comment archaeology as a portable, reusable guard and closed COMPLETED on 2026-08-30. Its body states the consumer-side contract explicitly:

> Consumer caller files remain separate leaves under neomjs/neo-agent-skills#14.

No such leaf was ever filed for `neomjs/neo-agent-brain`. This ticket is that leaf.

It was found sideways, which is worth recording: reviewing [neo-agent-brain#301](https://github.com/neomjs/neo-agent-brain/pull/301), @neo-opus-ada flagged a `[TOOLING_GAP]` — that `buildScripts/util/check-ticket-archaeology.mjs` "exists in this repo (13,403 bytes) and is wired to nothing." The gap is real. **The premise is not:** that file is not in the Brain. The 13,403-byte artifact is `node_modules/neo.mjs/buildScripts/util/check-ticket-archaeology.mjs` — the Engine dependency's vendored copy, sitting at the Engine's own path, which is why the path matched. `git ls-files | grep -i archaeolog` in the Brain returns empty.

The correction matters because it changes the remedy. Nothing here needs wiring up; a published guard needs *calling*.

**Live latest-open sweep:** checked the latest 20 open issues in `neomjs/neo-agent-skills` at 2026-09-03T16:29Z; no equivalent found. Nearest neighbours are `#24`, `#38` and `#23`, all of which concern the baseline's own internals or policy, not whether any repository invokes it.
**A2A in-flight claim sweep:** `list_messages` at 16:29Z — no `[lane-claim]`/`[lane-intent]` overlapping CI governance or baseline adoption.
**Memory Core rationale sweep:** `query_raw_memories` on baseline/caller/consumer-adoption nouns returned no prior decision.
**Own-assignment sweep:** no open assigned ticket in either repository covers this surface.

## The Problem

`reusable-pr-baseline.yml` in this repository is a `workflow_call` workflow that owns five guard jobs:

| job | shipped by |
|---|---|
| `pr-base` | `#15` |
| `skills-materialized` | `#15` |
| `source-comment-archaeology` | `#18` |
| `substrate-size` | `#25` |
| `pr-body` | `#28` / `#29` |

Every one of those tickets is CLOSED COMPLETED. `neo-agent-brain` calls none of them.

Concretely, on the Brain's default branch:

- `.github/workflows/` holds ten workflows (`brain-integration`, `brain-unit`, `check-retired-primitives`, `config-template-ssot-lint`, `detection-retention-sla`, `identity-vocabulary-lint`, `mcp-test-location-lint`, `openapi-service-parity-lint`, `script-plane-lint`, `substrate-sync`). None is a baseline caller; the only one referencing this repository at all is `substrate-sync.yml`.
- No archaeology implementation is tracked (`git ls-files` empty on that pattern).
- The published CLI **is already installed** — `neo-agent-skills@0.1.3` provides `node_modules/.bin/neo-agent-skills-ticket-archaeology`. The capability is present and unreachable from CI.

So a convention `AGENTS.md` treats as mechanical is discipline-only in the Brain, and PR #301 demonstrated the consequence rather than theorising it: it carries a `(#68)` tracking ref in a durable source comment, which `neomjs/neo` rejects and which passed Brain CI green.

**One claim I could not establish, deliberately left out of scope.** I attempted an org-wide census of baseline callers via GitHub code search and got `0`. That zero is masked, not a finding: a control query for `reusable-pr-baseline` scoped to this repository — where the file is literally named that — also returned `0`, as did a known-present string in the Brain. The search index is not answering. The Brain finding above rests on the contents API and `git ls-files`, which do answer. Whether `neo-agent-institution` and `devindex` also lack callers is unverified here and is not asserted.

## The Architectural Reality

`#18`'s design is deliberately two-sided: the Skills package owns the CLI and an immutable `workflow_call` coordinate, and each consumer owns only its event trigger and a `uses:` line. That split is what prevents the drift that copying creates — and it is exactly why publishing alone changes nothing. The reusable side landed; the caller side is a per-repo leaf that has to be filed and merged in the consumer.

`neomjs/neo` is the instructive contrast. It is covered for archaeology, but by its own native `.github/workflows/ticket-archaeology-lint.yml` plus its own `buildScripts/util/` implementation — i.e. by the duplicate that `#18` set out to eliminate. Its migration onto the shared coordinate is a separate leaf and is not proposed here.

`#38` is adjacent and should be read before implementing: it holds that the baseline installs guards from the workflow's own commit rather than an npm pin. Whatever `#38` settles about *how* the caller resolves its guards governs the `uses:` coordinate this ticket adds, so the two want sequencing, not merging.

## The Fix

Add one caller workflow to `neomjs/neo-agent-brain` under `.github/workflows/`, owning only its `pull_request` trigger and an immutable `uses:` coordinate onto this repository's `reusable-pr-baseline.yml`, per the caller contract in that file's header.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `neo-agent-brain/.github/workflows/<caller>.yml` (new) | `#18` body: "Consumer caller files remain separate leaves under neomjs/neo-agent-skills#14" | Invokes `reusable-pr-baseline.yml` on `pull_request` | none — absence is the current state | caller header in `reusable-pr-baseline.yml` | Brain default-branch workflow listing (10 files, no caller) |
| `source-comment-archaeology` job | `reusable-pr-baseline.yml:73` | Becomes enforcing in the Brain | discipline-only, as today | `#18` | `reusable-pr-baseline.yml:109` invokes `neo-agent-skills-ticket-archaeology` |
| `neo-agent-skills-ticket-archaeology` bin | `neo-agent-skills@0.1.3` | Already installed in the Brain | n/a | package bin | `node_modules/.bin/` entry present |

## Decision Record impact

`none`. No ADR governs consumer CI caller placement; the authority is `#18`'s shipped contract and the caller header in `reusable-pr-baseline.yml`.

**Structure Map Gate:** N/A — this introduces no `.mjs` file. The artifact is a `.github/workflows/*.yml` caller whose placement is prescribed by `#18`'s own contract, so `structural-pre-flight` does not fire.

## Acceptance Criteria

- [ ] A caller workflow exists on the Brain's default branch invoking `reusable-pr-baseline.yml` at an immutable coordinate, consistent with whatever `#38` settles about guard resolution.
- [ ] The `source-comment-archaeology` job runs on Brain pull requests and appears in `gh pr checks`.
- [ ] The guard is proven to discriminate, not merely to be green: a PR carrying a tracking ref in a durable source comment goes RED, and the same PR without it goes green. A green first run is not evidence — the red arm is.
- [ ] The remaining four baseline jobs are each explicitly dispositioned as adopted or deferred-with-reason, so a partial adoption is recorded rather than silently inherited.
- [ ] The existing `(#68)` ref in `ai/configBase.mjs` (merged via #301) is either removed or recorded as a known exception, so the guard's first run reflects a decision rather than an accident.

## Out of Scope

- Migrating `neomjs/neo` off its native archaeology workflow onto the shared coordinate.
- Filing caller leaves for `neo-agent-institution` or `devindex` — unverified here, per the masked-search note above.
- Any change to `reusable-pr-baseline.yml` itself, including the pinning question `#38` owns.

## Avoided Traps

- **Wiring a local guard.** The obvious reading of the original finding is "a guard sits here unwired, hook it up." Acting on it would have re-copied the Engine implementation into the Brain — precisely the drift `#18` was filed to prevent. The file that looked local was a dependency.
- **Trusting a zero from a search index.** The org-wide caller census returned `0` and would have supported a much larger claim. The control proved the index was returning `0` for strings known to exist, so the larger claim is absent from this ticket by choice.
- **Bundling `#38`.** Both touch the caller's `uses:` line, which makes merging them tempting; `#38` decides how guards resolve, this decides that the Brain resolves them at all, and folding them would let the harder question gate the trivial one.

## Related

- Parent: `#14` (Unify PR governance across Neo repositories)
- Origin of the contract: `#18`; sibling shipped guards `#15`, `#25`, `#28`, `#29`
- Sequencing dependency: `#38`
- Adjacent: `#24` (the mandated PR preflight is not runnable outside the Brain)
- Discovery surface: [neo-agent-brain#301](https://github.com/neomjs/neo-agent-brain/pull/301) review by @neo-opus-ada

`unowned-rationale:` filed from a heartbeat run that cannot finish it — the Brain checkout is parked on the open `#301` branch and a concurrent session holds the Engine tree. Parked deliberately, not dropped; free for any seat.

Origin Session ID: 85461d39-4567-48a6-a8c3-856b7bb9b3d8

Retrieval Hint: `query_raw_memories("reusable-pr-baseline consumer caller leaf neo-agent-brain archaeology")`



## Timeline

- 2026-09-03T16:31:17Z @neo-opus-grace added the `enhancement` label
- 2026-09-03T16:31:17Z @neo-opus-grace added the `ai` label
- 2026-09-03T16:31:18Z @neo-opus-grace added the `build` label
- 2026-09-03T16:31:18Z @neo-opus-grace added the `agent-os` label
- 2026-09-03T16:31:31Z @neo-opus-grace added parent issue #14
- 2026-09-03T16:31:58Z @neo-opus-grace cross-referenced by PR #301
- 2026-09-04T10:58:54Z @neo-opus-ada cross-referenced by PR #18270
- 2026-09-04T11:25:17Z @neo-opus-ada cross-referenced by #46
- 2026-09-04T20:43:07Z @neo-opus-vega cross-referenced by #51
- 2026-09-08T07:51:41Z @neo-opus-grace cross-referenced by #18465
- 2026-09-08T07:53:11Z @neo-opus-grace cross-referenced by PR #18466
- 2026-09-15T17:39:55Z @neo-opus-vega cross-referenced by #78
- 2026-09-16T08:58:36Z @neo-opus-vega cross-referenced by #80
- 2026-09-18T10:40:24Z @neo-opus-ada cross-referenced by PR #89
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90
- 2026-09-18T16:28:35Z @neo-opus-ada cross-referenced by #91
- 2026-09-23T12:33:15Z @neo-opus-ada cross-referenced by PR #108
- 2026-09-25T21:58:27Z @neo-opus-grace added the `enhancement` label
- 2026-09-25T21:58:27Z @neo-opus-grace added the `ai` label
- 2026-09-25T21:58:27Z @neo-opus-grace added the `build` label
- 2026-09-25T21:58:27Z @neo-opus-grace added the `agent-os` label
- 2026-09-25T21:58:33Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-25T21:58:36Z @neo-opus-grace added parent issue #14
- 2026-09-25T22:02:55Z @tobiu referenced in commit `19014f4` - "ci: leave agent-preflight's stale workflow reference for its own change (#523)"
- 2026-09-25T22:02:55Z @tobiu referenced in commit `ed582d9` - "test(memory-core): the WriteAhead spec's comments keep their reasons and drop review provenance (#523)"
- 2026-09-25T22:03:27Z @neo-opus-grace cross-referenced by PR #524
- 2026-09-25T22:04:39Z @tobiu referenced in commit `c535413` - "test(memory-core): AC-3 red arm, a tracking ref in a durable comment, reverted next (#523)"
- 2026-09-25T22:05:41Z @tobiu referenced in commit `5f12a23` - "test(memory-core): AC-3 red arm reverted (#523)"

