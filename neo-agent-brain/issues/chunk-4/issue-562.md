---
id: 562
title: 'Claude seats receive wakes through a session hook, not osascript'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-26T22:05:34Z'
updatedAt: '2026-09-26T22:05:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/562'
author: neo-opus-ada
commentsCount: 0
parentIssue: 30
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
# Claude seats receive wakes through a session hook, not osascript

## Context

**Operator, 2026-09-26 ~21:0x–21:45Z**, on #30:
- *"we still use the osascript adapter for delivery… we created tickets. and i have no clue what happened to them. it is driving me NUTS that it did not get changed."*
- On the Claude half: *"this has to be codified. and must work for FM messages too => interface to agent OS. flipping ONE live session is pointless. must work for all future sessions."*
- *"each peer has an own harness instance."*

Measured that evening, recorded on #30:
- **7 of 10 routes still arm `osascript`** (ada, grace, vega, fable, fable-clio, gpt, gpt-emmy); the route manifest was last written Aug 24.
- **The merged courier (#241) was never run.** Its resolver keys a seat by cwd and sends by session title. Neither is identity: three live sessions sit in one instance's cwd, and three seats share one title.
- **The Codex adapter is not a wake.** `send-message-v2` starts a new chat instead of delivering into the seat's thread (@neo-gpt's gate on #30). That half gets its own leaf.

This leaf is the Claude half of #30, reshaped to the ruling: codified in the Agent OS, provisioned to every seat, for every future session.

## The Problem

- **Delivery drives the UI.** A wake activates the seat's app, focuses the prompt field and types. The operator owns focus and is using the machine (#30's 2026-08-16 correction), so a delivery that must hold frontmost fails or types into the wrong window.
- **Relaying through `SendMessage` needs an address the platform does not give.** The courier needs an always-on relay session plus a unique target name. Names are session titles, and the `ListAgents` `[ref]` that disambiguates them joins to no registry field.
- **Push never needs to reach the window.** Everything a pull needs already exists:
  - The seat is bound to its harness instance at every session start.
  - Hooks are provisioned to every seat.
  - Memory Core serves a per-subscription digest.
- **Claude Code documents a background hook that can wake the model.** A command hook with `asyncRewake: true` runs in the background; exit code 2 wakes Claude with the hook's stderr (or stdout) as a system reminder. Hook edits reach a running session through the file watcher, and async hooks have no enforced timeout ([hooks reference](https://code.claude.com/docs/en/hooks)).
- **Whether that wakes an *idle* Desktop session is undocumented.** AC-1 measures it before anything else is built on it.

## The Architectural Reality

