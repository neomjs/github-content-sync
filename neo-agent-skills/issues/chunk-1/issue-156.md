---
id: 156
title: Harden published Skills scripts against seven CodeQL findings
state: CLOSED
labels:
  - bug
  - ai
  - build
  - security
assignees:
  - neo-gpt
createdAt: '2026-10-10T13:49:44Z'
updatedAt: '2026-10-10T16:19:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/156'
author: neo-gpt
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
closedAt: '2026-10-10T16:15:45Z'
---
# Harden published Skills scripts against seven CodeQL findings

## Context

The operator escalated the seven open [Skills code-scanning alerts](https://github.com/neomjs/neo-agent-skills/security/code-scanning) on 2026-10-10. At intake, the latest analysis and then-current `dev` both named `6c27692ab378b57a49e088e7d0e53731cb30bf95`. This is one bounded repair of four published scripts and their existing contract tests.

| Alert | CodeQL classification | Exact source at the scan head |
| --- | --- | --- |
| [1](https://github.com/neomjs/neo-agent-skills/security/code-scanning/1) | High, polynomial ReDoS | [check-secrets.mjs:69](https://github.com/neomjs/neo-agent-skills/blob/6c27692ab378b57a49e088e7d0e53731cb30bf95/scripts/check-secrets.mjs#L69) |
| [2](https://github.com/neomjs/neo-agent-skills/security/code-scanning/2) | High, polynomial ReDoS | [check-commit-authorship.mjs:145](https://github.com/neomjs/neo-agent-skills/blob/6c27692ab378b57a49e088e7d0e53731cb30bf95/scripts/check-commit-authorship.mjs#L145) |
| [3](https://github.com/neomjs/neo-agent-skills/security/code-scanning/3) | High, polynomial ReDoS | [check-workflow-concurrency.mjs:46](https://github.com/neomjs/neo-agent-skills/blob/6c27692ab378b57a49e088e7d0e53731cb30bf95/scripts/check-workflow-concurrency.mjs#L46) |
| [4](https://github.com/neomjs/neo-agent-skills/security/code-scanning/4) | High, polynomial ReDoS | [check-workflow-concurrency.mjs:66](https://github.com/neomjs/neo-agent-skills/blob/6c27692ab378b57a49e088e7d0e53731cb30bf95/scripts/check-workflow-concurrency.mjs#L66) |
| [5](https://github.com/neomjs/neo-agent-skills/security/code-scanning/5) | High, polynomial ReDoS | [check-workflow-concurrency.mjs:115](https://github.com/neomjs/neo-agent-skills/blob/6c27692ab378b57a49e088e7d0e53731cb30bf95/scripts/check-workflow-concurrency.mjs#L115) |
| [6](https://github.com/neomjs/neo-agent-skills/security/code-scanning/6) | High, polynomial ReDoS | [generate-agents-md.mjs:77](https://github.com/neomjs/neo-agent-skills/blob/6c27692ab378b57a49e088e7d0e53731cb30bf95/scripts/generate-agents-md.mjs#L77) |
| [7](https://github.com/neomjs/neo-agent-skills/security/code-scanning/7) | Medium, shell construction from library input | [check-commit-authorship.mjs:170](https://github.com/neomjs/neo-agent-skills/blob/6c27692ab378b57a49e088e7d0e53731cb30bf95/scripts/check-commit-authorship.mjs#L170) |

These are scanner ratings, not a claim of exploitation. At intake, all seven alerts were open. The verified closeout below records their final state.

## The Problem

Bounded disposable-process probes against the exact source reproduce superlinear matching at all six regex sites. On Node 25.9.0, each crossed a hard two-second child-process timeout with an input below 1 MB; 16 semantic controls passed. These are diagnostic probes, not a full-suite result. The costly cases are malformed or near-matching text: allow-marker lines containing a credential and a Unicode line separator, malformed co-author trailers, concurrency declarations with long whitespace near misses, repeated `github.run_attempt` without the terminating `github.run_id`, and newline-heavy section bodies. Ordinary matching text alone is not a sufficient regression control.

A separate exact-source probe of exported `run(args, payload)` intercepted the child-process API: a caller-supplied `--base` containing shell metacharacters reaches the `execSync` command string unchanged. It executed no injected command. The shipped reusable workflow obtains `BASE_SHA` from GitHub's PR base and passes it as a quoted CLI argument; that caller is not established as attacker-controlled. The library/CLI boundary nevertheless constructs a shell command unnecessarily.

## The Architectural Reality

These files already own the work: `findSecrets`, `findUnknownCoAuthors` / `pendingRanges` / `run`, `collectConcurrencyBlocks` / `gradeBlock`, and `parseSection`. They are package executables or generator exports with existing Node contract runners in `scripts/test-secrets.mjs`, `test-commit-authorship.mjs`, `test-workflow-concurrency.mjs` and `test-generate-agents-md.mjs`.

The owning JSDoc requires meaningful allow reasons, caller-owned roster and authorship policy, rerun-safe cancellation, and deterministic section generation. This repair changes input handling while preserving those contracts. The original authorship boundary and rebase exclusions are settled by `#93` and `#109`.

## The Fix

Replace the flagged backtracking searches with bounded line/token/delimiter parsing or equivalent linear scans. For Git reads, pass executable and arguments through a non-shell process API; separate range values from fixed revision options and refuse malformed/option-bearing inputs before they can alter Git invocation. Preserve valid hook tuples, deletion/new-branch/rebase behavior, remote exclusions, and the documented CI base form.

Extend the existing test runners with the reproducing near misses, valid/invalid semantic controls, and isolated child-process deadlines. Keep generated instruction text unchanged, including LF-only trailing-newline removal; generic whitespace trimming would alter valid output. One ticket and one reviewable PR can cover this scan cohort; no new parser subsystem or dependencies are required.

## Contract Ledger

| Surface | Authority | Behavior / fallback | Docs | Evidence |
| --- | --- | --- | --- | --- |
| `findSecrets` and allow reasons | Existing patterns and JSDoc in `check-secrets.mjs` | Same findings/redaction/reason requirement; malformed marker never bypasses scanning | Existing function JSDoc | Existing secret controls plus alert-1 near miss |
| Authorship runner, ranges and trailers | `check-commit-authorship.mjs`; `#93` / `#109` | No shell interpretation; valid range/exclusion and blocking/advisory policy retained; unreadable/malformed invocation fails visibly | Range/runner/trailer JSDoc | Safe process-boundary probe, range controls, alert-2 near miss |
| Workflow concurrency extraction and grading | `check-workflow-concurrency.mjs` | Same inline/block counts, cancellation selection and rerun-clause verdict; no backtracking growth | Existing parser/grader JSDoc | Existing workflow controls plus alerts 3–5 |
| Section parsing | `parseSection` in `generate-agents-md.mjs` | Same declarations and body bytes, deterministic output and missing-frontmatter refusal | Existing parser JSDoc | Existing generation controls plus alert 6 |

## Acceptance Criteria

- [x] All seven rows have an explicit implementation disposition and focused regression evidence in the same PR.
- [x] Caller-controlled range text never reaches a shell interpreter; metacharacters and Git-option-like input cannot execute commands or widen the intended read. Valid hook/CI ranges and rebase exclusions retain their behavior.
- [x] All six reproduced regex cases terminate within an isolated test deadline without the measured backtracking growth; semantic positive and negative controls still distinguish valid input from malformed input.
- [x] Secret findings remain redacted and require genuine allow reasons; trailer policy and concurrency verdicts remain unchanged; generated instruction output is byte-identical for the current valid corpus.
- [x] The four owning contract runners, full package test command and skill-corpus lint pass on the final head.
- [x] Final-head CodeQL covers the changed scripts with no unresolved findings from this cohort. **Post-merge:** link the new `dev` analysis and verify each of the seven alerts is closed as fixed; a local green test is not scan closure.
- [x] The normal package release carries the repair; consumer adoption continues through existing #144 without a second tracking lane.

## Decision Record impact

`none`: source hardening preserves existing policy and generated instruction content. Brain structure-map was run; placement remains the established Skills `scripts/` directory and existing sibling tests. No new production `.mjs`, skill rule, config leaf or architectural authority is prescribed.

## Out of Scope

Scanner suppression/dismissal, scanner-severity changes, credential rotation or incident-response claims, changes to team/roster policy, new dependencies, generic YAML parsing, unrelated guard cleanup, and FM/release feature changes.

## Avoided Traps

Do not accept only ordinary happy-path timing, silently truncate input into a successful verdict, split shell strings on whitespace, remove remote exclusions, widen allow markers, or silence CodeQL instead of repairing the source.

## Verified closeout

PR #158 merged into `dev` at `1105e1e22040c54c05088a11cb12a006c87c5ccf` on 2026-10-10. The final-head checks passed, and the new default-branch JavaScript CodeQL analysis `1929069456` reports zero results and no error. The previous analysis `1929008031` at `67a7634f` had seven results, preserving the before/after control.

GitHub's alert API was independently re-read: alerts **1–7** all have `state: fixed`, `fixed_at: 2026-10-10T16:16:40Z`, and `dismissed_at: null`. The open-alert query returns none. The source sites are repaired; none was suppressed or dismissed.

The normal [Publish run](https://github.com/neomjs/neo-agent-skills/actions/runs/38066943086) succeeded, and npm independently returns `neo-agent-skills@0.1.34`. Consumer delivery remains on existing #144; this source repair does not claim fresh-session adoption.

## Creation checks and ownership

Live latest-open sweep: latest 20 open Skills issues re-read immediately before creation on 2026-10-10; no equivalent. All-state security/CodeQL/backtracking searches and the open PR queue found no repair. `#51` is operator-identity shell-guard materialization, `#155` is architecture-review behavior, and own open `#140` is adoption validation; none covers these source sites.

A2A in-flight sweep: latest 30 messages across read states rechecked; no competing security-cohort claim. MC queries on the affected scripts and observed symptoms returned unrelated history, so no prior decision is inferred. KB returned unrelated archived material; exact source, current alerts and settled authored contracts are the evidence.

Owner: Euclid (`neo-gpt`). Operator escalation admits this independently actionable security work; existing neomjs/neo-agent-institution#596 ownership remains intact.

Origin Session ID: 0a0bd542-0f17-4244-a59e-cec534621a5c
Retrieval Hint: "Skills seven CodeQL alerts regex backtracking commit authorship shell range scan cohort"



## Timeline

- 2026-10-10T13:49:45Z @neo-gpt assigned to @neo-gpt
- 2026-10-10T13:49:47Z @neo-gpt added the `bug` label
- 2026-10-10T13:49:47Z @neo-gpt added the `ai` label
- 2026-10-10T13:49:47Z @neo-gpt added the `build` label
- 2026-10-10T13:49:47Z @neo-gpt added the `security` label
- 2026-10-10T15:08:14Z @neo-gpt referenced in commit `a38fe07` - "fix(security): bound guard parsing and Git execution (#156)"
- 2026-10-10T15:10:50Z @neo-gpt cross-referenced by PR #158
- 2026-10-10T15:56:37Z @neo-gpt referenced in commit `2b192d3` - "chore(merge): integrate dev for security release (#156)"
- 2026-10-10T16:15:45Z @tobiu referenced in commit `1105e1e` - "fix(security): bound guard parsing and Git execution (#156) (#158)"
- 2026-10-10T16:15:45Z @tobiu closed this issue
- 2026-10-10T16:24:16Z @neo-gpt cross-referenced by PR #160

