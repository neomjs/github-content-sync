---
id: 936
title: Wake activation must raise the addressed Codex instance
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-08T16:26:40Z'
updatedAt: '2026-10-09T05:12:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/936'
author: neo-gpt-emmy
commentsCount: 3
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
# Wake activation must raise the addressed Codex instance

## Context

The operator reports that Emmy receives wakes again while Sophie does not. Their production receiver is running. Sophie's self-probe and a peer message both reached that receiver and failed with `Target app lost frontmost status after activation (-2700)`; the immediately following Emmy dispatch succeeded. [Independent receiver evidence](https://github.com/neomjs/neo-agent-brain/issues/30#issuecomment-6064156878).

## The Problem

The subscription's managed `userDataDir` matches Sophie's actual main process. This is downstream of message persistence, filtering and webhook admission. Read-only macOS queries show `application "Codex"` resolves to the `Codex (Service).app` helper, bundle `com.openai.codex.helper`; the three main instances are `ChatGPT.app`, bundle `com.openai.codex`.

One activation-only System Events request for Sophie's verified PID returned Emmy as foreground. A native `NSRunningApplication` activation request for Sophie's PID was accepted, and a subsequent independent read observed Sophie foreground. This is a bounded host observation, not universal delivery proof. A later production wake still failed with the unchanged adapter, so manually selecting a window is not the repair.

## The Architectural Reality

`localWakeAdapters.mjs#deliverOsascript` correctly resolves the profile to a PID. `buildOsascriptArgs` then mixes generic application-name activation with a System Events frontmost assignment. Its guard also permits a different PID when bundle IDs match. Neither bound proves the addressed instance owns the input when multiple instances share a bundle.

Design authority: closed neomjs/neo#15054 requires that the "exactly resolved PID is raised before any prompt mutation when sibling residents exist" and that the live matrix excludes sibling delivery. This is a remaining activation subcase of that contract, not a change to mailbox ownership or app-name configuration. The deployed `2d839fc1` and current `aab9e2a0` adapter/resolver sources are identical.

## The Fix

For PID-addressed Codex delivery, bind native AppKit activation and bundle identity to the resolved `NSRunningApplication`, avoiding application-name alias lookup/activation. Require that same PID at the existing pre-input and paste guards. Preserve draft restoration, one dispatch attempt, and the existing non-Codex path. Keep `appName: Codex` and stable `userDataDir` registration unchanged.

Use the existing `localWakeAdapters.mjs` and its spec. No new module, adapter, subscription, service or global activation rename is needed.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Codex PID activation | `deliverOsascript`, neomjs/neo#15054 | Native activation and bundle identity come from the resolved PID | Missing/refused target fails before typing; no alias fallback | Builder/dispatch JSDoc | Generated-script regression controls, native compile/activation probe |
| Codex frontmost guard | `assertTargetFrontmost` in generated script | A sibling PID never qualifies merely by sharing a bundle | Refuse before clipboard/input mutation at existing guard boundaries | Existing guard documentation | Same-bundle negative control; retained draft/submit controls |
| Other harnesses and route tuple | Existing adapter and resolver contracts | Preserve non-Codex behavior and subscription metadata | Existing failure behavior | No route/schema change | Existing resolver/adapter suites |

Decision Record impact: none; restores the accepted instance-addressing contract.

## Acceptance Criteria

- [ ] A PID-addressed Codex script activates that PID without resolving or activating the generic application name.
- [ ] A same-bundle sibling PID cannot pass the Codex input guards; missing targets and failed activation cannot cause typing.
- [ ] Existing draft preservation, submit/restore boundaries, non-Codex adapter behavior and resolver checks remain passing.
- [ ] The generated native script compiles, and bounded activation-only checks record what was actually raised without typing into a live draft.
- [ ] Post-merge, installed: controlled production delivery reaches the intended Sophie and Emmy chats, with no sibling delivery, before fleet-wide recovery is claimed. Residual owner: #30.

## Out of Scope

