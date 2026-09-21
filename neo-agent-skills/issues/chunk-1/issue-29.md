---
id: 29
title: 'The PR-body lint gate belongs to the shared baseline, not to one repository'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-08-31T08:07:22Z'
updatedAt: '2026-08-31T17:01:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/29'
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
closedAt: '2026-08-31T17:01:15Z'
---
# The PR-body lint gate belongs to the shared baseline, not to one repository

> **Corrective comment RETIRED 2026-08-31**, on @neo-gpt's RA-2 (PR #31). This body promised the gate would post bounded corrective guidance on `opened`. It ships posting nothing, deliberately, and the three places that still promised otherwise are corrected below.
>
> **Why the side effect is not worth its cost.** The failure output names AT MOST ONE missing anchor and never an invisible one, because a message enumerating the full set is a template an agent can satisfy without writing the sections. A corrective comment is a strictly more enumerating surface, so shipping one would undo the anti-stuffing property the visible/invisible anchor split exists to create. It also costs privilege: a reusable workflow cannot grant itself `issues: write`, so every consumer would have to hand a shared workflow comment-write access to deliver a message the failed check already carries.

## Context

Sub of #14, and filed against my own third instance of the same mistake. `neomjs/neo#17911` and `#17913` were closed as superseded by this Epic after @tobiu caught them; `#17916` is the third, and he caught that one too — by title.

`pull-request-workflow.md` §9 — loaded by **every agent in every consuming repository** — makes this promise verbatim:

> `agent-pr-body-lint.yml` enforces `Evidence:`, `## AC Evidence`, `## Test Evidence`, `## Post-Merge Validation`, `## Deltas`, `Authored by ` as **unconditional** anchors.

**That workflow exists in no repository.** It was deleted from the Engine on 2026-08-26 (`91ae3604b4`, 253 lines) and re-homed nowhere — not Engine, not Brain, not here. Every clause of that sentence is currently false, and the seats reading it have no way to know.

It has a measured victim: **`neomjs/neo` PR #17902 merged at 06:31:27Z carrying no `## AC Evidence` and no `## Test Evidence`**, past two review rounds and green CI. That PR is mine; the body was repaired at 06:34:50Z, three minutes *after* the merge.

## The Problem

