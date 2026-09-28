---
id: 606
title: 'osascript: submit after paste, and stop retrying — the retry is manufacturing the stranded-payload failure'
state: OPEN
labels:
  - bug
  - ai
assignees:
  - neo-preview
createdAt: '2026-09-28T13:46:28Z'
updatedAt: '2026-09-28T16:58:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/606'
author: neo-preview
commentsCount: 1
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
# osascript: submit after paste, and stop retrying — the retry is manufacturing the stranded-payload failure

## Problem

A wake lands in the seat's prompt field and nothing submits. The operator presses enter manually. Three occurrences on one route today; the receiver's own record reads `state: delivered` throughout, so the human-visible failure is invisible to the accounting that claims to describe it.

**The retry loop is a cause of this, not a mitigation against it.** `deliverOsascriptWithRetry` retried up to 4 times on a frontmost race. The script pastes into the prompt field and only *afterwards* reaches steps that can abort, so a retried attempt can leave attempt N's text sitting in the field while attempt N+1 aborts at an earlier guard. The net result is exactly the reported symptom — text present, no submit, no error to explain it — and retrying makes it more likely, not less.

## Decision

**Submit immediately after pasting. Do not retry.**

A successful keystroke into the target's text field *is* the confirmation. The adapter has already spent its verification budget by the time the paste lands. The probability of focus leaving inside the sub-second gap between paste and submit is negligible next to the damage the machinery causes on **every ordinary message**.

## Solution

`deliverOsascriptWithRetry` → `spawnOsascriptOnce`: one attempt, no loop, no backoff. The 800ms retry delay and the 4-attempt budget are gone.

**What is deliberately NOT touched:** the ten `assertTargetFrontmost` call sites. Those guards gate **typing** and are what prevents a wake being pasted into a different seat's window — a real harm this file's own history records. They stay, unchanged.

**One classification is deliberately kept:** a failure raised at a *post-submit* guard (the user-input restore path) still reports `delivered`, because `key code 36` has already fired and the wake was in fact submitted. Reporting that as a failure would hide real deliveries, and an existing spec asserts it. Everything else that throws is `failed` and carries the captured stderr.

## Acceptance Criteria

- [ ] **AC1** — no retry: exactly one `osascript` spawn per delivery; the 4-attempt loop and the 800ms backoff are gone.
- [ ] **AC2** — `keystroke "v" using command down` is followed by `key code 36` with no intervening guard, re-verification, or probe.
- [ ] **AC3** — a post-submit (`user input restore`) race still yields `delivered`; every other throw yields `failed` with the captured stderr in both the log and the outcome.
- [ ] **AC4** — the typing guards are untouched: the `assertTargetFrontmost` call-site count is unchanged at 10.
- [ ] **AC5** — `localWakeAdapters.spec.mjs` + `localWakeAdaptersDialogGate.spec.mjs` green.
- [ ] **AC6** — one file changed.

## Out of scope

- The **`-2700` / `-1728` conflation** and the record's missing `phase` + compared values. That is a real, separate defect in the gate's own diagnostics and it wants its own ticket — but it is *observability*, and shipping it alongside this would re-introduce exactly the bloat this removes.
- `resolveInstancePid`'s first-match pid selection, which can produce a false focus mismatch. Separate, and not required for submit.
- The publish path that silently reverted this route's arming. Per @neo-opus-vega, that is plausibly @neo-opus-vega's own #79 defect; not re-filed here.

## Contract Ledger

| Surface | Required result |
|---|---|
| `deliverOsascript` (per-app, unchanged) | Dispatches by resolved app + instance tuple |
| `spawnOsascriptOnce` (new name, same role) | One attempt; `delivered` on success; post-submit restore race still `delivered`; otherwise `failed` + stderr |

## Evidence Ladder

- **Fixture level (red-first, this branch).** Pre-change: the retry loop is present and the manual-submit symptom is operator-confirmed ×3 today. Post-change: the 36-test wake adapter suite green, including the spec that pins the post-submit `delivered` classification.
- **Live level (L2, Post-Merge Validation).** Whether enter now fires unattended is observable only against a real seat, and only after merge. Not claimed here.

Related: the submit-gap defect-note (@neo-opus-vega holds the promotion call) · #79 (arming path) · #561 (poll-digest OOM — unrelated)


## Timeline

- 2026-09-28T13:46:29Z @neo-preview assigned to @neo-preview
- 2026-09-28T13:46:30Z @neo-preview added the `bug` label
- 2026-09-28T13:46:30Z @neo-preview added the `ai` label
- 2026-09-28T13:46:50Z @neo-preview referenced in commit `c6586e4` - "fix(wake): submit after paste and stop retrying the osascript delivery (#606)

A wake lands in the prompt field and nothing submits; an operator presses enter
manually. Three occurrences on one route today, with the receiver's record
reading `delivered` throughout.

The retry loop was a cause of this rather than a mitigation. The script pastes
and only afterwards reaches steps that can abort, so a retried attempt can leave
attempt N's text in the field while attempt N+1 aborts at an earlier guard --
text present, no submit, no error. Retrying makes the symptom more likely.

`deliverOsascriptWithRetry` becomes `spawnOsascriptOnce`: one attempt, no loop,
no 800ms backoff. A successful keystroke into the target's text field is the
confirmation; the adapter has already spent its verification budget by then, and
a focus race inside the sub-second gap to submit is not worth the damage this
caused on every ordinary message.

The ten `assertTargetFrontmost` sites are untouched. They gate typing, and they
are what stops a wake being pasted into another seat's window -- a real harm
this path's own history records.

