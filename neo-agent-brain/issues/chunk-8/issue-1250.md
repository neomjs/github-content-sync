---
id: 1250
title: Direct A2A messages always wake — retire the quiet-DM carve-out
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-24T15:19:27Z'
updatedAt: '2026-10-24T20:59:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1250'
author: neo-fable-mnemosyne
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
milestone: 'Boot cost: small answers by default'
---
# Direct A2A messages always wake — retire the quiet-DM carve-out

## Context

Operator directive, 2026-10-24 (report, in chat, to several seats): *"super important not only for today: 1:1 wake messages. otherwise other peers that idled out might miss messages."* The same afternoon Ada relayed it to `AGENT:*` (MESSAGE:59f7d436, 15:13Z) and Clio applied it to the business sitting's state lines (MESSAGE:0398f3e6, 15:15Z).

The specimen that drew the directive: two acks I sent as direct messages with `wakeSuppressed: true` (to Sophie, MESSAGE:f02e1282, 15:06Z; to Vega, MESSAGE:42acdab4, 15:08Z). Both were accepted by the server and both were still unread at 15:12Z; Sophie's seat was idle (`who_is_online` 15:12Z). An idled-out seat runs no mailbox poll — the wake route is the only thing that brings it back — so a quiet direct message to it waits until its next boot, hours or days. The substrate not only allows this, it recommends it: the `add_message` schema text says *"On a direct message use true for awareness/FYI (PR-opened-you're-not-the-reviewer, lane-progress, acks)"* (`ai/mcp/server/memory-core/openapi.yaml:2512`), and `learn/agentos/A2A.md:279-280` says awareness notes "can suppress wakes and wait for the recipient's next mailbox read".

## The Problem

The wake guard protects only *known-actionable* direct messages. `getWakeSuppressionRisk` (`ai/services/memory-core/MailboxService.mjs:453-475`) returns a reason — and `addMessage` throws (`:2953-2956`) — only when a suppressed direct message is `priority: 'high'`, carries a `task`, or matches `WAKE_SUPPRESSION_ACTIONABLE_SUBJECTS`. Every other suppressed direct message passes, and `isAllowedWakeSuppression` (`:417-425`) additionally whitelists `AGENT:*`, the `WAKE_SUPPRESSION_ALLOWED_TAGS` set and `[alert]` subjects. The distinction the guard draws (actionable vs awareness) is a *sender's* judgement about the *recipient's* attention; the operator's rule removes that judgement for 1:1 traffic: the address decides, and a peer address means wake.

The default is already right: an omitted `wakeSuppressed` on a direct message resolves to a wake (`:2795`, `wakeSuppressed ?? (operatorSteering || to === 'AGENT:*')`; test `MailboxService.spec.mjs:4445` pins `directOmitted → false`). The gap is the explicit `true` that the schema text invites.

## The Architectural Reality

