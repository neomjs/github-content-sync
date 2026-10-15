---
id: 1035
title: Restore automatic wakes for Fleet-launched Antigravity
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-pro
createdAt: '2026-10-14T23:42:08Z'
updatedAt: '2026-10-15T12:58:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1035'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-15T12:58:43Z'
---
# Restore automatic wakes for Fleet-launched Antigravity

## Context
The first-boot work in #1017 explicitly deferred exact-instance wake binding and directed its follow-up to reuse the existing wake transport. This leaf completes that integration for Fleet-launched Antigravity seats.

## The Problem
At `e219267e5b`, `armFleetSeatWake` has only a `codex-desktop` dispatch entry and returns `null` for Antigravity. `deriveHarnessWakeAddress` also excludes Antigravity. A read-only invocation with an Antigravity definition returned `armResult: null`, `address: null`, and zero client/publisher calls. Start can therefore succeed without attempting to publish that seat's wake route.

This proves the absence of Fleet's GUI-route arming. It does not establish which transport Antigravity used historically or whether native MCP push is supported by the current client.

## The Architectural Reality
- `ai/services/fleet/armFleetSeatWake.mjs:38-40,74-78` owns subscription and host-manifest publication using the launched seat's proven identity and profile.
- `deriveHarnessLaunchSpec.mjs:426-430` owns the wake address beside the actual launch derivation.
- `FleetManager.mjs:501-526` records arming only when the caller returns a result.
- `ai/daemons/wake/localWakeAdapters.mjs` already accepts Antigravity with an explicit instance address. `hostHarnessMetadata.mjs` supplies its historical `shift+i` shortcut; that gesture needs current-client validation, not an assumption.

Historical transport: [Engine PR #10401](https://github.com/neomjs/neo/pull/10401), merged 2026-04-27, implemented raw `mcp-notifications` wake delivery for Antigravity. [PR #10392](https://github.com/neomjs/neo/pull/10392) also implemented an osascript path. ADR 0002 §5 and §6.3.3 prefer MCP notifications, then existing webhooks, with a daemon fallback where needed. Today's adapter table does not identify the seat's historical or current selected route.

Design authority: ADR 0002's native-first selection and current-client evidence govern the transport. #79 establishes “The Fleet arms each GUI seat it launches, acting as that seat, right after the start returns.” Its Antigravity exclusion depended on having no Fleet GUI launch. #1017 now supplies that launch and names wake binding as a follow-up.

## The Fix
First read the seat-owned subscription and verify the current client's behavior on the existing MCP-notification path. Prefer that path when it demonstrably starts the intended session; otherwise use an existing supported receiver adapter bound to the profile actually launched. Do not choose osascript merely because a dispatch entry exists.

Make the selected path durable across managed Starts using its existing subscription, identity-proof, cancellation and delivery owners. If the GUI receiver is needed, extend the Fleet address/arming helpers and validate its current focus/submit gesture. Preserve explicit nonfatal refusal when delivery cannot be established.

## Contract Ledger
| Surface | Authority | Behavior / fallback | Docs / evidence |
|---|---|---|---|
| Antigravity registration | ADR 0002; managed Start and selected delivery owner | Arm the proven seat's supported route. The receiver fallback uses the actual launched profile. Refusal stays nonfatal and explicit | Existing owner JSDoc; composed controls and native capability witness |
| `status().wakeRoute` where Fleet owns receiver arming | Existing lifecycle launch identity | Ready only for the route actually published; canceled or old Starts cannot publish success for a newer Start | Existing status contract; cancellation and stale-result controls |
| Native delivery | Existing MCP notification or receiver/adapter owner | Reach the intended conversation without corrupting a draft; the GUI fallback retains its focus/target guards | Selected transport controls plus native recipient witness |

## Acceptance Criteria
- [ ] A managed Antigravity Start on the attached plane arms a verified existing wake path for its own unfiltered `SENT_TO_ME` subscription. If the receiver path is selected, publish its route for the actual launched profile. Repeated Starts do not duplicate routes or modify another seat's route.
- [ ] Missing receiver where required, unavailable/unproved credentials or target, cancellation and stale completion retain named nonfatal refusal behavior. Other harnesses retain their existing wake paths.
- [ ] Native MCP wake support is evaluated before choosing a fallback. If GUI delivery is needed, validate its current focus/submit gesture, draft preservation and wrong-instance refusal. No exact vendor-build allowlist is added.
- [ ] **[L4-deferred — installed handoff]** A normal-priority A2A wake reaches a freshly Fleet-started native Antigravity conversation and starts a turn, with a transport-specific receipt and recipient witness. An emitted notification, manifest entry or adapter exit alone is insufficient. Residual owner: #571.

## Out of Scope
New wake daemons/transports, subscription credential copying, general hook implementation, unrelated naming/MCP setup, and changes to delivery cadence or other families.

## Avoided Traps
Active subscription is not host-route arming; delivery-service liveness is not recipient delivery. Do not reuse an old IDE shortcut without checking the current app, or test against an operator's unsent draft.

## Related and sweeps
#571 (parent acceptance); #1017 (first boot); #79 (Fleet arming precedent); #135 (broader adapter design, not a prerequisite); #936 (separate Codex activation case).
Live latest-open and recent A2A sweeps checked before creation. Keyword/all-state reads found no equivalent Antigravity Fleet-arm leaf. MC searches for Antigravity wake delivery and Gemini wake subscriptions returned older routing records, no contrary ruling; KB returned #79 and the separate Claude pull-route #768. Own-assignment sweep: six open, with #936 the only wake-specific item. Scoped Brain structure map: `ai/services/fleet`, 121 files; existing helpers above own this change, no new module required.
Decision Record impact: aligned-with ADR 0002's native-first transport selection and the existing Fleet arming contract; no new transport architecture.
unowned-rationale: bounded follow-up capture; no source implementation or live route activation is claimed in this filing.
Retrieval Hint: "Antigravity Fleet Start active subscription no armed route"

Authored by Emmy · `@neo-gpt-emmy`.



## Timeline

- 2026-10-14T23:42:09Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-14T23:42:10Z @neo-gpt-emmy added the `ai` label
- 2026-10-14T23:42:10Z @neo-gpt-emmy added the `model-experience` label
- 2026-10-14T23:42:10Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-14T23:42:30Z @neo-gpt-emmy added parent issue #571
- 2026-10-14T23:52:56Z @neo-gpt-emmy changed title from **Arm Antigravity wake routes from Fleet Start** to **Restore automatic wakes for Fleet-launched Antigravity**
- 2026-10-15T09:38:39Z @neo-gemini-pro assigned to @neo-gemini-pro
- 2026-10-15T10:08:52Z @neo-gemini-pro cross-referenced by PR #1044
- 2026-10-15T10:18:40Z @neo-gemini-pro referenced in commit `cb356ad` - "feat(fleet): update wake daemon Antigravity shortcut and spec parity (#1035)"
- 2026-10-15T11:44:09Z @neo-gemini-pro referenced in commit `2ee4ed1` - "fix(wake): dismiss Antigravity autocomplete with Space without blurring composer (#1035)"
- 2026-10-15T12:58:43Z @tobiu referenced in commit `feb2965` - "feat(fleet): restore automatic wakes for Fleet-launched Antigravity (#1035) (#1044)

* feat(fleet): restore automatic wakes for Fleet-launched Antigravity (#1035)

* feat(fleet): update wake daemon Antigravity shortcut and spec parity (#1035)

* fix(wake): dismiss Antigravity autocomplete with Space without blurring composer (#1035)"
- 2026-10-15T12:58:43Z @tobiu closed this issue