Native cross-session messaging (#30/#768), subscription replacement, credential rotation, new retry loops, whole-computer or harness restarts, and FM installation.

## Avoided Traps

Do not interpret a correct profile/PID lookup as successful activation. Do not rename every Codex route to ChatGPT, loosen the guard to the helper bundle, or infer a turn started from an adapter exit code. Keep receipts distinct from an explanation of every earlier intermittent failure.

## Related

neomjs/neo#15054 · #30 · #768

Live latest-open and own-assignment sweeps on 2026-10-08 found no open activation repair; #30 is the separate native-messaging outcome. A2A recency found only Sophie's independent incident receipt, no competing implementation claim. MC `ece7ce88` from the predecessor's origin session recovers the exact-instance intent. An independent source/test audit confirmed the missing activation-binding control. Brain structure-map was read in this session; the existing wake adapter/spec own the repair.

Origin Session ID: 7cdef292-c073-447b-9afd-4eaab22ecdbf
Retrieval Hint: "Codex helper alias correct profile PID wrong foreground wake"


## Timeline

- 2026-10-08T16:26:41Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-08T16:26:42Z @neo-gpt-emmy added the `bug` label
- 2026-10-08T16:26:43Z @neo-gpt-emmy added the `ai` label
- 2026-10-08T16:26:43Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-08T16:30:01Z @neo-gpt-emmy cross-referenced by #30
### @neo-gpt-emmy - 2026-10-09T01:12:46Z

### Emmy migration handoff: parked, not shipped

The local branch `codex/936-codex-wake-activation` remains at `aab9e2a0`, with two uncommitted files: `ai/daemons/wake/localWakeAdapters.mjs` and its existing unit spec. The same patch is preserved in named stash `7263d77f1b69133cd79df395e55eb6040ec1e6f4` (`emmy-candidate-f-preserve-936-20261008`); restoration was verified byte-for-byte. No commit, PR or runtime installation was made for this patch.

The operator chose a computer restart during the original investigation, and subsequent Emmy/Sophie wake receipts passed. Do not ship the parked activation change merely because it exists. Revalidate the current wake failure and exact addressed-process behavior before resuming this ticket. The newer external-close/Stop-card observation belongs to the separate lifecycle evidence on #571, not to this unproven activation repair.

The old checkout and stash remain retained during Emmy's Fleet move. The new session can recover the patch there without overwriting fresh managed source.

Origin Session ID: 7cdef292-c073-447b-9afd-4eaab22ecdbf

- 2026-10-09T01:21:47Z @neo-gpt-emmy cross-referenced by #571
### @neo-gpt-emmy - 2026-10-09T04:31:23Z

### Post-migration source revalidation

At current `origin/dev` (`0b678f47806085094eb87d03bb881c7b26f1c59a`), `buildOsascriptArgs` still emits application-name lookup/activation and, after a PID mismatch, only raises the guard error when the bundle also differs (`localWakeAdapters.mjs:754–809`). The implementation premise has not disappeared from source.

Emmy's successful managed idle-wake receipt on #571 is one positive delivery witness; it does not resolve this ticket's same-bundle negative control or prove the earlier failure's current frequency. The parked patch remains a candidate, not a shipped repair.

Next bounded step: compare the preserved patch against current source, run the exact-addressed-PID negative controls and native compilation without activation or typing. Live delivery and deployment remain separately witnessed under #30. No live focus test, route change, old-harness restart or patch publication was performed in this revalidation.

Origin Session ID: b56dbc41-6e95-4210-a2ea-8d1f5f3ffcd0

### @neo-gpt-emmy - 2026-10-09T05:12:06Z

### Candidate source and compilation receipt

Compared current `origin/dev` at `0b678f47806085094eb87d03bb881c7b26f1c59a` with the preserved two-file patch. The candidate still fits the source gap: Codex activation and bundle identity come from the resolved `NSRunningApplication`, and its guard rejects a different PID before the generic same-bundle fallback. The existing non-Codex path and guard sites remain present. The patch remains uncommitted and uninstalled.

Generated the candidate through `dispatchLocalWake` with an injected PID resolver and a no-op spawn, then compiled the emitted AppleScript with `osacompile`: exit 0. This proves native script syntax, not activation behavior.

For a no-focus-change negative control, inspected isolated baseline/candidate guard handlers and attempted a read-only System Events foreground query. It returned no result before being interrupted after about 17 seconds. Foreground identity was therefore unknown, so neither guard was invoked and no negative-control pass is claimed. No activation, typing, clipboard write, route change, old-harness launch or checkout mutation occurred; the timeout's cause is unproven.

Remaining pre-publication evidence: an activation-only witness must record the intended and observed foreground PIDs, including the same-bundle sibling and missing/refused-target cases, without typing. The previously successful wake after reboot does not supply those controls. Installed production delivery remains on #30.

Origin Session ID: b56dbc41-6e95-4210-a2ea-8d1f5f3ffcd0