The Engine-local restoration (`neomjs/neo#17916` / PR #17917) is written, reviewed and approved — and it is the wrong address, for a reason I asserted wrongly on that PR and am correcting here.

**My argument was that a shared version "needs the cross-repository caller decision #14 owns." It does not.** `reusable-pr-baseline.yml` is `on: workflow_call`; **callers own their trigger**. This gate's trigger is `pull_request: [opened, edited, synchronize, ready_for_review]` — the same event family the baseline already serves. The caller mechanism exists; the Engine simply does not call it.

I also cited #22 as support, and #22 argues the opposite: it excludes the **review**-body lint from `reusable-pr-baseline.yml` *because* `pull_request_review` and `pull_request` have different semantics. This gate is `pull_request`. That exclusion is a reason to fold this one in.

**Restoring it Engine-local makes §9's promise true in one repository and leaves it false in four.**

## The Architectural Reality

- **The other baseline jobs invoke a packaged CLI**, not inline logic: `source-comment-archaeology` runs `neo-agent-skills-ticket-archaeology` from a pinned release in `runner.temp`. The deleted gate is **253 lines of inline `github-script`** inside YAML.
- **That inline shape is why #28 cannot be fixed where it lives.** #28 records that the anchor gate is satisfied by *naming* an anchor in prose — `body.includes(anchor)` against the whole body, so a PR that never carries the section passes by mentioning it. Measured on PR #17917: deleting the `## Deltas` heading left **3 surviving prose occurrences** and the check stayed green; only erasing every occurrence of `Authored by ` reddened it. Fixing substring-vs-heading semantics inside a YAML string is untestable; fixing it in a `scripts/check-pr-body.mjs` with a `scripts/test-check-pr-body.mjs` contract is routine.
- **The gate needs `pull-requests: read`** for the live body fetch, and nothing more. `reusable-pr-baseline.yml` declares `permissions: contents: read`, so the job carries its own read grant. It needs **no write grant**: with the corrective comment retired, there is no side effect to authorize, which is what keeps the no-elevation rule satisfiable for every caller.
- **The live-body-fetch property is load-bearing and must survive the move.** The gate reads the PR through `github.rest.pulls.get` rather than `context.payload.pull_request`, so a body corrected after a red run greens on the `edited` event **without a push**. Demonstrated at unchanged head `9d2d23d12f`: [red](https://github.com/neomjs/neo/actions/runs/33366874780) → [green](https://github.com/neomjs/neo/actions/runs/33366986918).

## The Fix

1. `scripts/check-pr-body.mjs` — the validator as a portable CLI with a `bin` entry, following `check-ticket-archaeology.mjs`. It receives resolved facts (body, action, anchors) and returns findings; it does not fetch or comment.
2. `scripts/test-check-pr-body.mjs` — the mutation-sensitive contract this package requires of its guards.
3. A `pr-body` job in `reusable-pr-baseline.yml`: pinned release into `runner.temp`, `working-directory` pinned to `github.workspace`, its own least-privilege `permissions` block, one stable check name. The wrapper owns the live fetch; the CLI owns the decision. Neither posts a comment.
4. Callers: one `uses:` coordinate per repository, tracked under #14.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `pr-body` job in `reusable-pr-baseline.yml` | This package (#14: Skills owns non-product PR governance) | Runs a pinned validator against the live PR body; fails on a missing anchor or a `Resolves` violation; posts no comment — the failed check carries the finding | None — the alternative is the current state, no gate in any repository | README beside the archaeology and substrate guards | a seeded anchor-removal fixture reds the job; a body-edit repair greens it with no push; a caller PR editing the validator in its own tree **still reds** |
| `neo-agent-skills-pr-body` bin | `scripts/check-pr-body.mjs` | Reports missing anchors and forbidden close-keywords from a supplied body | Exits non-zero on any finding | script header | `test-check-pr-body.mjs`, including the #28 heading-vs-substring arm |

Row anchors verified before assertion: `reusable-pr-baseline.yml`'s `workflow_call` trigger, job list and archaeology install steps read at Skills `dev`; the deleted workflow's anchor arrays, `if:` condition and live-fetch call read at `neomjs/neo@91ae3604b4^`; the run receipts are linked above.

## Decision Record impact

`none` — completes the npm Skills SSOT #14 already selected, and removes an instance of the duplication it exists to end.

## Acceptance Criteria

- [ ] `scripts/check-pr-body.mjs` exists with a `bin` entry, `files[]` entry, and a mutation-sensitive `test-check-pr-body.mjs`.
- [ ] All six §9 anchors and the `Resolves #N` / forbidden-`Closes`/`Fixes` rules are enforced, matching the arrays at `neomjs/neo@91ae3604b4^`.
- [ ] **#28 is closed by construction:** deleting a *heading* while leaving prose mentions reds the check. A fixture reproduces PR #17917's body, where 3 surviving prose occurrences kept the old gate green.
- [ ] A `pr-body` job in `reusable-pr-baseline.yml` installs a pinned release into `runner.temp`, pins `working-directory` to `github.workspace`, and declares its own least-privilege `permissions`.
- [ ] The live-body-fetch property survives: a body-only repair greens the check with `HEAD` unchanged, asserted rather than assumed.
- [ ] A caller PR that edits the validator in its own tree **still fails** — the gate is not disarmable by the diff it guards.
- [ ] **The PR body never reaches a shell.** The job must not interpolate body text into a `run:` block; the wrapper writes it to a file and the CLI reads `--body-file`. Transferred from `neomjs/neo` PR #17917 AC-4, which is the only place this property was written down — the deleted gate had zero `run:` steps by design, and a naive port that pipes the body through `run:` re-opens shell injection on attacker-controlled text.
- [ ] The workflow-contract suite asserts the new job's stable name, pinned install, absolute bin path and workspace pin, each with a negative mutation.
- [ ] `neomjs/neo#17916` closes as superseded and PR #17917 closes unmerged.

## Out of Scope

- Per-repository caller rollout — #14 states those have distinct owners and merge boundaries.
- The AC-Evidence **content** check and the stacked-PR guard. Both need repository state beyond the body; they are `neo-agent-skills#24`'s mechanism and stay routed there.
- The review-body lint — that is #22, a different event (`pull_request_review`) and deliberately a separate workflow.
- Changing the six anchors themselves. This relocates and repairs enforcement; it does not renegotiate what a body must contain.

## Avoided Traps

- **Restoring Engine-local "as a stopgap."** Tempting — the Engine has been unlinted since 2026-08-26 with a demonstrated victim. But once a repository holds the gate, moving it reads as deleting coverage, which is exactly what made `#17911` and `#17913` harder to correct. Named as a real cost rather than dismissed: the gap persists until this lands.
- **Porting the inline script verbatim into a job.** It would carry #28 with it and stay untestable. The package's own pattern — CLI plus `test-*.mjs` — is what makes the defect fixable.
- **Folding it into an existing job.** #22 is explicit that event semantics must not be flattened; this is `pull_request`, so it belongs in `reusable-pr-baseline.yml` but as its **own** job with its own permissions.

## Related

Sub of #14. Related: #28 (the substring defect this closes by construction), #22 (sibling, review-body, different event), #24 (the local preflight, unrunnable outside the Brain), `neomjs/neo#17916` and PR #17917 (superseded by this leaf), `neomjs/neo#17902` (the measured victim), `neomjs/neo#14344` (the false-negative half already recorded).

Retrieval Hint: `query_raw_memories` for "agent-pr-body-lint deleted no repository anchor substring" · the mutation receipts and the 4×-occurrence measurement are on `neomjs/neo` PR #17917.

Origin Session ID: 56bc214a-5b55-41a2-848a-cdfa372abbcd




## Timeline

- 2026-08-31T08:07:23Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-08-31T08:07:23Z @neo-opus-grace added the `enhancement` label
- 2026-08-31T08:07:23Z @neo-opus-grace added the `ai` label
- 2026-08-31T08:07:23Z @neo-opus-grace added the `architecture` label
- 2026-08-31T08:07:24Z @neo-opus-grace added the `build` label
- 2026-08-31T08:07:31Z @neo-opus-grace added parent issue #14
- 2026-08-31T08:17:30Z @neo-opus-grace cross-referenced by PR #30
- 2026-08-31T09:40:12Z @neo-opus-grace cross-referenced by #24
- 2026-08-31T10:02:35Z @neo-opus-grace referenced in commit `3c255c8` - "feat(ci): the PR-body anchor validator as a portable guard (#29)

First slice of the relocation: the decision half only. The reusable-workflow job
follows once #26 lands, because that PR owns reusable-pr-baseline.yml and
stacking the two would conflict on one file.

`pull-request-workflow.md` §9 tells every agent in every consuming repository
that `agent-pr-body-lint.yml` enforces six anchors as unconditional. That
workflow was deleted from the Engine on 2026-08-26 and re-homed nowhere, so the
promise is currently false everywhere. It has a measured victim: neomjs/neo PR
review rounds and green CI.

The port closes #28 by construction. The old gate matched `body.includes(anchor)`
against the whole body, and every anchor also appears in bodies that DISCUSS the
anchors -- a table cell naming `## Test Evidence`, or a sentence explaining why a
section was omitted, satisfied it while the section was absent. Measured on the
restoration PR: deleting the `## Deltas` heading left three surviving prose
occurrences and the check stayed green.

Anchors are now matched as LINES. That covers both shapes without a second rule:
`## AC Evidence` is a heading and `Evidence:` / `Authored by ` are line-initial
prefixes, so all three open a line, while a mid-sentence mention, a table row
(which opens with `|`), or a fenced snippet does not. Leading whitespace is
tolerated because indentation is formatting, not evasion.

Pure and transport-free: it receives a body and returns findings, never fetching
a pull request or posting a comment. That is the property the inline home could
not have -- 253 lines of `github-script` inside YAML cannot carry a test
contract, which is why the substring defect survived there.

The visible/invisible split is preserved and so is its reason: the failure
message names at most ONE anchor and never an invisible one, because a message
enumerating the set is a template an agent can satisfy without writing the
sections. A spec arm asserts the invisible anchors never reach the output.

Mutation-verified: restoring `body.includes` reds the prose-mention arm for every
anchor, and only that arm."
- 2026-08-31T13:29:56Z @neo-opus-grace cross-referenced by PR #31
- 2026-08-31T14:29:39Z @neo-opus-grace referenced in commit `f31ccfa` - "fix(ci): the PR-body job judges agent PRs, not every contributor (#29)

RA-1 from @neo-gpt, and it is the one that mattered. The deleted
`agent-pr-body-lint.yml` gated every step on `startsWith(login, 'neo-') ||
contains(labels.*.name, 'ai')` — its own comment says it "deliberately skips"
human-authored PRs. I ported the validator and left the boundary behind, so the
wrapper would have held every contributor to §9, the AGENT pull-request protocol.

That is a policy change, not a port, and nobody decided it.

Restored on both judging steps. Gated at the STEP rather than the job so the check
still REPORTS on a human PR: a skipped job cannot serve as a required status
context, which is what #14 needs these to become.

The contract counts the boundary rather than testing its presence. One gated step
and one ungated still runs the agent template against a human PR through whichever
half lost its condition, and a presence check passes on a single surviving
occurrence — so the assertion is `=== 2`, with a mutation that removes it from one
step only.

I had this fact in hand and dropped it: `neomjs/neo` PR #17917's body, which I
read and closed this morning, states that the author gate lives inside the script
as `isAgentAuthor` / `hasAiLabel` with an early return. I carried the property
that would have died with that PR and lost the one written plainly in its body.

39 negative mutations pass."
- 2026-08-31T14:29:40Z @neo-opus-grace referenced in commit `7856d39` - "fix(ci): retire the corrective comment, document the caller trigger, follow the value (#29)

RA-2, RA-3 and RA-4 from @neo-gpt.

RA-2 — the corrective comment is RETIRED, not implemented, and the reason comes
from this design rather than from effort. The failure output names at most ONE
anchor and never an invisible one, because a message enumerating the set is a
template an agent can satisfy without writing the sections. A corrective comment
is a strictly MORE enumerating surface, so shipping one would undo the
anti-stuffing property the visible/invisible split exists to create. It also costs
privilege: a reusable workflow cannot grant itself `issues: write`, so every
consumer would hand a shared workflow comment-write access to deliver a message
the failed check already carries. The CLI JSDoc no longer promises it.

RA-3 — the caller trigger is now stated where callers read it. `edited` is the one
easy to omit and expensive to miss: this job reads the LIVE body precisely so a
corrected body greens with no push, and without that event the live fetch buys
nothing — the author fixes the body, nothing re-runs, and the red stands until an
unrelated commit arrives. A reusable workflow cannot see its caller's `on:` block,
so this is a caller obligation, tracked under #14, and the header says so rather
than implying an assertion that cannot exist.

RA-4 — three guards that follow the VALUE instead of matching a token:

  (a) the fetched response must be what reaches the file. A `pulls.get` whose
      result is discarded while an event-payload alias gets written satisfies
      every presence check; the guard reads what `writeFileSync` is handed.
  (b) no body value may reach a shell-visible surface, `env:` included — an env
      var carrying the body is interpolated exactly like an inline expression,
      and my previous check only looked at `run:` lines.
  (c) the install must be pinned. `neo-agent-skills@latest` keeps the package
      name and changes what executes.

Each has a mutation that reds for its own named property while the canonical file
and every existing control stay green. 42 negative mutations; all four sibling
suites and corpus lint exit 0."
- 2026-08-31T17:01:16Z @tobiu referenced in commit `b419d0b` - "Merge pull request #31 from neomjs/feat/pr-body-baseline-job-29

feat(ci): the PR-body gate becomes a shared baseline job (#29)"
- 2026-08-31T17:01:16Z @tobiu closed this issue
- 2026-09-03T19:47:14Z @neo-opus-grace cross-referenced by #41
- 2026-09-07T00:14:47Z @neo-opus-grace cross-referenced by #56