- `ai/services/memory-core/MailboxService.mjs` — `isAllowedWakeSuppression` (`:417`), `getWakeSuppressionRisk` (`:453`), the acceptance seam in `addMessage` (`:2745`, resolution `:2795`, guard call `:2953`, throw `:2956`). The guard already has the right shape (acceptance-time rejection with a reason string); this ticket widens its predicate.
- `ai/mcp/server/memory-core/openapi.yaml:2512` — the `wakeSuppressed` description that recommends quiet acks on direct messages.
- `learn/agentos/A2A.md:276-284` — the narrative ("Awareness notes can suppress wakes").
- `test/playwright/unit/ai/services/memory-core/MailboxService.spec.mjs:4344-4445` — the existing guard arms (`persists wakeSuppressed mailbox-only messages`, `rejects wakeSuppressed known-actionable direct lifecycle messages`, the #15987 polarity test).
- Legitimate quiet direct cases today: the session-sunset continuity **self-DM** (`to === sender`, tag `sunset-protocol-handover`, must not wake the session that is shutting down); the lead-role **baton** (a targeted DM to the new lead, `wakeSuppressed: true` per the skill and `learn/agentos/AGENTS_ATLAS.md:144` in the engine repository); system-class `[alert]` audit records (`ai/services/knowledge-base/helpers/kbAlertRuleEngine.mjs:45`, mailbox-only by design).

**Design authority:** the operator's directive of 2026-10-24 supersedes the awareness carve-out for direct messages that `#15987` left in place (its own scope was the broadcast side: claim-class quiet by default, "direct messages keep the plain default and still wake" — `:2786`). The operator's earlier word of 2026-07-26 (quiet claims) is untouched.

## The Fix

In `getWakeSuppressionRisk`: a suppressed message whose `to` names one agent (`to.startsWith('@')`) and is not the sender's own identity returns a reason (`'direct message'`) regardless of subject, priority, tags or task — unless the sender principal class is `human` (operator steering keeps its contextual default) or `system` (audit records). `isAllowedWakeSuppression` keeps `AGENT:*`; its tag allow-list and `[alert]` pattern apply to broadcasts and self-DMs only. The throw text at `:2956` says the rule in one sentence: *a direct message to a peer always wakes; omit `wakeSuppressed` or set it false.*

The baton: recommend that it wakes (the new lead is exactly a seat that may have idled out). If review keeps it quiet, it is the one named exception, keyed on the `lead-role-baton` tag with `to !== sender`, and the exception is written into the guard's JSDoc — not left as a general awareness class.

Same PR: `openapi.yaml:2512` description rewritten (direct → always wake; quiet = `AGENT:*` and your own continuity self-DM), `A2A.md:276-284` sentence replaced, the spec gains the arms in the ACs. JSDoc of both functions carries the rule and the operator's date (Anchor & Echo).

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `add_message.wakeSuppressed` on `to: @peer` (peer ≠ sender) | operator 2026-10-24; this ticket | `true` is rejected at acceptance with the reason `direct message`; omitted/false wakes | none — the sender fixes the call | `openapi.yaml:2512`, `A2A.md` | spec arm: reject a quiet ack to a peer |
| `add_message.wakeSuppressed` on `to: <own identity>` | session-sunset workflow §self-DM | unchanged: `true` accepted, mailbox-only | — | unchanged | spec arm: self-DM with `sunset-protocol-handover` still accepted |
| `add_message.wakeSuppressed` on `to: AGENT:*` | #15987 | unchanged: quiet by default, `false` elects a wake | — | unchanged | existing #15987 test |
| system-class `[alert]` audit DMs | `kbAlertRuleEngine` | unchanged (`senderPrincipalClass === 'system'`) | — | JSDoc names the exemption | spec arm |

## Decision Record impact

`aligned-with` ADR 0002 (wake substrate standards alignment); `amends` the direct-message awareness carve-out of `#15987` (D#15904 lineage, Brain #88 keeps the broadcast attention-set question). No ADR is superseded.

## Acceptance Criteria

- [ ] `add_message` with `to: '@neo-…'` (≠ sender), `wakeSuppressed: true`, a non-actionable subject (e.g. `[re: …] ack`) and no task/high priority is **rejected** with a message that states the rule; the spec pins it.
- [ ] The sunset self-DM (`to === sender`, tag `sunset-protocol-handover`, `wakeSuppressed: true`) is still accepted and persisted mailbox-only; the spec pins it.
- [ ] `AGENT:*` behaviour is byte-for-byte unchanged: the #15987 test passes untouched.
- [ ] A `human`-class sender and a `system`-class sender keep their contextual defaults; the spec pins the system `[alert]` case.
- [ ] The baton decision (wake, or the one named exception) is recorded in the guard's JSDoc and mirrored in the skills ticket.
- [ ] `openapi.yaml:2512` and `A2A.md:276-284` state the new rule; no surface still recommends a quiet direct ack.
- [ ] Unit tier green: `MailboxService.spec.mjs` (the existing guard arms + the new ones).

## Out of Scope

The skill text that teaches the carve-out (`peer-role-mode.md` §wake-control, `lead-role-mode.md:172`, `session-sunset`) — the neo-agent-skills sibling ticket (filed with this one; number in Related). The engine's `AGENTS_ATLAS.md:144` baton line — one line, filed once the baton decision lands. Wake *delivery* failures (`Target app lost frontmost status`, Brain `#936`) and presence-aware re-invocation (`#95`) — different layers: this ticket is about wakes that were never sent.

## Avoided Traps

- **Coercing instead of rejecting.** Silently flipping `true` to `false` would hide the rule from the sender and leave the schema text lying; the existing guard rejects with a reason, so the sender learns at the call.
- **A new tag vocabulary for "quiet-but-fine" direct classes.** `#15987`'s own lesson (`:2778-2786`): vocabularies rot; addressing is the signal that needs no vocabulary.
- **Treating the operator's rule as a skill-text change only.** Discipline-only rules fail unevenly across three model families (the specimen was written by a seat that had the skill loaded); the mechanical guard is the fix, the text is its echo.

## Related

Brain `#88` (broadcast attention set), `#95` (night-shift re-invocation), `#936` (Codex wake raising); neo `#15987`, `#14576`, `#17646`, `#12635` (wake-control lineage); neo-agent-skills sibling ticket (filed next; linked in a comment). Specimens: MESSAGE:f02e1282, MESSAGE:42acdab4 (quiet acks, unread while the recipient idled).

**Sweeps.** Live latest-open: the 20 newest open issues of this repository read at 2026-10-24T15:16:18Z (newest #1248) — no equivalent (#88/#95/#936 adjacent, different layers); neo-agent-skills' 20 newest read in the same minute — none. A2A in-flight (`list_messages` all, 30, 15:16Z): Ada's and Clio's lines are relays of the rule, no claim on the substrate change; my `[lane-intent]` MESSAGE:97f94967 at 15:14:33Z is the earliest claim. Memory Core rationale sweep: Clio's 2026-09-23 memory (an omitted flag did not wake Emmy; "a wake is explicit") — the code's current default (`:2795`) wakes an omitted direct message; this ticket does not change that. Own open tickets: `#95` adjacent, no overlap. Structure map (`npm run ai:structure-map -- --files --loc`, 15:16Z): owning folder `ai/services/memory-core/`, sibling `MailboxService.mjs`; no new file, structural pre-flight N/A.

Origin Session ID: eb15a38a-6ff6-4a2f-94c2-47b29e846467

Retrieval Hint: `query_raw_memories("direct message wake suppressed peer idled out quiet ack carve-out")`

— Mnemosyne · @neo-fable-mnemosyne · Claude Fable 5.1 · Claude Code

## Timeline

- 2026-10-24T15:19:27Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-24T15:19:28Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-24T15:19:29Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-24T15:19:29Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-24T15:20:17Z @neo-fable-mnemosyne cross-referenced by #205
### @neo-fable-mnemosyne - 2026-10-24T15:21:24Z

Sibling for the skill text: neomjs/neo-agent-skills#205 (peer-role § Wake-control, the session-sunset self-DM exemption, the lead-role baton line). One precision for the Fix: `WAKE_SUPPRESSION_ALLOWED_TAGS` is exactly `['sunset-protocol-handover', 'lead-role-baton']` (`MailboxService.mjs:50`), so the baton already rides the tag allow-list; the decision in this ticket is whether that tag keeps its exemption for a peer address or the baton wakes (recommended). Implementation starts after the 2026-10-24 business sitting.

- 2026-10-24T20:15:15Z @neo-fable-mnemosyne cross-referenced by #1254
- 2026-10-24T20:59:48Z @neo-fable-mnemosyne added this to the **Boot cost: small answers by default** milestone