- **Hook wiring.** `ai/scripts/lifecycle/hooks/claude/events.manifest.json` is the Agent-OS-owned Claude event wiring. `ai/scripts/lifecycle/hooks/projectSeatHooks.mjs` reconciles it into each seat's `.claude/settings.json`, which is projected and untracked. Today: `SessionStart` → `wakeArmingHook` plus `seatProjectionCheck`; `UserPromptSubmit`/`PostToolUse` → `turnPresenceHook`; `Stop` → `laneStateStopHook`.
- **Instance binding.** `wakeArmingHook` → `ai/daemons/wake/armSeatWakeRoute.mjs` arms the seat's route at `SessionStart` from the live instance tuple; the route's `harnessTargetMetadata.userDataDir` is the harness instance. The Claude leg is the only harness with arming (#79).
- **The pull path.** `WakeSubscriptionService#pollDigest({subscriptionId, sinceLogId})`, exposed as `manage_wake_subscription` `poll-digest`. #561 reports it hanging on one seat.
- **Hook input.** A hook's stdin carries `session_id`, `cwd`, `transcript_path` and `hook_event_name`. A session's process ancestry ends at its instance's main process (`--user-data-dir`, absent for the default instance).
- **Several live sessions can share one instance** (three in `@neo-opus-ada`'s on 2026-09-26).

## The Fix

1. **A listener hook,** `ai/scripts/lifecycle/hooks/claude/wakeListenerHook.mjs`, registered in `events.manifest.json` on `SessionStart` and `Stop` with `asyncRewake: true`.
   - It resolves the seat's subscription the way `wakeArmingHook` does.
   - It pulls the seat's digest past its watermark.
   - On a digest, it writes the digest and exits 2, so Claude is re-invoked with it. Otherwise it keeps pulling.
2. **One listener per seat.** The newest session of the instance owns it: a listener armed by a newer session supersedes an older session's, which exits 0. Re-arming at `Stop` in the owning session is idempotent.
3. **The arming hook arms Claude routes for pull,** so the receiver stops driving the UI for that seat. Rollback is re-arming `osascript`.
4. **Fleet Manager messages need nothing extra.** They are A2A messages, so the seat's digest carries them.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `wakeListenerHook.mjs` (new) | this ticket; operator ruling on #30 | Pulls the seat's digest; exit 2 with the digest wakes the session; one owner per seat, newest session wins | Unconfigured plane: a named skip, like `wakeArmingHook`. Superseded: exit 0. Pull failure: backs off and retries, never exits 2 without a digest | JSDoc | AC-2, AC-3 |
| `events.manifest.json` `SessionStart`/`Stop` entries (existing file) | `projectSeatHooks.mjs` contract | Adds the listener with `asyncRewake: true` | A seat not yet re-projected keeps its current wiring | manifest `$comment` | AC-4 |
| The Claude route's adapter as armed by `armSeatWakeRoute` (existing) | #30, ADR 0002 | Pull, no UI driving, once the listener is provisioned | Rollback: re-arm `osascript` | JSDoc | AC-5 |

## Decision Record impact

`aligned-with ADR 0002`. Its bridge daemon is the fallback for harnesses without a native primitive, and Claude Code now has one: the `asyncRewake` hook. It is a pull that the hook drives, not one of the ADR's push rungs (MCP notifications, A2A webhooks), so Claude seats leave the fallback without claiming a rung. ADR 0014 keeps the receiver as `host-edge`; this changes what it does for Claude seats, not where it runs. Implementation reads ADR 0019 before any config touch (cadence, flags).

## Acceptance Criteria

- [ ] AC-1 (live, first): on one seat, with the operator present, an `asyncRewake` hook exiting 2 re-invokes an **idle** Claude Desktop session. If it does not, this leaf stops and reports; nothing below is built on an unproven transport. The hook is provisioned through `projectSeatHooks`, or installed by the operator; an agent editing its own harness hooks is refused as self-modification.
- [ ] AC-2 (unit): the listener exits 2 carrying the digest text exactly when the seat's digest has events past its watermark. It exits 0 when superseded, and a second `Stop` in the owning session arms nothing new.
- [ ] AC-3 (unit): across sessions of one instance, the newest session's listener owns the seat and an older one exits on supersession.
- [ ] AC-4 (unit): the manifest carries the listener on `SessionStart` and `Stop` with `asyncRewake: true`, and `projectSeatHooks` projects it.
- [ ] AC-5 (live): with the route armed for pull, a wake to the seat arrives in its newest session with no focus change and no keystroke, and the receiver's record shows no `osascript` dispatch.
- [ ] AC-6 (live): a message sent from the Fleet Manager wakes the seat the same way.

## Out of Scope

- Codex seats: existing-thread plus instance targeting, per @neo-gpt's gate on #30. That is the next leaf.
- OpenCode and Kimi seats (#532, #513, #514), and arming parity (#79).
- The `poll-digest` hang (#561). This leaf depends on a working pull: if AC-1's probe reproduces the hang on a Claude seat, #561 blocks it.
- Retiring the courier code, which is #30's close-out.

## Avoided Traps

- **A courier session relaying through `SendMessage`:** an always-on relay and an address the platform does not expose.
- **Channels:** a research preview that needs `--channels` at launch; Desktop sessions are not launched by us.
- **Per-seat hand flips of live routes:** the operator ruled them pointless.
- **An agent installing its own hook:** refused by the harness classifier. Provisioning belongs to `projectSeatHooks`.

## Related

#30 (parent) · #241 (the courier drain) · #79 (arming parity) · #561 (`poll-digest`) · #532 / #548 (plant provisioning, the OpenCode analogue) · #50 (pull over the ingress for remote clients) · #129 (outbound wake-stream)

Live latest-open sweep: the latest 20 open issues at 2026-09-26T22:05:19Z, no equivalent (adjacent: #561, #549, #547, #532). A2A in-flight sweep (all read states, last 60 min): no claim on this scope beyond my own #30 lane. Memory Core sweep ("wake delivery steals focus … pull own wakes"): the April–August osascript history, including the June finding that per-instance `--user-data-dir` already isolates the seats. Own-assignment sweep: #30 (the parent). KB ticket sweep: #19 (closed, OpenCode envelopes), #79, #503, #30. Structure map: `ai/scripts/lifecycle/hooks/claude` (sibling precedent: `wakeArmingHook`, `laneStateStopHook`).

Origin Session ID: 40558e40-aee4-4461-946d-a389266a3256
Retrieval Hint: `query_raw_memories("Claude seat wake pull asyncRewake listener hook harness instance osascript focus")`


## Timeline

- 2026-09-26T22:05:34Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-26T22:05:35Z @neo-opus-ada added the `enhancement` label
- 2026-09-26T22:05:35Z @neo-opus-ada added the `ai` label
- 2026-09-26T22:05:35Z @neo-opus-ada added the `architecture` label
- 2026-09-26T22:05:36Z @neo-opus-ada added the `agent-os` label
- 2026-09-26T22:05:41Z @neo-opus-ada added parent issue #30

