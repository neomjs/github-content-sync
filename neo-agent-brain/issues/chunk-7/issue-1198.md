---
id: 1198
title: Fleet boot hashes whole runtimes before its transport listens
state: OPEN
labels:
  - bug
  - ai
  - regression
  - performance
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-19T15:11:56Z'
updatedAt: '2026-10-20T08:36:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1198'
author: neo-opus-vega
commentsCount: 1
parentIssue: 732
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
# Fleet boot hashes whole runtimes before its transport listens

## Context

This is #1142's AC-3 leaf for the `runtime bind` and `recovery` components. Julia's breakdown names it Leaf 2 ([#1142 comment](https://github.com/neomjs/neo-agent-brain/issues/1142#issuecomment-6269645262)).

**Observed (installed, 2026-10-19).** Brain `48b8812b`, which contains #1186, booted after an FM update. Read from the FM's `main.log`:

| Line | Time | Since previous |
|---|---|---|
| `HARNESS_BRAIN_PLAN` | 14:39:04.070Z | — |
| `seams bound … verified plane-side` | 14:39:16.419Z | 12.3 s (runtime bind + viewer claim + plane init) |
| `continuity recovery: previous issuer unclean-issuer-exit; 10 retained generation(s)`, all stale or revoked | 14:40:05.111Z | **48.7 s** |
| `transport listening` | 14:40:05.129Z | 18 ms |

- The shell's window ran out at 14:39:19Z (`fleet transport not ready after 15000ms; still waiting`), and the roster arrived 46 s later. The 14:31Z boot on Brain `99804a5` had the same shape, at 64 s. Five runtime generations were on disk.
- Those 48.7 s returned no authority. After an unclean exit every retained generation is stale and admits nothing (ADR 0034 §2.5 item 4).

## The Problem

After an FM update the cockpit shows no roster for about a minute. Every boot that retains pins pays the same cost, and the window grows with the number of distinct runtimes the seats were started on. Per the #1142 breakdown, `runtime bind` costs 2.9–6.7 s and `recovery` about 6 s per distinct pin, against a ≤ 2.3 s baseline.

## The Architectural Reality

- **Runtime bind.** `runtimeGeneration.mjs:293` `bindCurrentRuntime` calls `readRuntimeGeneration` at `:314`: a full inventory and SHA-256 of the running generation (23,747 files, 965 MB) on every packaged boot. The installer already verified those bytes when it prepared the generation (Institution `harness/install.mjs`, "prepare verified runtime").
- **Recovery.** `FleetLifecycleService.mjs:2357` `recoverRuntimeCustodyUnderGuard` defaults `verifyRuntime` to `readRuntimeGeneration`:
  - `:2403` `restoreSeat` is the path that returns authority, after a proved handoff.
  - `:2406` re-verifies every retained seat in sequence, stale ones included. It only sets `runtimeUnresolved`, which the lifecycle uses for display and retirement. That pass spent the 48.7 s.
  - `verified` (`:2360`) starts empty, so a seat pinned to the running generation re-hashes what `bindCurrentRuntime` just hashed.
- **Ordering.** `devFleetServer.mjs` runs both before `startFleetBridgeServer`, so the shell's 15 s window absorbs them.
- **Invariant to keep.** #1185 moved the handoff to record reads because "the successor's `verifyPin` re-verifies the bytes". Its AC-3 reads: "Bytes are still checked before authority returns."

Design authority: #1142's, namely the operator's line and the FM shell's 15 s readiness window (Institution `harness/brain.mjs`, `awaitFleetReady`), plus the outcome of neomjs/neo-agent-institution#732 (an FM update never requires closing a harness).

## The Fix

1. **Hash only where authority returns.**
   - The `:2406` pass reads records (`readRuntimeGenerationRecord`), because it grants nothing.
   - `restoreSeat` keeps the full read.
   - Seed `verified` with the generation `bindCurrentRuntime` read, so the running generation is inventoried at most once per boot.
2. **Move the inventories past the listening transport.**
   - `bindCurrentRuntime` binds by the record at boot.
   - The transport starts listening while `continuityPhase` still excludes Starts.
   - Then the running generation and each authority-returning pin are inventoried. A seat's authority returns only after its pin verifies. A failed inventory of the running generation leaves the issuer refusing Starts, by name.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `recoverRuntimeCustodyUnderGuard`, `:2406` pass | ADR 0034 §2.5 items 4–5 | record read; sets `runtimeUnresolved` only | an injected `verifyRuntime` still overrides | method JSDoc | AC-1 |
| `recoverRuntimeCustodyUnderGuard`, `verified` | same | seeded from the boot bind's read | none | method JSDoc | AC-3 |
| `bindCurrentRuntime` | #1185 AC-3 (bytes before authority) | record bind at boot; payload inventory follows the transport | a failed inventory refuses Starts | function JSDoc | AC-2, AC-4 |
| `devFleetServer.mjs` boot order | the shell's 15 s window | transport listens before any payload inventory | Starts excluded until recovery settles | module JSDoc | AC-4 |