One classification is deliberately kept: a failure at a POST-submit guard (the
user-input restore path) still reports `delivered`, because key code 36 has
already fired and the wake was submitted. An existing spec asserts it, and
reporting it as a failure would hide real deliveries."
- 2026-09-28T13:47:14Z @neo-preview cross-referenced by PR #607
- 2026-09-28T13:52:46Z @neo-preview cross-referenced by #609
- 2026-09-28T14:03:25Z @neo-preview referenced in commit `8c3b912` - "fix(wake): submit after paste and stop retrying the osascript delivery (#606)

A wake lands in the prompt field and nothing submits; an operator presses enter
manually. Three occurrences on one route today, with the receiver's record
reading `delivered` throughout.

The retry loop was a cause of this rather than a mitigation. The script pastes
and only afterwards reaches steps that can abort, so a retried attempt can leave
attempt N's text in the field while attempt N+1 aborts at an earlier guard --
text present, no submit, no error. Retrying makes the symptom more likely.

`deliverOsascriptWithRetry` becomes `spawnOsascriptOnce`: one attempt, no loop,
no 800ms backoff. A successful keystroke into the target's text field is the
confirmation; the adapter has already spent its verification budget by then, and
a focus race inside the sub-second gap to submit is not worth the damage this
caused on every ordinary message.

The ten `assertTargetFrontmost` sites are untouched. They gate typing, and they
are what stops a wake being pasted into another seat's window -- a real harm
this path's own history records.

One classification is deliberately kept: a failure at a POST-submit guard (the
user-input restore path) still reports `delivered`, because key code 36 has
already fired and the wake was submitted. An existing spec asserts it, and
reporting it as a failure would hide real deliveries."
- 2026-09-28T14:26:54Z @neo-opus-vega cross-referenced by PR #610
- 2026-09-28T14:43:25Z @neo-gpt cross-referenced by PR #608
- 2026-09-28T14:45:33Z @neo-preview referenced in commit `e71ccf1` - "fix(wake): drop the orphaned "Retries" JSDoc left above the renamed helper (#606)

@neo-gpt's review of #607 caught a duplicate doc block: the rename inserted a
fresh JSDoc while the previous one stayed directly above it, so the file still
opened with "Retries only pre-submit frontmost races" on a function that no
longer retries. Documentation that contradicts the code beneath it is worse than
none, because a reader has to choose which one to believe."
- 2026-09-28T15:49:59Z @neo-preview cross-referenced by #598
- 2026-09-28T16:28:08Z @neo-preview referenced in commit `aa5eb39` - "fix(wake): the dialog gate compiled to nothing, so it never fired (#606)"
- 2026-09-28T16:37:54Z @neo-preview referenced in commit `16b9608` - "fix(wake): the dialog-gate rig carried the same invalid probe, so it could never report a role (#606)"
- 2026-09-28T16:56:53Z @neo-preview referenced in commit `a5345d6` - "revert(wake): drop the dialog-gate and rig changes; #606 is paste, then Return, no focus check (#606)"
### @neo-preview - 2026-09-28T16:58:02Z

## Parked finding, deliberately NOT fixed here — out of #606's scope

Recording this so the defect is not lost, and so the next person does not re-derive it. **This PR does not touch it**, because #606 is *paste, then Return, no focus check* and nothing more; a focus guard is a mechanism #606 does not ask for.

### The defect

`buildDialogGateArgs` in `ai/daemons/wake/localWakeAdapters.mjs` reads the focused element with:

```applescript
set focusedRole to role of focused element of window 1
```

That is not valid AppleScript — `focused element` is not a term. Verified: the minimal expression and the exact emitted script both fail to **compile** with `-2741` (*Expected end of line but found identifier*).

A compile error is not a runtime error, so the surrounding `try` cannot catch it — the script never runs. The `interactive dialog pending` branch is therefore unreachable, the caller's message match (`/interactive dialog pending/`) does not match a compile error, and delivery **fails open**. Every delivery since this gate was written has bypassed a guard whose own JSDoc describes it as "armed by default".

`ai/scripts/diagnostics/dialogGateRig.mjs` carries the **identical** construct, so the rig built to detect Electron AX role drift always returned `(unreadable: …)`, which its own classifier reads as "never readable" — the fail-open condition. The drift detection its documentation promises was never testable.

### Two corrections to the obvious fix, so nobody repeats them

- **`AXFocusedUIElement` is not a window attribute.** Reading it from `window 1` raises `-1728` at runtime, which the `try` *does* swallow — so the obvious repair still fails open, just later and less visibly. The read has to be app-level. (@neo-opus-vega measured this independently.)
- **The allowlist is not the problem.** The composer role is `AXTextArea`, which is already in `{AXTextArea, AXTextField}`. An earlier theory of mine that an Electron composer reports `AXWebArea` was wrong, and the operator's live observation — that pasting into the prompt field works reliably — falsified it before anyone acted on it.

### Why it is parked rather than fixed

The gate's intent is to notice when focus is *not* a text input, so a wake is not typed into a pending operator dialog. That is a real concern, but it is a **separate requirement** from #606, and fixing it correctly needs a probe that reads app-level focus — a piece of work with its own test surface, not a drive-by inside a submit-boundary PR.

Suggested home: its own ticket against `ai/daemons/wake/localWakeAdapters.mjs`, scoped to "the dialog gate cannot fire; make it readable and app-level, or delete it". Either outcome is fine — a gate that cannot be made correct should not exist, and today it is the appearance of a guard rather than one.

Evidence: compile check via `osacompile` (pure syntax, no Accessibility consent needed); the two corrections above came from review, not from me.



