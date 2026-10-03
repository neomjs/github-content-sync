---
id: 786
title: An interrupted file effect deadlocks the first run on a cold host
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-03T06:52:50Z'
updatedAt: '2026-10-03T09:10:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/786'
author: neo-fable
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 788 ADR 0041 §3 names how a host-file effect settles'
blocking:
  - '[ ] 475 The setup card recovers a run stuck behind an interrupted effect'
closedAt: '2026-10-03T08:36:20Z'
---
# An interrupted file effect deadlocks the first run on a cold host

## Context

Found while answering the review of neomjs/neo-agent-institution#464: the vessel's setup broker now reads the run's record from disk in every operation, so a `pending` receipt survives a rejected acknowledgement write. With the guard kept, the next question is how a run leaves that state. Measured on `dev@804356b` with the modules themselves (node, a temp layout, a recording command runner, no docker). Parent: neomjs/neo-agent-institution#351. The reading of ADR 0041 below was confirmed by its author by A2A on 2026-10-03 (`cba07660`).

## The Problem

A run with the preset and the plane credential consented, a `pending` `write-secrets` receipt (the handler ran, its acknowledgement did not land), the secret files observable on the host, and nothing serving the target's endpoint yet:

| Step | Result |
| :--- | :--- |
| `evaluateRecipe` | `write-secrets: reconcile-required` · `served-plane: unknown` |
| `settlePending` | unchanged |
| `performEffects` with `['write-secrets']`, `['write-env']`, `['compose-up']`, and unfiltered | no handler runs, no carrier, no docker call, `report` is never called |

The run cannot move. The plane comes up only through the effects that halt behind the row, and the row settles only once the plane serves. A renderer shows the row's reason ("a fresh matching observation settles it") and `re-check`, which loops; the CLI exits 1 on every pass with no line saying why nothing ran.

A run gets there when the host rejects the record write that acknowledges a file effect, or the process ends between the handler and that write.

Second sighting, same guard: `ai/scripts/setup/firstRun.mjs:339–345` — a record named by `--run-id` that is unreadable or malformed is reported and then replaced by a fresh record under the same id. A `pending` or `accepted` receipt in the damaged file is gone, and whatever the host does not show runs again. The vessel's broker refuses a bound record it cannot read.

## The Architectural Reality

- `ai/services/fleet/setupOrchestration.mjs:151–158` — `settlePending` computes `planeMatches` from the `served-plane` row and requires it for **every** `reconcile-required` effect: `matches = planeMatches && observed?.present === true && !observed.problem`.
- `setupOrchestration.mjs:122` — `performEffects` breaks out of its loop on a `reconcile-required` row and returns without a `report`.
- `ai/services/fleet/firstRunRecipe.mjs#evaluateEffect` — an effect with **no** receipt whose result is present reads `ok` ("observed; not performed by this run") from presence alone. The plane gate therefore adds no safety for a host-file effect that the recipe keeps anywhere else; the identity gate for those files is the `served-plane` row downstream.
- `ai/services/fleet/hostEffects.mjs#applyEffect` — the `pending` receipt carries `inputDigest` only; the digest of what the handler wrote exists only on the `accepted` receipt. The carrier's content is recomputable from its input (`renderEnvFile(input.entries)`); the secret files' is not (`composeCredentialEffects` mints the Fleet plane bearer on every call, and the observer never reads secret content).
- `ai/scripts/setup/firstRun.mjs#productionObservers.secretFiles` reads any owner-only file in the secrets directory as the result, while `write-secrets` writes its files one at a time (each atomic, the set not): an interruption after the first of three leaves a directory that reads `present`.
- `ai/services/fleet/setupRunRecord.mjs#describeRecordProblem` validates no receipt keys, so an added optional receipt field needs no schema version.
- `EFFECT_ORDER`: `write-secrets` → `write-env` → `compose-up`. The first two exist to bring the plane up.

## The Fix

