---
id: 448
title: Projection and temporal-summary failures log only a bare exit code
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-24T11:19:54Z'
updatedAt: '2026-09-24T13:03:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/448'
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
closedAt: '2026-09-24T13:03:12Z'
---
# Projection and temporal-summary failures log only a bare exit code

## Context

#442 AC-3 routes a recurrence of the core corpus projection failure to its own defect ticket. The failure recurred on every cycle: 3/3 (2026-09-23 12:14Z and 18:41Z, 2026-09-24 10:38Z). Its cause was plane config, and #442 records that and the fix. This ticket covers why the cause took 22 hours and one wrong diagnosis to find: for every cycle, the orchestrator log held only `[ERROR] [ProcessSupervisor] core corpus projection exited with code 1.`

## The Problem

Observed on the local plane (images `b99ea11`). The failing cycle's reason, `Error: GitMirror failed to read a revision file` at `materializeCoreCorpusRevision (coreCorpusProjection.mjs:218)`, exists only in `logs/mc-server-<date>.log`. Git's own reason, `fatal: path '_index.json' does not exist in '<rev>'`, is printed nowhere: GitMirror attaches it to the error, and no logger prints it. A reader of the orchestrator log cannot tell a configuration fault from a transient one. The first diagnosis on #442 read the wrong mirror and called the failure possibly transient.

## The Architectural Reality

- `ProcessSupervisorService#writeChildStderr` re-logs every child stderr line at the child's severity. An unprefixed line takes the ERROR fail-safe (`ProcessSupervisorService.mjs:433–490`). Stdout is the child's JSON outcome channel (`captureStdoutJson`).
- Two orchestrator task children end in a top-level catch that logs through `ai/mcp/server/memory-core/logger.mjs` and exits 1: `ai/scripts/maintenance/projectCoreCorpus.mjs` and `ai/scripts/maintenance/aggregate-temporal-summary.mjs`. That logger runs `fileSink: true, filePrefix: 'mc-server', stderrMode: 'debug'`. Once `aiConfig.isReady`, a non-debug line goes to the file sink only (`ai/mcp/server/shared/logger.mjs`, `writeFile` / `writeStderr`). Lines logged before ready go to stderr. That is why the child's `[SessionService]` boot lines reach the orchestrator log and its fatal does not.
- `ai/scripts/maintenance/ingestCiFailures.mjs` is the working sibling. Its top-level catch writes `console.error(\`ingestCiFailures: ${error?.message || error}\`)`, and on the same plane its "Could not authenticate with GitHub" line reached the orchestrator log.
- `createGitMirrorError` attaches git's redacted stderr as `error.stderr` (`ai/services/knowledge-base/helpers/gitMirror.mjs:57–70`).

## The Fix

In both children's failure path, keep the file-sink line and also write one stderr line carrying the task label, `error.code`, the message and `error.stderr` when present (already redacted). For `projectCoreCorpus.mjs`, the line goes through `runProjectCoreCorpus`'s existing `output` / `exit` seams, which makes it unit-testable. `aggregate-temporal-summary.mjs` keeps its direct `main()` and gains an exported `reportAggregationFailure(error, {output, exit})`, which its entry guard calls. No logger, supervisor or config change.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| orchestrator log line for a failed projection cycle | `projectCoreCorpus.mjs` failure path → `ProcessSupervisorService#writeChildStderr` | one ERROR line naming the code, the message and git's stderr | the message alone when the error carries no code or stderr | unit arm with a GitMirror-shaped error through the output / exit seams |
| orchestrator log line for a failed temporal-summary cycle | `aggregate-temporal-summary.mjs` failure path | same | same | unit arm |
| the `mc-server-<date>.log` line | `ai/mcp/server/memory-core/logger.mjs` | unchanged | — | the arms assert it is still written |

Decision Record impact: none.

## Acceptance Criteria

