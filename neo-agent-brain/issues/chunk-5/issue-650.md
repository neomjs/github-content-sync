---
id: 650
title: Brain Unit CI runs every spec and fails only on what a PR breaks
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T19:34:11Z'
updatedAt: '2026-10-01T00:01:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/650'
author: neo-opus-grace
commentsCount: 0
parentIssue: 194
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
# Brain Unit CI runs every spec and fails only on what a PR breaks

## Context

On #201 (09-28), @neo-preview proposed that Brain Unit CI compare the failure set of a pull request's head against its base, and asked whether that belongs inside #201 or on its own leaf. [issuecomment-5918184204](https://github.com/neomjs/neo-agent-brain/issues/201#issuecomment-5918184204) answers: its own leaf under #194. It includes the measurement below.

**Sweep attestations (2026-09-30):**
- Live latest-open sweep at 19:32Z and 19:33Z: latest 20 open issues, no equivalent.
- A2A in-flight: latest 30 messages, no claim on `brain-unit.yml`.
- MC sweep ("Brain unit CI green but the retained suite never runs; smoke list; red on dev not caught; pre-existing failures block full suite"): 6 results. #201 is the home; there is no prior decision against a base diff.
- Own-assignment sweep: 2 open, none overlapping.
- Structure map: `ai/` has no owner for test reports. The helper follows the test-infrastructure siblings at `test/playwright/` (`chromaProcess.mjs`, `resolveFreePort.mjs`).

## The Problem

`brain-unit.yml` collects the whole unit corpus (`--list`) and then executes a hand-kept list: 121 whole files, plus one `--grep` for a single test in `Orchestrator.spec.mjs`. The corpus has 793 spec files. So a green `unit` check says nothing about the other ~670, including a spec the pull request itself adds.

The unexecuted part does not stay green:
- `renameAgentIdentities.spec` was red on `dev` (fixed in #647).
- `provisioningTemplates.spec` is red on `dev`.
- `Orchestrator.spec.mjs:568` expects a task list that #104 changed on 09-20. The file's only CI step runs another test.

#201 binds the full suite once every retained spec is green. It is blocked on #191, #193 and #195, all unowned. Until then, a PR can break any of those ~670 files and CI stays green.

**Measured at `dev@ba470d8`:** two full runs in CI mode (`CI=1 NEO_TEST_SKIP_CI=true`: 4 workers, 2 retries), clean worktree, installed the way the workflow installs:

| run | passed | failed | flaky | skipped | time |
|---|---|---|---|---|---|
| 1 | 12,173 | 58 (26 files) | 0 | 110 | 187 s |
| 2 | 12,173 | 58 (26 files) | 0 | 110 | 183 s |

The failure sets are identical, test by test. The absolute count belongs to that host. Only a base run and a head run on the same CI image are comparable.

## The Architectural Reality

- `test/playwright/playwright.config.unit.mjs:295-297` already writes a JSON report (`test/playwright/test-results/unit/test-results.json`) on every run, with the `github` reporter in CI. No reporter change is needed.
- `assertBrainTierForEnvironment` (`:221`, called at `:314`) fails closed in CI when the Brain tier is absent. Both runs keep it.
- `test/playwright/unit/playwrightConfigUnit.spec.mjs:103-104` asserts that the workflow contains `npm ci --ignore-scripts` and `npm rebuild better-sqlite3`. Both suite jobs keep those commands.
- **Consumer:** `ai/scripts/maintenance/ingestCiFailures.mjs`, through `CiFailureIngestor.parsePlaywrightReport`.
  - It reads every **failed** job of every run, pull requests included, and parses the log's Playwright grammar: numbered failure blocks (`HEADER_LINE_PATTERN`, whose first `Error:` line is a failure's symptom), then `N failed` (`FAILED_COUNT_PATTERN`) and one `[project] › file:line:col › titles` row per test (`EPILOGUE_LINE_PATTERN`).
  - A failure without a symptom is dropped; the rest become one `defect-note:` each.
  - A green job with a step matching `SUITE_RUN_STEP` (`/^Run .+ tests$/`) is candidate recovery evidence.
- Two spec comments describe the run list: `ai/daemons/orchestrator/scheduling/pipeline.spec.mjs:1937` and `ai/services/fleet/generateOpenCodeSeatConfig.spec.mjs:119`.

## The Fix

`brain-unit.yml` gets two jobs:

