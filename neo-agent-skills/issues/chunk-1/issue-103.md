---
id: 103
title: 'Nothing checks Resolves on a contributor PR, and the green check says otherwise'
state: OPEN
labels:
  - bug
  - contributor-experience
  - ai
  - github_actions
assignees:
  - neo-opus-ada
createdAt: '2026-09-21T16:40:36Z'
updatedAt: '2026-09-23T12:28:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/103'
author: neo-opus-ada
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
# Nothing checks Resolves on a contributor PR, and the green check says otherwise

## Context

@tobiu, 2026-09-21, on [neomjs/neo#19044](https://github.com/neomjs/neo/pull/19044): *"sigh. now we get PRs without resolves."*

@tobiu, 2026-09-23, after I opened neomjs/neo-agent-brain#428 with `Refs #425` because three of #425's ACs were observable only after deploy: *"a PR MUST ALWAYS resolve ONE ticket. zero exceptions."* That removes the draft deferral the Fix below originally kept. The substrate still grants it in three places: the CLI's `--draft` flag, `pull-request-workflow.md`'s "Draft-only exception", and `ticket-create-workflow.md` §1d ("Any pre-marker PR stays draft with `Refs #N`"). The same payload also permits several `Resolves` lines ("Multiple delivered tickets get one standalone line each"). A ticket whose ACs a PR cannot deliver is re-scoped, with post-merge checks moved to `## Post-Merge Validation`. The keyword is never downgraded. *(Scope extended 2026-09-23 by the author.)*

That PR has no closing keyword. The rule it misses is not template preference — it is repo policy, stated by the operator the same day: **every PR resolves exactly one ticket, and fully delivers it.** `Resolves` rather than `Closes`, because an issue can be closed without being resolved.

The contributor did nothing wrong on this one; the scope question was genuinely open and they raised it instead of guessing. What is wrong is that **no guard would have caught it either way, and the check that looks like it did, didn't.**

## The Problem

`reusable-pr-baseline.yml`'s **PR body** job gates its *steps* — not the job — on authorship:

```yaml
if: ${{ startsWith(github.event.pull_request.user.login, 'neo-')
     || contains(github.event.pull_request.labels.*.name, 'ai') }}
```

The job's own comment says why the job is not gated: *"a skipped job cannot serve as a required status context, which is what `#14` needs these to become."* That reasoning is right. The consequence is not: on a contributor PR both steps skip, the job succeeds, and **`baseline / PR body` reports green having validated nothing.**

Measured on [neomjs/neo#19032](https://github.com/neomjs/neo/pull/19032), same contributor: `baseline / PR body` = SUCCESS, while the body at that moment carried `Refs #19026` and **no closing keyword at all**. A vacuous pass that reads identically to a real one.

**And the only opt-in is all-or-nothing.** The `ai` label switches on the *entire* agent protocol. Run against #19044's body:

```
$ neo-agent-skills-pr-body --body-file 19044-body.md
❌ Agent PR body is missing required template anchors.
   First missing: Evidence:
exit 1
```

The first complaint is `Evidence:` — an agent-only anchor — and the missing close target is not what it names. So labelling a contributor's PR `ai` reds it for six sections nobody told them about, which is precisely the outcome the job's comment warns against: *"Widening it to every contributor would hold human PRs to a template nobody agreed they must follow, and that is a policy change, not a port."*

**That warning is correct about the six anchors and wrong about one rule.** `## AC Evidence`, `## Test Evidence`, `## Post-Merge Validation`, `## Deltas`, `Evidence:` and `Authored by ` are §9 agent protocol. **The close target is not** — it is what the repository requires of every pull request, and CONTRIBUTING.md and `PULL_REQUEST_TEMPLATE.md` already say so to humans. Bundling the two means the universal rule can only be enforced by imposing the agent-only ones, so in practice it is enforced on nobody outside the team.

## The Architectural Reality

- `neo-agent-skills-pr-body` validates the anchor set and the close target together, as one pass over one body.
- The caller (`neomjs/neo`'s `pr-baseline.yml`) pins the reusable workflow by SHA and passes only roster inputs, so the split has to happen in this repo, not per consumer.
- `#14` wants these jobs to become **required status contexts**. A context that is green-when-inapplicable can be made required today and still gate nothing, which is worth knowing before it is marked required rather than after.
- The live-fetch design (body read via `pulls.get`, not from the event payload, so an edited body re-validates) is right and this ticket does not touch it.

## The Fix

Split the validation into two claims over the same body:

1. **Close target — ungated, every pull request.** Exactly one standalone `Resolves #N`; `Closes` / `Fixes` rejected; comma-separated `Resolves #X, #Y` rejected; a second standalone `Resolves` line rejected; **no draft exception** (the `--draft` relaxation goes, and the two payload passages that grant it are rewritten to the one-ticket rule).
2. **Agent anchors — gated exactly as now**, on `neo-` login or the `ai` label.

The CLI grows a flag selecting which claim to run (`--close-target-only`, or the inverse), and the workflow runs (1) for everyone and (2) under the existing condition. Contributors get the one rule that is genuinely theirs, in a message that names it, and no agent anchor ever appears in their check output.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `findBodyViolations({body, isDraft})` in `scripts/check-pr-body.mjs` | operator rulings 2026-09-21 / 2026-09-23 | exactly one standalone `Resolves #N`; `isDraft` no longer relaxes it | two `Resolves` lines → violation naming the one-ticket rule | JSDoc | unit arms per AC |
| `neo-agent-skills-pr-body` CLI flags | same | `--draft` removed; a flag selects the close-target claim alone for non-agent PRs | an unknown flag fails loudly, never silently | CLI usage text | CLI runs before/after |
| `reusable-pr-baseline.yml` `pr-body` job | #14 (reporting job) + the rulings | body read and close target checked for EVERY PR; agent anchors stay gated on `neo-` / `ai` | a draft is judged like a ready PR | workflow comments | workflow-contract suite |
| `pull-request-workflow.md` close-target rules · `ticket-create-workflow.md` §1d | the rulings | draft-only exception and "multiple delivered tickets" line removed; pre-marker work opens no PR | — | the payloads | lint + byte delta |

## Intake (2026-09-23, author, drift probe fired on `reusable-pr-baseline.yml` via #104's version pins only)

Prescription checked: `scripts/check-pr-body.mjs` — owns the concern (the close-target logic and the draft relaxation both live in `findBodyViolations`). Live CLI 0.1.15: a draft body with only `Refs #1` passes; a body with two standalone `Resolves` lines passes; the workflow gates the body read on `neo-` / `ai` and exports `--draft` for drafts. Verdict: valid-as-written with the 2026-09-23 extension.

## Acceptance Criteria

- [ ] AC-1 — A PR from a non-`neo-` author with no closing keyword **fails** `baseline / PR body`, and the failure message names the close target and nothing else.
- [ ] AC-2 — The same PR with a valid standalone `Resolves #N` passes, with no agent anchor required and none mentioned.
- [ ] AC-3 — An agent-authored PR still fails on a missing anchor exactly as today; the gated behaviour is unchanged. Pinned by an arm, not asserted.
- [ ] AC-4 — `Closes` / `Fixes`, and `Resolves #X, #Y`, are rejected for every author, not only agents.
- [ ] AC-5 — Mutation-checked in both directions: the close-target arm reds when the keyword is removed, and the anchor arm reds when an anchor is removed — run separately, so one cannot mask the other. The vacuous-green failure mode this ticket is about is exactly what an unmutated arm would reproduce.
- [ ] AC-6 — Net loaded-bytes accounted per the accretion rule.
- [ ] AC-7 — A **draft** PR without a standalone `Resolves #N` fails exactly like a ready one; `--draft` no longer relaxes the close target. `pull-request-workflow.md`'s draft-only exception and `ticket-create-workflow.md` §1d's pre-marker `Refs` are rewritten to the one-ticket rule.
- [ ] AC-8 — A body with two standalone `Resolves` lines fails, and the message says a PR resolves exactly one ticket; the payload's "multiple delivered tickets" line is rewritten to match.

## Out of Scope

- Making these jobs required status contexts. That is `#14`.
- The six agent anchors' content or count.
- Fork workflow-approval policy. Related and separate: #19044's eight workflow runs currently sit in `action_required` because the author has no merged PR yet, so **none** of the guards ran on it — including this one. That is a GitHub repository setting and a human decision about executing untrusted code, not a substrate change.

## Avoided Traps

- **Reading a green check as a performed check.** `baseline / PR body` was green on #19032 while validating nothing. A gated step inside an ungated job reports success for "not applicable" in the same colour as "passed".
- **Reaching for the `ai` label as the fix.** Measured before proposing: it makes `Evidence:` the first failure on a contributor's PR. It is an agent-protocol switch, not a policy switch.
- **Trusting a piped exit code.** My first run of the CLI was piped into `head` and reported `exit=0` beside a `❌`; re-run unpiped it is `exit 1`. The guard is sound — the instrument was not. Same family as the `git commit | head` trap already on record.

## Decision Record impact

`none` — splits an existing guard's applicability; introduces no new authority.

## Related

- [neomjs/neo#19044](https://github.com/neomjs/neo/pull/19044) — the trigger; no closing keyword, no guard run
- [neomjs/neo#19032](https://github.com/neomjs/neo/pull/19032) — the vacuous green, measured
- [neomjs/neo#19033](https://github.com/neomjs/neo/issues/19033) / neomjs/neo PR #19035 — the template half of the same problem, merged today
- `#14` — Unify PR governance across Neo repositories; wants these as required contexts
- `#86` — the AC table is what the merge gate reads and nothing checks it against its body; adjacent, same job

## Sweeps

- Live latest-open sweep: 12 most recent open issues on this repo at 2026-09-21T16:39:55Z; no equivalent. `#86` is the nearest and concerns the AC table's contents, not the job's applicability.
- A2A in-flight claim sweep at 16:39Z: no `[lane-claim]` on the PR-baseline workflow.
- Own-assignment sweep: `#102`, `#76`, `#66`, `#63` are mine here; `#76` is the closest (an approval surviving an invalidating push) and is a different gate.

Origin Session ID: c54728f6-de9d-46a5-921f-aef7e79b91c8

Retrieval Hint: `query_raw_memories("PR body guard skips contributor authors vacuous green close target")`

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5 · Claude Code


## Timeline

- 2026-09-21T16:40:36Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-21T16:40:37Z @neo-opus-ada added the `bug` label
- 2026-09-21T16:40:38Z @neo-opus-ada added the `contributor-experience` label
- 2026-09-21T16:40:38Z @neo-opus-ada added the `ai` label
- 2026-09-21T16:40:38Z @neo-opus-ada added the `github_actions` label
- 2026-09-21T16:44:41Z @neo-opus-ada cross-referenced by PR #19044
- 2026-09-21T22:12:13Z @neo-opus-ada cross-referenced by #17171
- 2026-09-23T11:42:05Z @neo-opus-ada cross-referenced by #427
- 2026-09-23T12:33:15Z @neo-opus-ada cross-referenced by PR #108
- 2026-09-23T12:49:28Z @neo-opus-ada referenced in commit `118e188` - "fix(pr-body): a second GitHub closing expression is a second ticket (#103)

One canonical Resolves line no longer masks another close target: any
expression GitHub acts on outside that line -- any case, an optional colon,
an owner/repo#N or issue-URL target, inline prose -- is refused in both
scopes. Non-closing references (Refs, Related, see) stay green."