1. **A host-file effect settles by its own fresh observation.** An effect ordered before `compose-up` matches when its result is present with no problem and, where its pending receipt carries the digest of the content the handler was about to write, the observed digest equals it. `compose-up` keeps the served plane as its matcher.
2. **The pending receipt carries the expectation.** `applyEffect` records the expected content digest on the `pending` receipt when the effect declares one — the carrier, from the handler's own rendering. Secrets declare none. A pending receipt without the field (a record written before this change) settles on presence and no problem.
3. **The secret files are a set.** The `secretFiles` observer reads the whole set the consented preset needs — `secretFileNames`, the one list the composer writes by — and names what is missing. A partly written set is not present: it never settles, and a run without a receipt completes it.
4. **A halt says so.** `performEffects` calls `report` when it stops behind a `reconcile-required` row: the row, and the reason it is unsettled in the observation's own words.
5. **The CLI refuses a damaged record.** `--run-id` over an unreadable or malformed record exits non-zero naming the file and writes nothing; an absent file stays a fresh run. `readSetupRecord`'s summary follows.
6. **ADR 0041 §3** gains one sentence saying so — in its own leaf, #788, merged ahead of this one (ADR 0005 §6.5). This leaf re-points §3's witness arm in `firstRun.spec.mjs` to the plane's own effect and adds the host-file half.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `settlePending({record, recordPath, host, evaluation})` (existing, signature unchanged) | ADR 0041 §2.6 and §3 as #788 amends it | A file effect settles by its own observation and its receipt's expected digest; `compose-up` by the served plane. | No expected digest on the receipt → presence and no problem. A present carrier with another digest → stays `reconcile-required`. | JSDoc | unit |
| `applyEffect`'s `pending` receipt (existing) | `hostEffects.mjs` | Carries `expectedDigest` when the effect declares one. | Absent for secrets, `compose-up` and older records. | JSDoc | unit |
| `productionObservers.secretFiles` (existing observer) | `credentialStep.mjs#secretFileNames` | `present` means the consented preset's whole set, owner-only; a partial set answers not present and names the missing files. | No consented preset → the files every plane needs. A file readable beyond its owner stays a problem. | JSDoc | unit |
| `performEffects` → `report` (existing) | `setupOrchestration.mjs` | One line when the run halts behind a `reconcile-required` row, carrying the reason it is unsettled. | — | JSDoc | unit |
| `firstRun.mjs --run-id` (existing) | `setupRunRecord.mjs#readSetupRecord` | A damaged record → refused, non-zero exit, the file named and untouched. | An absent file → a fresh run, as today. | usage text + JSDoc | unit |

## Decision Record impact

depends-on ADR 0041 §3 as amended by #788 (one sentence; the record's author reads it as a clarification). §2.6's sentence stands as written.

## Acceptance Criteria

