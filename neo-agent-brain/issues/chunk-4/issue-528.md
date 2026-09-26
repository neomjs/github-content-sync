---
id: 528
title: The OpenCode wake plant drops the seat identity its reader requires
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-25T22:34:14Z'
updatedAt: '2026-09-25T22:55:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/528'
author: neo-opus-ada
commentsCount: 1
parentIssue: 503
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
# The OpenCode wake plant drops the seat identity its reader requires

## Context

This is #503's half 1, split out so #503 keeps AC-4 (`who_is_online`) and stays the tracker. Half 2 shipped as #512 / PR #510. @neo-preview handed half 1 over on 2026-09-25 (#503's comments).

An OpenCode seat's wakes travel the `opencode-server` route. The receiver's adapter reads the **seat envelope** (not the wake *digest* envelope, which is a different artifact) and refuses it without `agentIdentity`: `localWakeAdapters.mjs` `readOpenCodeEnvelope` throws "opencode-server envelope requires 'agentIdentity'". `assertOpenCodeEnvelopeOwner` then compares that identity with the route's. That refusal is what @neo-kimi-phoebe's route has returned for 307 of 348 dispatches (#513), and what @neo-preview's returned before a hand heal.

## The Problem

The envelope has two writers, and they disagree. `ai/daemons/wake/daemon.mjs` (~:1139) says: "Two producers, one contract … both write the IDENTICAL shape … so last-writer-wins is a no-op; a third producer must update this contract, never drift it."
- **Producer 1**, the seat-config boot hook `generateOpenCodeSeatConfig.mjs` emits (`write-wake-envelope.mjs`, ~:277–312), stamps `agentIdentity` from `NEO_AGENT_IDENTITY` and refuses to run without it.
- **Producer 2**, the plant `opencodeWakeEnvelopePlugin.mjs` (`writeEnvelope`, ~:111), writes `{hostname, port, sessionId, projectId, directory, username, password, updatedAt}`, with no `agentIdentity`. Its docblock documents the same eight-field shape.

Whenever the plant writes last (`session.created`, a restored session's first update), it replaces a good envelope with one the reader refuses. It does so silently. That is the expiry @neo-preview names on their own hand-healed envelope.

**Correction to #503's reframed AC-1.** That AC names `consumeWakeOutbox.mjs`'s `readOwnerAuthority` as the reader. That module is the **`kimi-pull-bridge`** consumer (its usage names `~/.kimi-code/wake-outbox.jsonl`), and its `pid`/`pidStartedAt` legs belong to that route. The OpenCode seat envelope's reader is `readOpenCodeEnvelope`, which has no owner-epoch leg. So the "which process is `pid`" question raised on #503 does not arise for this envelope. The JSDoc should say so, so the two envelopes stop being confused.

## The Architectural Reality

- `ai/daemons/wake/localWakeAdapters.mjs`: `readOpenCodeEnvelope` validates `agentIdentity, hostname, sessionId, projectId, directory, username, password` (non-empty strings), `port`, a loopback host and an absolute directory. `assertOpenCodeEnvelopeOwner` refuses a foreign owner.
- `ai/services/fleet/opencodeWakeEnvelopePlugin.mjs` is a **plant**, copied to `~/.config/opencode/plugins/` and run inside OpenCode, outside this repo. It must stay import-free. It already reads its credential pair from `process.env` (`OPENCODE_SERVER_USERNAME` / `_PASSWORD`).
- A seat's identity reaches its env through its launcher: the seat wrapper sources the seat's `.env`, and `generateOpenCodeSeatConfig.mjs` names `NEO_AGENT_IDENTITY` among its keys. A Dock launch bypasses the wrapper (the class #223 fixed for the shell), so the plant can find no identity in its env.
- `generateOpenCodeSeatConfig.mjs` (~:306) already normalizes a bare handle to `@handle` before stamping.

## The Fix

1. **The plant stamps the seat's identity**, in the same canonical `@` form as producer 1:
   - it takes `NEO_AGENT_IDENTITY` first;
   - else it carries over the identity of the envelope it replaces, which covers a Dock launch after a heal or an earlier wrapper launch;
   - else it **does not write**. It logs why and leaves the existing envelope in place, since a write the reader must refuse is worse than none.
2. **The contract is declared once.** `localWakeAdapters.mjs` exports the `opencode-server` seat envelope's required fields (`OPENCODE_SEAT_ENVELOPE_FIELDS`), and `readOpenCodeEnvelope` validates against it. The plant can't import it, so a spec binds them.
3. **Specs, red-first:** the plant's real `writeEnvelope` (a fake OpenCode context, a temp `XDG_DATA_HOME`) and producer 1's generated script both emit envelopes that pass the **real** `readOpenCodeEnvelope` and `assertOpenCodeEnvelopeOwner`. Arms cover the carry-over and the refusal.
4. **JSDoc:** the daemon's "two producers, one contract" names the declaration. The plant's docblock shows the current shape, and says this envelope carries no owner epoch, unlike the `kimi-pull-bridge` one.

## Acceptance Criteria

- [ ] AC-1: The plant's emitted envelope passes the real `readOpenCodeEnvelope` and `assertOpenCodeEnvelopeOwner` for its seat. Red-first: on `dev` it fails on `agentIdentity`.
- [ ] AC-2: With no identity in the env, the plant carries over the replaced envelope's identity. With none there either, it writes nothing and logs why.
- [ ] AC-3: Producer 1's generated script passes the same readers, and both producers' shapes are pinned to the one exported field list.
- [ ] AC-4 (post-merge, a live OpenCode seat): after the plant rewrites the envelope, a dispatch is recorded `delivered` by the receiver. This is #503's AC-2.

## Out of Scope

- The `kimi-pull-bridge` envelope writer (@neo-kimi-iris's route, #514).
- Healing the benched seats' envelopes (#513, #514).
- `who_is_online` (#503 AC-4).
- A FleetLifecycle-launched seat's env. `deriveHarnessLaunchSpec` does not receive the seat identity; that's a follow-up if FM launches OpenCode seats.

## Related

Parent #503. #512 / #510 (half 2), #513 / #514 (the live cost), #19 (the owner guard, closed tonight), neomjs/neo-agent-institution#223 (the same launch-env class for the shell).

Live latest-open sweep: the latest 20 open neo-agent-brain issues at 2026-09-25T22:33:34Z, plus `gh search issues --owner neomjs "opencode envelope agentIdentity"`; only #503/#513/#514 match, and this is #503's half 1 by the author's handoff. A2A: @neo-preview's handoff 22:31Z, no other claim. MC sweep: "OpenCode wake envelope plugin writes no agentIdentity…", 5 results. The prior art is the 08-01 Dock-launch env gap (@neo-kimi-phoebe) and the 08-23 writer/reader path pairing (mine); no prior decision on the plant's identity. Own-assignment sweep: none overlapping.

Origin Session ID: 0f80515e-7682-4313-8101-b926da48c55c


## Timeline

- 2026-09-25T22:34:15Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-25T22:34:15Z @neo-opus-ada added the `bug` label
- 2026-09-25T22:34:15Z @neo-opus-ada added the `ai` label
- 2026-09-25T22:34:15Z @neo-opus-ada added the `agent-os` label
- 2026-09-25T22:34:22Z @neo-opus-ada added parent issue #503
- 2026-09-25T22:39:55Z @neo-opus-ada cross-referenced by PR #529
- 2026-09-25T22:46:40Z @neo-opus-ada referenced in commit `7d7de1c` - "fix(wake): the plant reads its launch identity in a module-level helper (#528)

The config SSOT lint walks an exported arrow's whole body, so the factory's
lazy NEO_AGENT_IDENTITY read counted as a C1 competing resolver. As module
functions (the Kimi hook's readAgentIdentity shape) the read is what it is:
a use-site read in a process that has no AiConfig."
### @neo-opus-ada - 2026-09-25T22:55:11Z

**Handover (session sunset, 2026-09-25 22:5xZ).** Owner: @neo-opus-ada.

**State:** PR #529 is this ticket's PR. Head `7d7de1c`, CI 8/8, CLEAN. @neo-preview is requested (they own #503) and was sent an A2A.
- `ae048e7`: the OpenCode plant stamps `agentIdentity` in `@handle` form. It reads `NEO_AGENT_IDENTITY` first, else carries over the identity of the envelope it replaces, and with neither it writes nothing and logs why. `OPENCODE_SEAT_ENVELOPE_FIELDS` declares the reader's contract. `opencodeSeatEnvelopeParity.spec.mjs` runs both producers through the real `dispatchLocalWake`. Three wake specs join `brain-unit.yml`.
- `7d7de1c`: the env read moves into a module-level helper, the shape of the Kimi hook's `readAgentIdentity`. The config SSOT lint's C1 detector walks an exported arrow's whole body, so the lazy read inside the plugin factory had counted as a competing resolver. The same read inside an exported function declaration passes. Defect-note `b2e12b1f195435b9` records the probe.

**Evidence:** red-first, all 4 plant arms failed on `dev`. The 3 wake specs pass 56/56 with the identity env unset and set. The SSOT lint reports 0 C1, and archaeology reports 0 violations.

**Pickup:**
1. If there is a review round, address it on `ada/528-plant-identity` (Brain worktree `wt-brain-dev`), then re-request the reviewer and send a delivered wake. On approval, hand off to @tobiu for the merge.
2. After the merge, **re-plant first**. Copy `ai/services/fleet/opencodeWakeEnvelopePlugin.mjs` to `~/.config/opencode/plugins/neo-wake-envelope.mjs` on each OpenCode seat (@neo-preview, @neo-kimi-phoebe). Nothing copies it automatically, and the old plant keeps writing envelopes with no identity.
3. Restart the seat or open a new top-level session. The next dispatch should be recorded `delivered`: this is the PR's post-merge box, #503's AC-2, and the cure #513 needs.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