Decision Record impact: aligned-with ADR 0034 §2.5. Authority still returns only after a byte check. I read the ADR for an ordering clause that binds recovery before the transport listens and found none.

## Acceptance Criteria

- [ ] AC-1 (unit): after an unclean issuer exit, recovery runs no payload inventory. Every retained generation stays stale, and a `verifyRuntime` spy counting full reads sees zero.
- [ ] AC-2 (unit): after a proved handoff, a seat's authority returns only after its pin's payload verifies. A pin whose bytes changed after the handoff stays stale with `runtime pin unverified`, so #1185's AC-3 still holds.
- [ ] AC-3 (unit): a seat pinned to the running generation causes no second inventory of it.
- [ ] AC-4 (unit): the transport listens before any payload inventory runs, and a Start is refused until recovery settles.
- [ ] AC-5 [L3-deferred — operator handoff needed]: on the next FM update with ≥ 5 running seats, #1142's phase lines show `runtime bind` ≤ 50 ms and `recovery` ≤ 50 ms, `transport listening` follows `HARNESS_BRAIN_PLAN` within 2.5 s, and each proved seat turns active after its verification.
  - **Carries #1200 AC-4 (residual of PR #1206):** in the same run, Euclid, Emmy and Sophie (`codex-desktop`) are running; each logs `survivor <id>: verified` and the successor lists them `active`. With the host Bridge on a revision carrying `2d62f2e6`, each also appears in the Bridge's agent list and its NL calls answer. A run without the three Codex seats does not retire this line.

## Out of Scope

- Memory Core's WAL scans that slow `plane init` (#1196).
- The handoff's own reads (#1185, closed).
- The shell's single 10 s SIGKILL grace on quit (a defect-note, 2026-10-19).
- The `gh api user` viewer claim.

## Avoided Traps

- **Dropping the byte check.** It breaks #1185's AC-3, and nothing would check the bytes after install.
- **Returning authority optimistically** before the background verification ends.
- **Treating inode, size or mtime as the integrity proof.** They identify the generation; only the bytes prove it.

## Related

#1142 (parent exploration), #1185 / #1186 (handoff component), #1196 (plane init), #1197 (the phase lines AC-5 reads), neomjs/neo-agent-institution#732.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-19T15:10Z; nearest #1142 (the parent) and #1081 (quit/handoff budget), neither equivalent.
A2A sweep: last 30 messages, no claim on this scope.
MC sweep: "boot takes a minute after update, roster empty, recovery re-hashes retained runtimes" plus "bindCurrentRuntime / verifyPin re-hash filed ticket", 11 results, no prior decision found.
Own-assignment sweep: 10 open, none overlapping.
Structure map: `ai/services/fleet` owns all three files; no new files.

Origin Session ID: f305e423-d492-448d-8a00-db6ba01398cc
Retrieval Hint: "fleet boot recovery verifyPin readRuntimeGeneration stale unclean-issuer-exit 48.7 s bindCurrentRuntime record read before transport"

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿

## Timeline

- 2026-10-19T15:11:57Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-19T15:12:06Z @neo-opus-vega added the `bug` label
- 2026-10-19T15:12:07Z @neo-opus-vega added the `ai` label
- 2026-10-19T15:12:07Z @neo-opus-vega added the `regression` label
- 2026-10-19T15:12:07Z @neo-opus-vega added the `performance` label
- 2026-10-19T15:12:07Z @neo-opus-vega added the `agent-os` label
- 2026-10-19T15:12:45Z @neo-opus-vega added parent issue #732
- 2026-10-19T15:15:05Z @neo-opus-vega cross-referenced by PR #1197
- 2026-10-19T15:28:19Z @neo-opus-vega cross-referenced by #1200
- 2026-10-19T15:33:34Z @neo-opus-vega cross-referenced by #739
- 2026-10-19T15:37:27Z @neo-opus-ada cross-referenced by PR #1193
- 2026-10-19T15:57:30Z @neo-fable-mnemosyne cross-referenced by PR #1199
- 2026-10-19T16:01:07Z @neo-fable-mnemosyne cross-referenced by #1204
- 2026-10-19T16:24:31Z @neo-opus-vega cross-referenced by PR #1206
- 2026-10-19T19:26:43Z @neo-opus-vega cross-referenced by PR #1214
- 2026-10-19T21:47:41Z @neo-opus-vega cross-referenced by #1217
- 2026-10-20T08:36:43Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-20T08:36:43Z @neo-gemini-julia unassigned from @neo-opus-vega
### @neo-gemini-julia - 2026-10-20T08:36:44Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-opus-vega`
**New assignees:** `@me`
**Reason:** handoff from Vega (MESSAGE:652c8a39-55ce-4b1b-bd51-f0c6bc777d8c)

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-10-20T09:45:22Z @neo-gemini-julia cross-referenced by PR #1224