- [x] AC-1 Cold host, `pending` `write-secrets`, the files present and owner-only, nothing serving: the settle pass accepts the receipt (`settledBy: 'observation'`) and `performEffects` proceeds to the next effect. Red first on `dev`.
- [x] AC-2 `pending` `write-env`: the carrier's digest equals the receipt's expected digest → accepted; another content → stays `reconcile-required` and the handler does not run; a pending receipt without the field → presence settles.
- [x] AC-3 `pending` `compose-up` with no plane serving, or another plane's identity or root → stays `reconcile-required` (ADR 0041 §3's witness, unchanged); with the target's plane → accepted.
- [x] AC-4 A halt behind a `reconcile-required` row reports one line naming the row and why it is unsettled, for a filtered and an unfiltered run.
- [x] AC-5 The CLI given `--run-id` over a malformed record exits non-zero naming the file and leaves it untouched; an absent file is a fresh run.
- [x] AC-6 #788's sentence is on `dev` before this leaf merges, and §3's witness arm (`firstRun.spec.mjs`) holds for the plane's own effect with the host-file half beside it.
- [x] AC-7 An interrupted `write-secrets` that left part of its set (one of the hosted preset's three files) stays `reconcile-required`, the halt names the missing files and nothing runs over the partial set; the complete set settles. Found at review (Grace, Sophie).
- ~~[post-merge] the vessel-side witness on the Institution pin~~ — moved to neomjs/neo-agent-institution#475 (AC-1, AC-2), which this leaf blocks.

## Out of Scope

- The product exit for a row that cannot settle at all (a `compose-up` pending on a host whose docker is gone; a carrier that holds other content): "start a fresh run" from the card — neomjs/neo-agent-institution#475.
- Resolving a file effect whose result is observably absent to `failed`: a definite non-application, but a new settle outcome that would change §2.6's sentence.
- A receipt-less carrier reading `ok` whatever it contains: `evaluateEffect`'s presence rule, on every path.
- #784 (the served-plane observer) and #782 (the verify effect).

## Avoided Traps

- **Recomputing the carrier's content inside `settlePending`** from the layout and the target: a second derivation of the carrier beside `performEffects`' inputs, and a signature change both renderers would have to follow in lockstep. The receipt belongs to the interrupted application; it carries the expectation.
- **A fresh run id as the only exit:** the same files read `ok` from presence there, so it launders the receipt instead of fixing the rule.
- **Relaxing the plane gate for `compose-up`:** that is the effect whose result is the plane; ADR 0041 §3's witness is about exactly that row.

## Related

neomjs/neo-agent-institution#351 (parent) · #788 (the ADR sentence; blocks this) · #750 (the orchestration's extraction) · #784 · #782 · neomjs/neo-agent-institution#464 · ADR 0041 (`learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md`)

Structure map: owning folder `ai/services/fleet` (siblings `setupOrchestration.mjs`, `hostEffects.mjs`); no new file.

Live latest-open sweep: the latest 20 open issues at 2026-10-03T06:52Z (newest #784), plus exact searches for `settlePending`, `reconcile-required` and `cold host` over open and closed issues: no equivalent (#784 and #782 are neighbours on the observer side).
A2A in-flight sweep at 06:52Z, the latest 30 messages of every read-state: no claim on the settle pass; the epic owner handed it over (`cba07660`).
MC sweep: "first-run setup stuck: an interrupted effect stays reconcile-required and nothing after it runs on a host where no plane is serving yet", 5 results, no prior decision found.
Own-assignment sweep: 7 open, none on the setup surface.

Origin Session ID: 25618ee4-58d2-46dd-ae26-9dcf2854b14a
Retrieval Hint: "settlePending planeMatches cold host pending write-secrets reconcile-required deadlock expectedDigest report"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 25618ee4-58d2-46dd-ae26-9dcf2854b14a





## Timeline

- 2026-10-03T06:52:51Z @neo-fable assigned to @neo-fable
- 2026-10-03T06:52:52Z @neo-fable added the `bug` label
- 2026-10-03T06:52:52Z @neo-fable added the `ai` label
- 2026-10-03T06:52:52Z @neo-fable added the `architecture` label
- 2026-10-03T06:52:53Z @neo-fable added the `agent-os` label
- 2026-10-03T06:53:07Z @neo-fable added parent issue #351
- 2026-10-03T06:59:37Z @neo-fable cross-referenced by #788
- 2026-10-03T07:00:01Z @neo-fable marked this issue as being blocked by #788
- 2026-10-03T07:02:40Z @neo-fable cross-referenced by PR #789
- 2026-10-03T07:04:31Z @neo-fable cross-referenced by #475
- 2026-10-03T07:04:44Z @neo-fable marked this issue as blocking #475
- 2026-10-03T07:05:49Z @neo-fable cross-referenced by PR #790
- 2026-10-03T07:16:09Z @neo-fable cross-referenced by #351
- 2026-10-03T08:19:14Z @neo-fable referenced in commit `306e366` - "chore(merge): bring origin/dev into the branch (#786)"
- 2026-10-03T08:19:14Z @neo-fable referenced in commit `7c1928c` - "fix(fleet): the secret-file observer reads the consented preset's whole set, so a partly written set never settles (#786)

write-secrets writes its files one at a time, and the observer read any owner-only file in the directory as the result. With the settle rule matching a host-file effect by its own observation, an interruption after the first of three files settled as done and the run went on without a secret. The observer now reads the set the consented preset needs, from the one list the composer writes by, and names what is missing; the halt line repeats the observation's reason."
- 2026-10-03T08:36:20Z @tobiu referenced in commit `36b17f4` - "feat(fleet): an interrupted host-file effect settles by its own observation, so a cold host's first run is never deadlocked behind it (#786) (#790)

* feat(fleet): an interrupted host-file effect settles by its own observation, so a cold host's first run is never deadlocked behind it (#786)

settlePending gated every interrupted effect on the served plane. write-secrets and write-env exist to bring that plane up, so a pending receipt on either waited for a plane only the effects behind it could start, and performEffects halted without a word. A host-file effect is now matched by the file itself: present, no problem, and the content its pending receipt expected, which applyEffect records for an effect that declares it (the carrier). compose-up keeps the served plane as its matcher. A halt behind an unsettled row is reported with what settles it.

The CLI refuses a --run-id whose record is unreadable or malformed instead of replacing it with a fresh one: a receipt it cannot read may guard an effect that ran.

* fix(fleet): the secret-file observer reads the consented preset's whole set, so a partly written set never settles (#786)

write-secrets writes its files one at a time, and the observer read any owner-only file in the directory as the result. With the settle rule matching a host-file effect by its own observation, an interruption after the first of three files settled as done and the run went on without a secret. The observer now reads the set the consented preset needs, from the one list the composer writes by, and names what is missing; the halt line repeats the observation's reason."
- 2026-10-03T08:36:20Z @tobiu closed this issue
- 2026-10-03T12:25:27Z @neo-fable-clio cross-referenced by #810

