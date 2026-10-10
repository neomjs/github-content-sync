---
id: 655
title: 'The seat card reads an unanswered launch proof as waiting, not refused'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-10-10T13:56:22Z'
updatedAt: '2026-10-10T18:06:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/655'
author: neo-fable-clio
commentsCount: 0
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-10T18:06:25Z'
---
# The seat card reads an unanswered launch proof as waiting, not refused

## Context

Brain #964 (Sophie, in implementation 2026-10-10) changes what the Fleet's launch-admission audit says when a Claude Desktop seat's Neo MCP row redeems its grant while the plane is slow: an unanswered credential proof answers a new refusal code, `proof-unavailable`, with the probe's closed-vocabulary reason beside it, and the launcher retries under a ≤ 60 s budget instead of exiting. Today the seat card's admission words (#590 / PR #597, `apps/agentos/util/SeatLaunchAdmission.mjs`) know two refusal codes only and read the audit entry's `reason` as the credential's kind. This leaf is #964's AC-3 residual on the Institution side (Sophie's hand-off, 13:54Z): the card must say what the seat is doing while its tool connection waits, and when a restart is the next step.

Design authority: #477 (every cockpit surface names its state with a reason and a next step); Brain #964's contract (`LAUNCH_ADMISSION_REFUSALS.PROOF_UNAVAILABLE`, the audit entry's probe reason).

## The Problem

`SeatLaunchAdmission.guidance` (`util/SeatLaunchAdmission.mjs:116–137` at dev `af1b85d6`) keeps only the latest refused entry per server whose `code` is in `CREDENTIAL_REFUSAL` (`credential-missing`, `credential-unproven`), words it as `New <server> connection refused` with `<credential> is missing | could not be verified`, and reads `entry.reason` as `seat-pat` / `plane-bearer`. A `proof-unavailable` entry falls through: no `latestFailure`, the plain server line, nothing said — while the seat's Memory Core and Knowledge Base rows are retrying, and still nothing once the launcher's budget ends and Desktop reads `Server disconnected`. The operator saw exactly that silence on the first fleet start.

## The Architectural Reality

- The issuer owns the generation and its audit; the util only words a validated observation (its own JSDoc). `statusOf().recent` carries `{at, server, outcome, code, reason}`; #964's wire decision (Sophie, 2026-10-10 14:00Z): no new field — for `proof-unavailable` the entry's `reason` IS the producer's bounded diagnostic (strictly allowlisted wording), for the two credential codes it stays the credential label (`seat-pat` / `plane-bearer`); the launcher classifies retries by code alone and never parses `reason`; the launcher's full startup budget (60 s) is fixed protocol, not a config leaf.
- `CARD-CONTRACT.md` records the card's status-line words; the visual gate holds a golden for the refusal line from #597. (Correction 2026-10-10: no visual arm covers the admission line — #597's refusal words are unit-asserted on the card, not captured; hence AC-3's restatement.)
- The audit entry's `at` is the issuer's answer time; a launcher's 60 s budget runs from its own start, and Desktop may start a later launcher under the same generation, each with its own budget (Sophie's R1 RA-2 and R2 probes, 2026-10-10). The card can observe a run's duration from its first entry; it cannot observe whether a launcher is retrying now.
- The card cannot see the launcher's retry budget; it can see the entry's `at`.

## The Fix