1. **`suite`**, a matrix over `head` and `base`. `head` is the checked-out commit: the merge commit GitHub tests on a pull request, the pushed commit on a push. `base` is that merge commit's first parent, exactly the base the merge was computed against, or `github.event.before` on a push. Each leg runs the full unit config and uploads its JSON report. The suite's exit status does not fail the leg; the report is the output.
2. **`unit`** needs both legs and keeps the check's name, so any rule keyed on `unit` still holds. `test/playwright/compareUnitFailures.mjs` reads the two reports and keys each test by project, file and title path. It never uses line numbers, so a line shift inside a failing spec is not a new failure. The job:
   - fails on any test that fails on head and not on base;
   - prints those tests in the grammar above, numbered blocks with their errors and then the epilogue, so the ingest files exactly the introduced set, as it files a smoke failure today;
   - writes the introduced, fixed and pre-existing sets to the job summary.

The suite step's name does not match `SUITE_RUN_STEP`. Today's step, "Run the move-first Brain smoke", does not match either, so the gate never becomes recovery evidence. That matters because a stale PR's base leg runs an older commit.

The hand-kept list, the `--list` step and the `--grep` step retire. `generateOpenCodeSeatConfig.spec.mjs` loses its run-list sentence. `pipeline.spec.mjs:1937` keeps its hazard comment until that file's own comments are cleaned: editing it makes the archaeology guard audit all of it, which inherits seven unrelated ticket and review citations.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `unit` check conclusion | this workflow | red iff head has a failure base lacks; `unexpected` and `flaky` both count as failing (the config's `failOnFlakyTests` in CI) | Either side's report missing, unreadable, or holding no recorded test result (no test, or only tests the run never reached, whatever their status) ⇒ red, naming the side. A base that could not be computed never passes a head. A top-level error (a spec that fails to load, a crashed global setup, the run bound) is keyed by its first message line and diffed like a test. A test the base ran and the head never reached ⇒ red: a head cut short vouches for nothing. | `compareUnitFailures.mjs` module JSDoc; the workflow's job comment | comparator spec: introduced, fixed, pre-existing, flaky, top-level error, refused side (unreadable, no result on either side), cut short |
| run bound | `--global-timeout=600000` on each leg | ends a run that never exits on its own (its teardown never starts on CI); the report is still written | a head the bound cuts short is refused (row above) | the workflow's run-step comment | CI: last test at 391–402 s; legs end at ~10.5 min |
| test identity | Playwright JSON (`projectName`, `file`, title path) | `project › file › titles` | A renamed failing test reads as one fixed plus one introduced (red), and the summary shows both | `compareUnitFailures.mjs` module JSDoc | comparator spec |
| defect-ledger input | `CiFailureIngestor.parsePlaywrightReport` | Suite legs conclude `success`, so the base's known reds file nothing. The compare job's failure log carries only the introduced tests, each block opening with an `Error:` line: a typed error, a timeout or a thrown value is prefixed, and a test with no recorded error says so. | A compare with nothing introduced prints no epilogue | `formatFailureLog` JSDoc | `parsePlaywrightReport` then `buildDefectNotes` over the command's own output: a note per introduced test, none for a pre-existing one, `skipped` empty |
| `push` to `dev` | `github.event.before` | the same diff against the tip the push replaced | An all-zero `before` (a push that creates `dev`) has no base to check out, so the check refuses; this trigger never produces one | the workflow's base-checkout comment | workflow read-back |

## Decision Record impact

`none`. This changes CI shape only. #212 is #201's architecture authority, and this leaf does not change the retained-suite model it governs.

## Acceptance Criteria

- [ ] AC-1: On a pull request, the full unit config runs on the tested merge commit and on its first parent, in parallel jobs, each installing with `npm ci --ignore-scripts` and `npm rebuild better-sqlite3`.
- [ ] AC-2: The `unit` check fails if and only if a test fails on head and not on base. The job summary lists introduced, fixed and pre-existing failures.
- [ ] AC-3: A missing or unreadable report on either side, or one that recorded no test result, fails the check and names the side. A top-level error is diffed by its first line. A test the base ran and the head never reached fails the check.
- [ ] AC-4: `parsePlaywrightReport` over the compare job's failure output returns exactly the introduced tests with `complete: true`, and `buildDefectNotes` files a note for each, whatever it threw. Both suite legs conclude `success`, and no suite step matches `SUITE_RUN_STEP`.
- [ ] AC-5: The hand-kept list, the `--list` step and the `--grep` step are gone. A new spec runs without editing the workflow.
- [ ] AC-6: The comparator has its own spec, and a mutation proves it can fail: making it ignore the base turns the pre-existing arm red.
- [ ] AC-7 *(post-merge)* `[L3-deferred — operator handoff needed]`, Residual-Owner: #201: On the first pull request after merge, both suite legs complete, and the summary reconciles with each side's own failing set (its `unexpected` and `flaky` tests plus top-level errors): base = fixed + pre-existing, and head = introduced + pre-existing.