- [ ] **AC-1** A projection cycle that throws a GitMirror error writes one stderr line carrying `KB_GITMIRROR_FILE_READ_FAILED`, the message and git's stderr, and exits 1. The file-sink line is still written. Unit witness through `runProjectCoreCorpus`'s seams, red on the current catch.
- [ ] **AC-2** `aggregate-temporal-summary.mjs` reports its fatal the same way. Unit witness, red on the current catch.

Evidence ceiling L2: the supervisor half (stderr → orchestrator log at ERROR) is covered by its own specs, and no deployed-plane AC is needed.

## Out of Scope

- The plane's corpus-source pin (retired; recorded on #442).
- The memory-core logger's stderr mode and the supervisor.
- Other children: a sweep of `ai/scripts` for scripts that import the memory-core logger and exit 1 found these two plus one migration (`priorityBackfill.mjs`), which is not an orchestrator task.

## Avoided Traps

- **Switching the memory-core logger to stderr in children.** Every child INFO line would then land in the orchestrator log, which is the noise the supervisor's severity mapping was written to contain.
- **Having the supervisor read a child's log file after a non-zero exit.** That couples the supervisor to one child's sink.
- **Injecting the temporal-summary service as a seam to test the failure path.** This was tried and measured. `lint-script-plane` walks the entry's static call chain, and with the service passed in it could no longer follow `runCycle`. It then reported the entry's known authority conflict as stale (`aggregate-temporal-summary.mjs::temporal-summary::authority-conflict-in-plane`), although the conflict is unchanged at runtime. Only the failure report is extracted, so the chain stays visible.

## Related

#442 (receipt owner; its AC-3 routes here) · #253 · #411 · neomjs/neo#17416

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-24T11:16Z; none equivalent. Org exact search (`"exited with code 1"`, `stderrMode`, `"mc-server log"`, `projectCoreCorpus`, `aggregate-temporal-summary`): only the closed neighbours neomjs/neo#12812 (progress logs) and neomjs/neo#13755 (deferred runs logged as completed), neither this defect. A2A in-flight sweep (30 most recent, to 11:18Z): no claim on child failure logging. MC sweep: `query_raw_memories` on the symptom (6 results); a 2026-06-08 note records that children then logged to stdout, and no prior decision covers this. Own-assignment sweep: #442 is the only same-surface ticket, and it routes here. Structure map: N/A — no new file; `ai/scripts/maintenance`, sibling `ingestCiFailures.mjs`.

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("core corpus projection exited with code 1 no reason in orchestrator log mc-server file sink stderrMode debug")`

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿



## Timeline

- 2026-09-24T11:19:55Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-24T11:19:56Z @neo-opus-vega added the `bug` label
- 2026-09-24T11:19:56Z @neo-opus-vega added the `ai` label
- 2026-09-24T11:19:56Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T11:46:56Z @neo-opus-vega cross-referenced by #449
- 2026-09-24T11:48:06Z @neo-opus-vega cross-referenced by #442
- 2026-09-24T11:54:41Z @neo-opus-vega cross-referenced by PR #450
- 2026-09-24T12:05:15Z @neo-opus-vega cross-referenced by PR #452
- 2026-09-24T12:27:15Z @neo-opus-vega referenced in commit `a67c375` - "fix(orchestrator): the temporal-summary failure line carries the command stderr too (#448)

Resolves review R1 on #452. reportAggregationFailure kept the code and the
message but dropped error.stderr, so the parity the ticket and the PR body
claim for both children held only for the projection. The line now carries
the command's stderr collapsed to one line, exactly like the projection
child's. The existing arm stays as the no-stderr control; a new arm with a
GitMirror-shaped two-line stderr is red with the stderr part removed."
- 2026-09-24T12:33:01Z @neo-opus-vega cross-referenced by PR #454
- 2026-09-24T13:03:12Z @tobiu referenced in commit `4712fb7` - "Merge pull request #452 from neomjs/vega/448-child-fatal-line

fix(orchestrator): a failed projection or temporal-summary child names its reason in the orchestrator log (#448)"
- 2026-09-24T13:03:12Z @tobiu closed this issue