1. `SeatLaunchAdmission`: a third worded code. For each enabled server's current run of `proof-unavailable` entries (an admitted or any other entry ends a run): `text: New <server> connection waiting`, `title: <Credential> proof for <server> went unanswered (<probe reason>). The seat's launcher retries an unanswered proof for about a minute after it starts. Existing tools may still work.` while the run's FIRST entry is younger than one launcher's budget; past it, `text: New <server> connection still unanswered`, `title: … went unanswered (<probe reason>) for longer than one launcher's retry budget, about a minute. Existing tools may still work. Restart this seat to try the connection again.` with `restart: true` only when the lifecycle controls allow one. The words observe the run's duration and advise; they never assert what a launcher is doing (Desktop may start a later launcher under the same generation, with its own budget) nor Desktop's connection state. The waiting line names the budget's end as `until`, so the card can re-render then without a record change. The two credential codes keep their words and their `reason` keeps meaning the credential's kind; for `proof-unavailable` the probe reason is the entry's own `reason`, printed only when the contract's `isLaunchAdmissionProofReason` (Brain #973) admits it (else omitted). The "Existing tools may still work." sentence stays in every title: an already-running tool is never reported as the one that waits or was refused. The 60 s is the launcher's fixed protocol budget, written once as a documented constant beside the words. (Disposition 2026-10-10, Sophie's R1 RA-2 + R2: the first wording dated the budget from the latest entry's `at` and ended the title with "It gave up; restart the seat." — an answer's time is not a launcher's start, and no launcher state is observable from the audit; restated above.)
2. `CARD-CONTRACT.md`: the waiting line beside the refused line.
3. The rendered card line asserted in the card container spec (`control-status` text and title). No visual arm covers the admission line today, so no golden is added for one line of text; `CARD-CONTRACT.md` names the line. (Restated 2026-10-10 16:4xZ, before the PR closes the ticket: the first wording assumed a golden from #597 that does not exist.)

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `SeatLaunchAdmission.guidance` (`util/SeatLaunchAdmission.mjs:38–138`) | #590, this ticket | words `proof-unavailable` as waiting, with the probe reason and a time-bound restart hint | an unknown code still falls through to the server line | JSDoc + CARD-CONTRACT.md | unit (AC-1, AC-2) |
| `recent[].reason` by code (Brain #964) | Brain #964 | `proof-unavailable` → the producer's allowlisted diagnostic, read into the title when it is one of the allowlist; credential codes → the credential label as today | omitted from the words | #964's contract row | unit (AC-1) |
| the rendered card line | this ticket (AC-3, restated 2026-10-10) | asserted in the card container spec: `control-status` text and title, no raw code, cleared by a later admitted entry; no golden, since no visual arm covers the admission line | — | CARD-CONTRACT.md | unit (AC-3). Was: "the card's status line golden · the visual gate · one capture per skin · visual (AC-3)" |

Decision Record impact: `none`.

## Acceptance Criteria

- [ ] AC-1 — unit: a `proof-unavailable` entry younger than the budget words `New Memory Core connection waiting` with the probe reason in the title and `restart: false`; an entry older than the budget words the restart hint with `restart: true`; an entry whose reason is outside the closed vocabulary prints no reason.
- [ ] AC-2 — unit: the two credential codes' words are unchanged (the existing tests pass), and an unknown code still falls through to the server line.
- [ ] AC-3 — the rendered card line (`control-status` text and title, no raw code, cleared by a later admitted entry) is asserted in the card container spec; the input stamp is re-taken for the util change; `CARD-CONTRACT.md` names the line. No golden: no visual arm covers the admission line (restated 2026-10-10, see Fix 3).
- [ ] AC-4 — post-merge, on an installed candidate with Brain #964: a Claude seat started while the plane is busy shows the waiting line, then the connected server line — recorded on Institution #12 beside #964's AC-6.

## Out of Scope

The launcher's budget and the refusal codes (Brain #964); the Agent Detail's admission block, if it ever words these entries (a sibling leaf); Codex seats (no launcher rows).

## Related

#590 / #597 (the refusal words), #477 (parent), #654 (the paced fleet start), Brain #964 (AC-3's residual owner is this leaf), #12 (installed acceptance).

Sweeps: live latest-20 open Institution issues read 2026-10-10 13:55:24Z — no equivalent (#654 is the pacing leaf); A2A last 30 — Sophie's #964 implementation note asks for this leaf, no competing claim; Memory Core — Emmy's planned consumer leaf (2026-10-07) became #590, closed by #597, which words only the two credential codes; own assignments (#505 #507 #351 #654) — none on this surface; structure map: an Institution util + card leaf, no new file.

Retrieval Hint: "SeatLaunchAdmission proof-unavailable waiting line probe reason restart hint"

Origin Session ID: f45d36fd-6e77-4c89-bd56-dd49d95b5b0a

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session f45d36fd-6e77-4c89-bd56-dd49d95b5b0a



## Timeline

- 2026-10-10T13:56:23Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-10T13:56:24Z @neo-fable-clio added the `enhancement` label
- 2026-10-10T13:56:24Z @neo-fable-clio added the `agent-os` label
- 2026-10-10T13:56:24Z @neo-fable-clio added the `ai` label
- 2026-10-10T13:56:24Z @neo-fable-clio added the `design` label
- 2026-10-10T13:56:32Z @neo-fable-clio added parent issue #477
- 2026-10-10T14:01:42Z @neo-gpt-sophie cross-referenced by PR #966
- 2026-10-10T14:07:27Z @neo-fable-clio cross-referenced by #658
- 2026-10-10T14:33:02Z @neo-gpt-sophie cross-referenced by #964
- 2026-10-10T14:48:01Z @neo-gpt-sophie cross-referenced by #12
- 2026-10-10T16:24:28Z @neo-fable-clio cross-referenced by PR #660
- 2026-10-10T17:06:13Z @neo-fable-clio referenced in commit `72cd10b` - "fix(agentos): the admission words age on the card's own wait, dated from the run's first unanswered proof (#655)

Sophie's R1 on the first head: the card printed any string as the issuer's diagnostic, dated the launcher's budget from the latest audit answer instead of the launcher's start, and had no path to change its words while the record stayed equal. The diagnostic is now gated on the contract's isLaunchAdmissionProofReason (Brain #973); the budget is dated from the run's first proof-unavailable entry per server, a certain lower bound on the launcher's elapsed time; the waiting line names its instant and the card arms one owned wait for it, re-applying the record at the budget's end and retiring the wait with the words and with the card; afterSetNow re-applies the record. The terminal words name the issuer's recorded fact (not admitted), never Desktop's connection. The card's JSDoc is compressed to stay under the 1000-line bar."
- 2026-10-10T17:19:18Z @neo-gpt-sophie cross-referenced by PR #975
- 2026-10-10T17:45:17Z @neo-fable-clio referenced in commit `99aa264` - "fix(agentos): the admission words observe a run's duration and advise, never a launcher's state (#655)

Sophie's R2: the terminal title claimed the launcher no longer retries, but Desktop may start a later launcher under the same generation, each with its own budget, so no retry state is observable from the audit. Past one launcher's budget the line reads 'still unanswered' and advises a restart; the budget and waitingLine JSDoc and CARD-CONTRACT.md say what the words do not claim."
- 2026-10-10T18:06:25Z @tobiu referenced in commit `db1c8e6` - "feat(agentos): the seat card reads an unanswered launch proof as waiting, not refused (#655) (#660)

* feat(agentos): the seat card reads an unanswered launch proof as waiting, not refused (#655)

Brain #964 / #966 answer a proof the plane did not answer with a new refusal,
proof-unavailable, whose audit reason is the issuer's bounded diagnostic or
the credential's kind. The card words it: "New <server> connection waiting"
while the seat's launcher retries within its fixed 60 s budget, "New <server>
connection not made" once the entry outlives it, with a restart hint only when
the lifecycle controls allow one; the title names the credential and prints
the diagnostic as it is, never a code. The credential codes keep their words;
an unknown code still falls through. SeatLaunchAdmission gains waitingLine, a
now clock and the static LAUNCH_STARTUP_BUDGET_MS; CARD-CONTRACT.md names
the line.

* fix(agentos): the admission words age on the card's own wait, dated from the run's first unanswered proof (#655)

Sophie's R1 on the first head: the card printed any string as the issuer's diagnostic, dated the launcher's budget from the latest audit answer instead of the launcher's start, and had no path to change its words while the record stayed equal. The diagnostic is now gated on the contract's isLaunchAdmissionProofReason (Brain #973); the budget is dated from the run's first proof-unavailable entry per server, a certain lower bound on the launcher's elapsed time; the waiting line names its instant and the card arms one owned wait for it, re-applying the record at the budget's end and retiring the wait with the words and with the card; afterSetNow re-applies the record. The terminal words name the issuer's recorded fact (not admitted), never Desktop's connection. The card's JSDoc is compressed to stay under the 1000-line bar.

* fix(agentos): the admission words observe a run's duration and advise, never a launcher's state (#655)

Sophie's R2: the terminal title claimed the launcher no longer retries, but Desktop may start a later launcher under the same generation, each with its own budget, so no retry state is observable from the audit. Past one launcher's budget the line reads 'still unanswered' and advises a restart; the budget and waitingLine JSDoc and CARD-CONTRACT.md say what the words do not claim."
- 2026-10-10T18:06:26Z @tobiu closed this issue

