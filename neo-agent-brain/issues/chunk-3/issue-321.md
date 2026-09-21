---
id: 321
title: 'The defect ledger has no machine producer, so CI''s failures reach it only if an agent happens to notice'
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-09-04T23:31:33Z'
updatedAt: '2026-09-05T13:41:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/321'
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
closedAt: '2026-09-05T13:41:07Z'
---
# The defect ledger has no machine producer, so CI's failures reach it only if an agent happens to notice

## Context

@tobiu, 2026-09-04 ~23:28Z, on my reporting a third live `components` intermittent as *"unticketed"*: **`friction->gold`**.

The word was the finding. Here is what "unticketed" cost in one evening on one suite.

Three `components` failures on one Dependabot PR, which I attributed as one recurring defect and used to recommend splitting a `@playwright/test` bump out of the update group:

| run | failing spec | actually owned by |
|---|---|---|
| original | `DockRailLazyModule.spec.mjs:152` — "one construction — 2 recorded" | #18331 → neomjs/neo#18275 |
| post-rebase | `SpawnAnimation.spec.mjs:183` — menu remount | neomjs/neo#18290 |
| attempt 2 | `tab/OverflowAction.spec.mjs:134` — **30.0s timeout** | nothing — unticketed |

Three runs, three unrelated intermittents, one wrong recommendation to the operator. @neo-opus-ada independently reached the same methodological finding from the other side and stated it exactly: **a red `components` job is not a measurement of any one defect; attribute per failing spec line, never per job colour.** Between us that error took four forms in one evening — workflow-conclusion for job-conclusion, conclusion without status, job colour for spec line, and mine again after being told.

**Live latest-open sweep:** checked the latest 20 open issues in this repository at 2026-09-04T23:29:53Z; no equivalent. Cross-repo sweep on `flake` / `intermittent` / `CI failure` over `neo-agent-skills`, this repository, and `neomjs/neo`, open **and** closed: no ticket proposes a producer for the defect ledger. A2A claim sweep over the current window: no overlapping claim. Own-assignment sweep: my open Brain items touching this surface are #304 (analysed below) and #317; neither covers it.

## The Problem

**The ledger is finished and has one producer: a human typing.**

`ai/services/memory-core/helpers/defectObservationFold.mjs` computes a deterministic fingerprint per observation, folds sightings into one standing record, and runs a `red → recovered` state machine. `defectObservationTriggers.mjs` holds the promotion predicates. `ai/scripts/diagnostics/defectObservations.mjs --digest` broadcasts newly-qualifying rows, de-duplicated against prior digests. `ai/daemons/orchestrator/taskDefinitions.mjs:443` drives it on a periodic tick.

Every one of those consumes A2A `defect-note:` lines that an agent chose to write.

**CI is the highest-volume, most reliable defect signal this organism produces, and it feeds that ledger nothing.** Nothing under `ai/` harvests a workflow run; the only `flake`/`intermittent` matches are the wake daemon and a dialog rig, both unrelated.

**What that costs, measured rather than asserted.** `neomjs/neo` currently carries four *separately hand-filed* flake tickets — #17796 (*"the unit suite loses one arbitrary test per run and names no cause"*), #16754, #18215, #18331 — plus #18275 and #18290. Each was noticed by a different maintainer, diagnosed from scratch, and written up by hand. #18215's body records *"roughly one run in eight"*, a rate somebody measured manually because nothing counts. This repository's own #17 (closed) is the same shape one suite over: *"a quarter of the AgentOS e2e layer is red on dev and no pipeline reports it."*

So the failure mode is not that flakes go unnoticed. It is that **each one is discovered privately, costs a full diagnosis, and leaves no shared identity** — which is precisely why three of them could hide behind one job colour tonight.

## The Architectural Reality

- `defectObservationFold.mjs:45` — `defectNoteFingerprint`, deterministic identity computed from the note text alone, with context-keyed volatility normalization so a count or status code inside a symptom does not fork the identity. **A `<spec path>:<line>` key is exactly the shape this already handles**, and it is the same attribution unit tonight proved correct.
- `defectObservationFold.mjs:101` — the `red → recovered` machine. A flake that stops firing is a *recovery note*, not a deletion, so a rate survives its own fix.
- `ai/scripts/diagnostics/defectObservations.mjs` + `taskDefinitions.mjs:443` — digest and periodic tick, both already wired. Nothing here needs building.
- `ai/services/ingestion/` — the owning folder, and the sibling precedent is close: `IssueIngestor.mjs` and `PrOutcomeReward.mjs` already turn GitHub state into graph signal. A CI-failure ingestor is the same layer and the same shape, not a new class of thing.
- **#304 is adjacent and inverts the way it first reads.** It measures the *human* producer failing silently twice: 3 of 9 well-formed notes never entered because the intake gate is `subject.startsWith('defect-note:')` and the prefix sat at index 43 or inside a bracketed tag, and only 1 of 6 admitted notes parsed. Every one of those is a **subject-line composition** failure. A machine producer emits the canonical form at index 0 with the exact `<surface> broke <symptom>` grammar, so it does not inherit either fault. #304 therefore **does not block this**, and this must not be used as a reason to leave #304 unfixed — humans keep filing notes and their path stays lossy until it is.

