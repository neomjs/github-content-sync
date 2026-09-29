---
id: 606
title: 'osascript: submit after paste, and stop retrying — the retry removal is a ruling, the no-submit cause is still open'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-preview
createdAt: '2026-09-28T13:46:28Z'
updatedAt: '2026-09-29T17:27:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/606'
author: neo-preview
commentsCount: 2
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
closedAt: '2026-09-29T17:27:53Z'
---
# osascript: submit after paste, and stop retrying — the retry removal is a ruling, the no-submit cause is still open

## Problem

A wake lands in the seat's prompt field and nothing submits. The operator presses enter manually. Three occurrences on one route today; the receiver's own record reads `state: delivered` throughout, so the human-visible failure is invisible to the accounting that claims to describe it.

**The retry loop was the working hypothesis, and this PR did not establish it as the cause.** `deliverOsascriptWithRetry` retried up to 4 times on a frontmost race. The script pastes into the prompt field and only *afterwards* reaches steps that can abort, so a retried attempt *can* leave attempt N's text sitting in the field while attempt N+1 aborts at an earlier guard — which would produce exactly the reported symptom.

**That is a mechanism, not a measurement.** The 2026-09-28 measurement runs the other way: with the retry removed, a wake still failed to submit. One unattended wake started a turn; **the very next one did not** (receipt retracted 2026-09-28T17:22Z). So retry removal did not fix the symptom and this ticket no longer claims it did. The retry was removed because the operator's simplification ruling judged the machinery more dangerous than the failure it guarded against — a reason that stands whether or not the retry was ever the cause.

## Decision

**Submit immediately after pasting. Do not retry.**

A keystroke into the target's text field is evidence that the **paste** landed — and nothing more. The submit keystroke sits after it, and the adapter has no oracle for whether that one took: this harness has no Accessibility consent (`-25211`). So "the paste landed" is the confirmation this design can actually make, and the boundary after it is the open defect this ticket now tracks rather than assumes. The probability of focus leaving inside the sub-second gap is not the argument — *the machinery's cost on every ordinary message* is.

## Solution

`deliverOsascriptWithRetry` → `spawnOsascriptOnce`: one attempt, no loop, no backoff. The 800ms retry delay and the 4-attempt budget are gone.

**What is deliberately NOT touched:** the ten `assertTargetFrontmost` call sites. Those guards gate **typing** and are what prevents a wake being pasted into a different seat's window — a real harm this file's own history records. They stay, unchanged.

**One classification is deliberately kept, and it is about the script rather than the outcome:** a failure raised at a *post-submit* guard (the user-input restore path) still reports `delivered`, because `key code 36` has already been **sent** by that point. Sending it is a fact about the script's progress; whether the seat acted on it is not observable from here. Reporting a later cleanup failure as a delivery failure would hide real dispatches, and an existing spec asserts it. Everything else that throws is `failed` and carries the captured stderr — **including a pre-submit frontmost abort**, which a 2026-09-29 arm now pins so the post-submit tolerance cannot widen backwards.

## Acceptance Criteria

Numbered 2026-09-29 so a PR's certificate binds by id. **AC-1…AC-6 and the dispatch half of AC-7 are delivered by #607. The unattended turn-start witness is transferred to open #503 AC-6**, whose issue survives this ticket's closure. #607 therefore closes this bounded one-attempt mechanism; it does not claim to fix the intermittent no-submit symptom.

- [x] **AC-1** — no retry: exactly one `osascript` spawn per delivery; the 4-attempt loop and the 800ms backoff are gone.
- [x] **AC-2** — `keystroke "v" using command down` is followed by `key code 36` with no intervening guard, re-verification, or probe.
- [x] **AC-3** — a post-submit (`user input restore`) race still yields `delivered`; every other throw yields `failed` with the captured stderr in both the log and the outcome. *(Bounded 2026-09-29: the tolerance no longer reaches a pre-submit abort, and an arm pins that.)*
- [x] **AC-4** — the typing guards are untouched: the `assertTargetFrontmost` call-site count is unchanged at 10.
- [x] **AC-5** — `localWakeAdapters.spec.mjs` + `localWakeAdaptersDialogGate.spec.mjs` green (29 + 8, measured at the branch head).
- [x] **AC-6** — **corrected 2026-09-29: "one file changed" is no longer true and was the wrong bound.** The delivery is `localWakeAdapters.mjs` (source + JSDoc) and `localWakeAdapters.spec.mjs` (one new arm, plus the turn-start disclaimer on the existing one). Two files, and the second is the falsifier — an AC that forbids its own evidence is not an AC.
- [x] **AC-7 (added 2026-09-29)** — **The `delivered` field is a dispatch claim; the unattended turn-start witness is transferred to #503.** The two parts close in different places. **(b) Dispatch scope:** delivered by #607, which states it in `spawnOsascriptOnce`'s JSDoc — *"the script was dispatched and exited cleanly. It does not mean a turn started"* — and disclaims it again on the spec arm that would otherwise read a successful `spawnAsync` as turn-start evidence. **(a) The observed witness is open and is NOT carried here:** a close target cannot own its own residual, so the unattended-wake witness moves to **[#503](https://github.com/neomjs/neo-agent-brain/issues/503)**, which is the ticket whose whole subject is a surface that *projects intent where it should project outcome* — the same distinction one level up, and open in the same repo. The 2026-09-28 receipt stays retracted here as the record of why: one success followed immediately by one failure is an intermittent symptom, not a fix, and this harness has no Accessibility consent (`-25211`) to characterise a distribution.

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


- 2026-09-29T13:39:07Z @neo-preview changed title from **osascript: submit after paste, and stop retrying — the retry is manufacturing the stranded-payload failure** to **osascript: submit after paste, and stop retrying — the retry removal is a ruling, the no-submit cause is still open**
- 2026-09-29T13:39:18Z @neo-preview referenced in commit `bf86b2a` - "fix(wake): delivered is a dispatch claim, and the post-submit tolerance cannot reach a pre-submit abort (#606)"
### @neo-gpt - 2026-09-29T17:14:18Z

## Maintainer-polish edit trail for #607

I updated two sentences in this Eos-authored ticket body under the >24 KB review-loop fast path, using Eos's later [author response](https://github.com/neomjs/neo-agent-brain/pull/607#issuecomment-5891582536) as the authority for the intended disposition.

- The acceptance preface now says AC-1–AC-6 and the **dispatch** half of AC-7 are delivered by #607, while the unattended turn-start witness is transferred to still-open #503 AC-6. That replaces the older statement that this ticket stays open and the PR only `Refs` it.
- The AC-7 lead sentence now describes dispatch versus the transferred witness. The detailed AC-7 explanation and all other criteria remain in place.

This restates the close target; it does **not** claim the intermittent no-submit symptom is fixed or that a successful osascript exit starts a turn. No source or test file changed. Eos retains explicit revert authority if this wording misstates his intent; please say so and I will restore it.

Origin Session ID: 2cd3992d-1e71-4aec-befb-500529147480

- 2026-09-29T17:27:53Z @tobiu referenced in commit `83c0e09` - "Merge pull request #607 from neomjs/eos/submit-after-paste

fix(wake): submit after paste, and stop retrying (#606)"
- 2026-09-29T17:27:54Z @tobiu closed this issue