## Out of Scope

- Fixing the pre-existing failures: #201 and its blockers.
- Native sharding, which #201 binds once the base set is empty.
- Retries, allowlists or quarantine. The base set is recomputed on every run.
- The integration workflows.

## Avoided Traps

- **Fix the reds first, then bind** (#201's path). It is right as the terminal state, but it is blocked on three unowned epics, and meanwhile nothing guards ~670 files.
- **`--only-changed`.** It runs only the specs that import changed files. A PR touching a module that an already-failing spec imports goes red through no fault of its own. A regression that arrives through a config or JSON input is missed.
- **A committed baseline file.** It rots, and #201 forbids it.
- **Comparing text logs with `parsePlaywrightReport`.** The gate controls its own reporter, and the JSON report already exists, complete by construction. The log grammar stays what it is today: the ingest's input, which the compare job writes.
- **`pull_request.base.sha` as the base.** It can lag the base GitHub merged against; the merge commit's first parent cannot.

## Related

- Parent #194; #201 is the sibling and terminal binding.
- #647 fixes `renameAgentIdentities.spec`.
- #104 added `community-reconciliation`.
- #321 is the CI-failure ingest.

Retrieval Hint: "failure-set diff against the base", "brain-unit.yml smoke list", "collected but never executed".

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636




## Timeline

- 2026-09-30T19:34:11Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T19:34:12Z @neo-opus-grace added the `enhancement` label
- 2026-09-30T19:34:13Z @neo-opus-grace added the `ai` label
- 2026-09-30T19:34:13Z @neo-opus-grace added the `testing` label
- 2026-09-30T19:34:13Z @neo-opus-grace added the `build` label
- 2026-09-30T19:34:19Z @neo-opus-grace added parent issue #194
- 2026-09-30T19:53:09Z @neo-opus-grace cross-referenced by PR #651
- 2026-09-30T20:27:10Z @tobiu referenced in commit `56031c3` - "feat(ci): the unit run is bounded as a whole, and a pass that needed a retry counts as failing (#650)

The first run of the full config went silent after 4.7 minutes with about 480 tests left and was cancelled at the 30-minute job limit: a test that blocks its worker cannot time itself out. --global-timeout ends the run at 15 minutes with its report written and the interrupted tests named. The comparator now counts flaky as failing, matching the config's failOnFlakyTests in CI."
- 2026-09-30T20:58:56Z @neo-fable-clio cross-referenced by #652
- 2026-09-30T21:05:40Z @tobiu referenced in commit `fdc4019` - "feat(ci): the unit run is bounded just past its measured length, and a head run cut short is refused (#650)

On CI the last test ends about 6.5 minutes in (391 s head, 402 s base) and the run never exits: chroma-teardown never starts. The bound drops to 10 minutes. A test the base ran but the head never reached now refuses the comparison, so the bound can never trade coverage for time silently. Run on the second CI run's own reports: 0 introduced, 3 fixed, 58 pre-existing."
- 2026-09-30T23:04:07Z @neo-opus-grace referenced in commit `5cb46eb` - "chore(ci): merge dev into the unit failure diff after #654 landed, whose run-list line leaves with the list (#650)"
- 2026-09-30T23:04:07Z @neo-opus-grace referenced in commit `41172b0` - "fix(ci): the unit diff refuses a report with no recorded result, and every introduced test reaches the ledger with a symptom (#650)

A side whose listed tests were never reached recorded nothing and vouches for nothing, whatever statuses
it carries. Each introduced failure's block opens with an Error: line, the ledger's symptom: a typed
error, a timeout or a thrown value is prefixed, and a test with no recorded error says so."
- 2026-09-30T23:10:19Z @neo-opus-grace cross-referenced by #657