## The Fix

An ingestor that turns a completed CI run's failing spec lines into canonical `defect-note:` observations.

1. **`ai/services/ingestion/CiFailureIngestor.mjs`** — pure mapping from a run's failed-job logs to `{fingerprintSource, surface, symptom}` triples, one per failing spec line. Pure and unit-testable; no GitHub client inside it, matching `PrOutcomeReward`'s split.
2. **The surface is the spec line, never the job.** `defect-note: components tab/OverflowAction.spec.mjs:134 broke <assertion or timeout>`. Three failing specs in one job produce three observations, which is the whole point.
3. **Recovery is emitted, not inferred — and proven, not assumed.** A record that folded `red` and has been quiet for the recovery window emits the `[recovered]` arm when a newer green run of its own job names the record's test with a pass mark in its log; a job's colour says nothing about one test, and a reporter that names no passing tests can prove nothing, so neither recovers anything.
4. **Existing consumers, one new coordinate.** No new store, no new digest, no new rule an agent loads. The fold records the distinct threads a fingerprint was sighted under and the promotion trigger reads two threads from one reporter as independent — a machine producer is one identity, so the run is its independence. The digest is unchanged; the orchestrator gains the `ci-failure-ingest` task and its config leaves.
5. **Admission is per observation, completeness is stated.** A failed job's run-and-job thread is read in full and only the notes it lacks are sent, so an interrupted write resumes at the note it lost. The mailbox is read page by page up to a cap, a run listing that exhausted its page bound leaves its unread remainder as a continuation slice drained before the next window, and a tick that hit either bound files sightings but certifies no recovery. Runs still running when their window was read are carried as pending and finished by id.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `CiFailureIngestor.mjs` (new) | this ticket; `PrOutcomeReward.mjs` as the pure-mapping precedent | failing spec lines → one canonical note per line | a log it cannot parse yields **no** note rather than a malformed one — a wrong fingerprint is worse than a missing row | module JSDoc | three distinct specs in one job produce three fingerprints |
| `defect-note:` grammar | `defectObservationFold.mjs:69` `parseDefectNote` | emitted at index 0, exact `<surface> broke <symptom>` | — | unchanged | 100% parse rate against the shipped fold, versus 1/6 on the human path (#304) |
| the fold (`foldDefectObservations`) | already shipped | records `threads` — the distinct `partOfThread` values a fingerprint was sighted under | a hand-filed note carries no thread and records none | helper JSDoc | two runs of one test → one record, two threads |
| the promotion trigger (`independentSecondOccurrence`) | already shipped | two distinct threads from one reporter count as independent | the reporter rule stays for thread-less notes | helper JSDoc | a CI-only record with two runs qualifies; a human echoing their own note does not |
| the digest | already shipped | **unchanged** | — | unchanged | digest output is identical in shape, only better populated |
| the orchestrator | `taskDefinitions.mjs` | gains `ci-failure-ingest` (container plane, lightweight, `intervals.ciFailureIngestMs`) and the `orchestrator.ciFailureIngest.*` leaves | the task fails supervised; nothing else depends on it | leaf JSDoc + parity snapshot | the task runs the script exactly where the digest's task runs its own |
| admission (`ingestCiFailures.mjs`) | this ticket | per observation: a run-and-job thread is read in full and only the notes it lacks are sent | an interrupted write leaves the run unreceipted and resumes at the missing note | module JSDoc | the two-note interrupted-write control |
| recovery evidence | this ticket | a newer green run of the record's own job, its `Run … tests` step run, **and** that job's log naming the record's test with a pass mark (`parsePlaywrightPasses`) | no proof, no recovery — the record stays red; a reporter that names no passing tests proves nothing | module JSDoc | the test-not-observed-passing, no-per-test-evidence and evidence-log-unreadable controls |
| completeness | this ticket | the mailbox is paged to a cap; a run listing that exhausted its page bound leaves a continuation slice, drained before the next window; an incomplete scan or an open continuation certifies no recovery; running runs are pending and finished by id; an epilogue shorter than its count files nothing | reported in the tick summary and the receipt, never assumed | module JSDoc | the page-boundary, continuation, pending-run and epilogue-truncation controls |

## Decision Record impact

`none` — a new producer for an existing store; no ADR authority is amended, superseded or challenged.

## Acceptance Criteria

- [ ] AC-1 — One failing job containing N distinct failing spec lines yields **N** observations with **N** distinct fingerprints. Tonight's three-spec run is the fixture.
- [ ] AC-2 — The same spec failing across two runs folds to **one** record with a sighting count, not two records.
- [ ] AC-3 — Every emitted note passes the shipped intake gate and `parseDefectNote` — asserted against the real `defectObservationFold.mjs`, not a copy. This is the arm that would have caught #304's class at authoring time.
- [ ] AC-4 — A record that folded `red`, has been quiet for the recovery window, and whose own job ran green in a newer run **whose log names the record's test with a pass mark** emits the `[recovered]` arm and moves the record's state. A green job whose log names the spec file but not this test (renamed, removed, or never run there), a reporter that names no passing tests (the `unit` job's `github` reporter today — those records stay red until a human recovers them or the reporter names passes), an unreadable evidence log, a skipped suite, an older run, another job, or a fresh sighting recovers nothing.
- [ ] AC-5 — **Non-vacuity:** an unparsable log, a log truncated before its epilogue, or an epilogue shorter than its own `N failed` count yields **zero** notes, and a test proves each — never a note whose fingerprint is derived from the failure to read the log. A row that identifies "the parser broke" as a defect in the code under test is worse than silence. Failure blocks are matched to epilogue rows by full test identity (project, location, title path), so two tests declared on one line keep their own symptoms.
- [ ] AC-6 — Accretion: net-new is one ingestor plus its wiring (the pure mapper, the REST reads, the tick, the orchestrator task and its leaves). The fold and the promotion trigger gain the thread coordinate above and nothing else; the digest is untouched. The ticket states which artisanal step it retires — hand-filing a flake to *discover* it, not to fix it.
- [ ] AC-7 — **Retirement trigger:** if the runner gains first-class flake reporting that the fold can consume directly, this ingestor retires with it — stated in the module docblock of `ingestCiFailures.mjs`.
- [ ] AC-8 — Post-merge: the digest names at least one observation nobody filed by hand. `tab/OverflowAction.spec.mjs:134` is the live candidate — currently a defect-note I wrote only because the operator asked why a PR was red.
- [ ] AC-9 — **Lossless retry:** admission is per observation. A write interrupted after its first note leaves the run unreceipted, and the retry sends exactly the notes the run-and-job thread lacks — the two-note interrupted-write control proves it. A note already on the thread, whichever orchestrator filed it, is never sent again.
- [ ] AC-10 — **Completeness is stated, never assumed:** the mailbox is read page by page up to the configured cap and a tick that hit the cap files sightings but certifies no recovery; a run still running when its window was read is carried as pending in the receipt and finished by id on a later tick, whatever window it was created in; a run listing that exhausted its page bound leaves the unread remainder — the window's start up to the oldest run it read — as a continuation slice in the receipt, drained oldest-first before the next window, and a tick with an open continuation certifies no recovery. The page-boundary, pending-run and continuation controls prove all three.

## Out of Scope

- **Fixing any flake.** #18275, #18290 and the OverflowAction timeout are their own lanes. This measures; it does not repair.
- **Auto-filing tickets from observations.** Promotion already has stated triggers and stays a judgement call. A ledger that opens issues by itself is the machinery this repository has declined before.
- **Retrying or quarantining flaky specs.** Suppression hides the rate this exists to expose.
- **#304's human-path fix.** Complementary and explicitly not superseded — see The Architectural Reality.

## Avoided Traps

- **Building a flake ledger.** My first shape, and it was wrong: the ledger exists, complete with fingerprinting, a state machine, promotion triggers, a digest and an orchestrator tick. Only the producer was missing. Adding a second store beside a finished one is the accretion this repository's own defense names.
- **Treating #304 as a blocker.** It reads like one until you look at *which* stage fails: every #304 loss is a human composing a subject line. A machine producer is the population that cannot make those mistakes.
- **Keying on the job.** The entire origin of this ticket is that a job colour stood in for three unrelated defects. Fingerprinting anything coarser than the spec line reproduces the defect in the tool built to fix it.
- **Emitting a note when the log cannot be read.** The tempting failure-open. It would file the harvester's own breakage as a defect in the code under test, and the fold would faithfully keep it forever.

## Related

- neomjs/neo#18331 (closed on evidence by @neo-opus-ada) · neomjs/neo#18275 · neomjs/neo#18290 · neomjs/neo#18215 · neomjs/neo#17796 · neomjs/neo#16754 — six hand-filed instances of the class
- #304 — the human producer's silent losses; complementary
- #17 (closed) — the same shape in the AgentOS e2e layer

unowned-rationale: parked deliberately rather than dropped. I authored it from tonight's friction and I am the wrong owner — @tobiu has just asked me to stop serializing work behind myself, and I already hold #317 in review. It is self-contained, needs no dock context, and the fixture it wants is in this ticket. Claimable by any seat.

Origin Session ID: `b983b2a8-18bc-4dd0-a63f-23380dc3a49d`

Retrieval Hint: `query_raw_memories("defect ledger has no machine producer, CI failures per spec line, red job is a bucket not a measurement")`



## Timeline

- 2026-09-04T23:31:34Z @neo-opus-grace added the `enhancement` label
- 2026-09-04T23:31:34Z @neo-opus-grace added the `ai` label
- 2026-09-04T23:31:34Z @neo-opus-grace added the `testing` label
- 2026-09-04T23:31:35Z @neo-opus-grace added the `agent-os` label
- 2026-09-05T00:40:49Z @neo-opus-grace cross-referenced by #18348
- 2026-09-05T01:50:01Z @neo-fable assigned to @neo-fable
- 2026-09-05T02:26:36Z @neo-fable cross-referenced by PR #327
- 2026-09-05T12:03:20Z @neo-fable referenced in commit `cda0519` - "fix(brain): the CI producer admits per observation, proves recovery at the green head, reads the mailbox to its end and matches blocks by full identity (#321)

Round 1 of the review on the defect ledger's machine producer. Four executed
controls showed the first head could lose an observation, recover a record
without observing its test, certify a partial mailbox read, and hand two tests
declared on one line the same error.

Admission moves from the job to the observation: a failed job's run-and-job
thread is read in full and only the notes whose fingerprints it lacks are sent,
so a write interrupted after its first note leaves the run unreceipted and the
retry sends the second. Recovery needs applicable evidence: the candidate green
run of the record's own job must carry a head that contains the record's spec
(one Contents read), or the record stays red — a job's colour on a branch that
never had the test says nothing about it. Completeness is stated: the mailbox is
paged to the configured cap and a tick that hit the cap files sightings but
certifies no recovery; runs still running when their window was read are carried
as pending in the receipt (version 2) and finished by id later, whatever window
they were created in. Failure blocks are matched to epilogue rows by project,
location and title path — a lone block still serves a reshaped row — and an
epilogue shorter than its own count files nothing.

The Actions client lists runs across every status, reads one run by id and
answers a Contents existence read (404 as absent, other failures thrown). The
ticket's ledger and ACs now state these rules (AC-4/5/6/7 amended, AC-9/10
added); the module docblock carries the retirement trigger AC-7 names."
- 2026-09-05T13:11:53Z @neo-fable referenced in commit `18542dc` - "fix(brain): recovery needs the test named passing in the green job's log, and a bounded run listing leaves a continuation slice instead of losing its oldest run (#321)

Round 2 of the review. Two controls at the round-1 head still fell: a green
job whose head held the spec file but not the affected test recovered the
record anyway, and a window with one run more than the listing's page bound
lost that run for good — the receipt advanced past it with nothing pending.

Recovery evidence is now the test, not the file: the evidence job's log is
read (once per job per tick) and the record's surface must appear with a pass
mark — the list reporter's `✓ N [project] › path:line:col › titles` line,
whatever project, line or retry — or nothing is filed. A log that names the
spec but not this test, a reporter that names no passing tests (the unit
job's github reporter today), an unreadable log: each is a stated reason in
the tick summary and the record stays red. The Contents read and specPathOf
leave with the premise they served.

The Actions client's listing says whether it reached the window's start and
takes an upper bound, so an exhausted page bound hands back the newest runs
and the tick keeps the unread remainder — the window's start up to the
oldest run it read — as a continuation slice in the receipt, drained oldest
first before the next window; a slice too large to finish narrows to what it
did not read. An open continuation certifies no recovery, like an incomplete
mailbox scan. The ticket's AC-4 and AC-10 carry both rules."
- 2026-09-05T13:24:51Z @neo-fable cross-referenced by #18361
- 2026-09-05T13:28:39Z @neo-fable cross-referenced by PR #18362
- 2026-09-05T13:41:07Z @tobiu referenced in commit `26f61e9` - "Merge pull request #327 from neomjs/agent/321-ci-failure-ingestor

feat(brain): CI failures reach the defect ledger, one note per failing test with the run as its independence (#321)"
- 2026-09-05T13:41:07Z @tobiu closed this issue

