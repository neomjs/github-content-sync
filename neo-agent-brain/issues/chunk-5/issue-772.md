---
id: 772
title: Lifecycle failure reasons copy a caught message onto the roster
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-02T17:12:42Z'
updatedAt: '2026-10-02T17:38:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/772'
author: neo-opus-ada
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
closedAt: '2026-10-02T17:38:53Z'
---
# Lifecycle failure reasons copy a caught message onto the roster

## Context

- **Promoted from defect-note** `MESSAGE:338de443-e67f-4311-ac09-c9a43db18fc6` (`@neo-opus-grace`, 2026-10-02 14:15Z):
  - `finalizeCodexDesktopHelpers` writes `failureReason = Codex Desktop helper cleanup failed: ${error?.message}`;
  - `FleetManager` copies `failureReason` into roster rows.
  - Its author routed it to me (15:57Z), to be taken after #753 merged.
- **#753 set the contract for the start path:** a system code may travel, a message never does, and the Fleet log keeps the cause. `writeSeatLease` implements it, and the bridge's `rejectionOf` surfaces caller-named refusals without their prefix.
- **Measured against the real producers at Brain dev `2fa96b5`:**

| Case | What the status reads today |
|---|---|
| `cleanupCodexDesktopCrashpad`, ambiguous scan | `Codex Desktop helper cleanup failed: cleanupCodexDesktopCrashpad: ambiguous profile-owned process identity (pid 4242); refusing cleanup.` (an internal function name in operator text) |
| A `kill` refusal | `… failed: kill EPERM` (the raw message, although `error.code` is `EPERM`) |
| `adoptLeasedSeats` (`FleetLifecycleService.mjs:1101`), whose reachable throws are `deriveAgentInstanceHome`'s refusals | `the seat lease could not be read: ${error.code \|\| error.message}`; those refusals embed the received value or derived path (`… must be an absolute path, received '<value>'`), so a filesystem path reaches the wire |

- No secret reaches the roster today. Each copy is unbounded, so the bound rests on every producer's discipline rather than on the Fleet's.

## The Fix

1. **One module helper in `FleetLifecycleService`.** A status may carry:
   - a refusal worded by a named caller, with the `<caller>: ` prefix removed;
   - otherwise a system error code (`SYSTEM_ERROR_CODE`);
   - otherwise nothing, in which case it takes bounded words and `console.error` keeps the cause.
2. **`finalizeCodexDesktopHelpers`** passes `cleanupCodexDesktopCrashpad` and `inspectCodexDesktopCrashpadProcesses` as its callers. Their refusals are written for the operator and carry only pids and counts.
3. **`adoptLeasedSeats`** passes no callers, because its refusals embed paths. It gets a code or `the seat lease could not be read; the Fleet log names the cause`.
4. **`writeSeatLease`** uses the same helper, with no change to its output.

## Acceptance Criteria

- [ ] **AC-1:** each cleanup failure reaches the status as follows:
  - an ambiguous cleanup reads `Codex Desktop helper cleanup failed: ambiguous profile-owned process identity (pid 4242); refusing cleanup.`;
  - a `kill` refusal reads `… failed: EPERM`;
  - any other message never reaches the status, and the Fleet log names it.
- [ ] **AC-2:** a lease-read failure carries its system code or the bounded words, never its message. A canary path in the thrown message appears in the Fleet log and never in the status.
- [ ] **AC-3:** `writeSeatLease`'s output is unchanged, so #751's canary test stays green.

## Out of Scope

- `FleetManager`'s roster copy, which carries whatever the producer wrote.
- The cleanup module's own wording.

## Related

#751 · #753

## Sweeps

- Live latest-open sweep: checked the latest 20 open issues at 2026-10-02T17:12Z, plus `gh search issues "failureReason"`; no equivalent found.
- A2A in-flight claim sweep: the last 15 messages (all read-states, back to 16:34Z); no overlapping claim.
- MC sweep: one query on the symptom (roster `failureReason` carrying a raw caught message, from Codex helper cleanup and the seat-lease read). It found #662's per-agent adoption try/catch and no decision on bounding the reason.
- Own-assignment sweep: 19 open; none on lifecycle failure reasons.

Origin Session ID: 6f7d14a3-e126-4b47-888f-fc28c748ae83

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-02T17:12:43Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-02T17:12:43Z @neo-opus-ada added the `bug` label
- 2026-10-02T17:12:43Z @neo-opus-ada added the `ai` label
- 2026-10-02T17:12:43Z @neo-opus-ada added the `agent-os` label
- 2026-10-02T17:17:42Z @neo-fable-clio cross-referenced by #773
- 2026-10-02T17:18:30Z @neo-opus-ada cross-referenced by PR #774
- 2026-10-02T17:38:53Z @tobiu referenced in commit `30c3b44` - "fix(fleet): a lifecycle failure reaches the roster as a refusal's words or a code, never its message (#772) (#774)

The Codex Desktop helper cleanup and the seat-lease read copied a caught message into failureReason,
which FleetManager copies into roster rows: an internal caller name, a raw "kill EPERM", or a path
that a deriveAgentInstanceHome refusal embeds. One helper now carries a named caller's refusal
without its prefix, else the system code, else bounded words while the Fleet log keeps the cause.
writeSeatLease uses it with unchanged output."
- 2026-10-02T17:38:53Z @tobiu closed this issue

