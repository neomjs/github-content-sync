---
id: 766
title: 'Seat hooks resolve no plane: they read Fleet-transport leaves'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-02T16:45:32Z'
updatedAt: '2026-10-02T17:48:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/766'
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
blocking:
  - '[ ] 768 The Fleet stops arming claude-desktop on osascript once a Fleet-launched Claude seat arms itself at SessionStart'
closedAt: '2026-10-02T17:48:34Z'
---
# Seat hooks resolve no plane: they read Fleet-transport leaves

## Context

- **Measured 2026-10-02**, on `@neo-opus-ada`'s operator-launched Claude seat:
  - 41 session transcripts carry `wakeArmingHook`'s `[wake-arming] seat is UNARMED`.
  - 99 of those reports name `fleet.planeBase is not configured`. None reports armed.
- **The same skip on `@neo-fable`'s seat, 2026-09-04:** running the projected `wakeArmingHook` and `turnPresenceHook start` by hand skipped both on an unconfigured plane.
- **No harness's beacon works today:** `who_is_online` at 2026-10-02T16:50Z reports `turnPresence: null` for every rostered agent, across the Claude, GPT/Codex and Kimi families.
- **Promoted from defect-note** `MESSAGE:c19d1788-c7a6-4dcf-9ce1-619b4b02244e`, which #752 named out of scope. Two things wait on a Claude seat whose hooks reach the plane:
  - #562's PMV-1 and PMV-2.
  - `@neo-opus-vega`'s follow-up to #705, which retires the osascript route for Fleet-launched Claude seats.
- **The note's first reading was wrong.** It read "the Claude hook commands load no seat `.env`". The Problem below shows why loading that file is not the fix.

## The Problem

Every seat hook that reaches the plane resolves it from `AiConfig.fleet.planeBase` and `AiConfig.fleet.planeBearer`. Those leaves bind `NEO_FLEET_PLANE_BASE` and `NEO_FLEET_PLANE_BEARER` (`ai/configBase.mjs:334`, `:342`). There are four readers:
- `claude/wakeArmingHook.readSeatConfig`, which `wakeListenerHook` (#752) also reads;
- three byte-identical `readPlaneConfig` copies, in `claude/turnPresenceHook`, `kimi-code/turnPresenceHook` and `codex/codex-context`.

Those leaves configure the **Fleet transport**, which is a different principal from a seat:
- The bearer's own doc says the Fleet entry refuses plane mode unless the bearer's subject *is* the boot-resolved viewer. That is the single-viewer invariant.
- `ai/scripts/lifecycle/local-agent-os/README.md` ("Bind the host Fleet transport to this plane") calls `NEO_FLEET_PLANE_BEARER` a different credential class from the seat-side MCP slot `NEO_MCP_REMOTE_TOKEN`. It says never to copy or alias one class into the other.

So no seat can carry those leaves legitimately:
- **A Fleet-launched seat:**
  - `FleetLifecycleService.start` gives it the allowlisted ambient vars, its launch spec, and the reserved injections. Those are:
    - the PAT;
    - its own identity-bound plane credential under `NEO_MCP_REMOTE_TOKEN` (`:636`–`:639`, ADR 0041 §2.4);
    - the Bridge token;
    - `NEO_AGENT_IDENTITY`.
  - No file under `ai/services/fleet/` writes `NEO_FLEET_PLANE_BASE` (`git grep` at `447d96e`).
  - Desktop's launch env does reach its Code sessions. The LaunchServices markers (`__CFBundleIdentifier=com.anthropic.claudefordesktop`, `XPC_SERVICE_NAME`) are present in a Code session's shell.
  - So such a seat's hooks see exactly what the Fleet injected: the seat's own credential under the seat-side slot, and no base.
- **An operator-launched seat** could carry them only by aliasing the transport class into the seat, which the README forbids.

Every plane-reading seat hook is therefore a named skip on every seat. That is how #752 (pull wakes) and #758 (the turn-presence beacon) merged green and still do nothing on a real seat.

## The Architectural Reality

- **The seat's credential already reaches the seat process.** `REMOTE_MCP_CREDENTIAL_ENV_VAR = 'NEO_MCP_REMOTE_TOKEN'` (`ai/services/fleet/mcpServers.mjs:12`) is a reserved slot that `start` injects from `opts.resolvedMcpCredential`.
  - The generated MCP configs consume it.
  - No AiConfig leaf binds it (`git grep` at `447d96e`).
- **The seat's plane endpoint belongs to the seat, not the Fleet.**
  - `armFleetSeatWake` resolves `resolveSeatPlaneTarget({target, harnessType, planeBase}).endpoint`, either a tenant endpoint or the plane the Fleet serves, before it proves the stored credential (`armFleetSeatWake.mjs:88`–`:101`).
  - Nothing hands that endpoint to the seat process.
- **The hook seam is already clean.**
  - `connectSeatPlane` and `seatPlaneGap` (`ai/daemons/wake/armSeatWakePull.mjs:42`–`:67`) take `{planeBase, planeBearer}` as read by the entrypoint.
  - The writers take `{baseUrl, credential}` and resolve nothing themselves.
  - Only the leaves the four readers read are wrong.
  - The three `readPlaneConfig` copies show how the transport-leaf read spread to every harness at once.
- **The seat `.env` is not the route.**
  - Kimi's generated hooks load the seat `.env` (`generateKimiSeatConfig.mjs:243`).
  - The operator's 2026-09-27 ruling, recorded on #571, says the Fleet always holds the credentials and seat `.env` files are a stopgap until the Fleet Manager is usable. That route is not one to extend.

## The Fix

1. **Seat-side plane leaves in AiConfig** (ADR 0019 §5: declarative `leaf(default, env, type)`, read at the use site):
   - the seat's plane base under a new seat-side env slot (e.g. `NEO_SEAT_PLANE_BASE`);
   - the seat's bearer bound to the existing `NEO_MCP_REMOTE_TOKEN`.
   - They go in a seat-scoped subtree, not `fleet.*`, which belongs to the transport.
2. **`FleetLifecycleService.start` injects the seat's plane endpoint under the new reserved slot**, under the same condition as `NEO_MCP_REMOTE_TOKEN`.
   - The endpoint is the one the start path placed the seat on.
   - The slot joins the pairwise-distinct reserved-slot contract (`:514`–`:534`), so `launch.env` cannot pre-load it.
3. **One seat reader for every harness.** `ai/scripts/lifecycle/hooks/seatConfig.mjs` exports `readSeatConfig` (base, bearer, identity) and `readPlaneConfig` (the writer's `{baseUrl, credential}`), both from the seat-side leaves.
   - It replaces the three `readPlaneConfig` copies and `wakeArmingHook`'s reader.
   - The projector enumerates only `hooks/<harness>/*.mjs`, so the module stays runtime-resident and the hooks reach it through their rewritten imports.
   - `fleet.planeBase` and `fleet.planeBearer` stay the transport's alone.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Seat-side plane leaves (new) | ADR 0019 §5; README credential classes | Base ← the new seat slot; bearer ← `NEO_MCP_REMOTE_TOKEN` | Empty: the hooks keep their named skip, with the seat leaf named | Leaf JSDoc names the class split | AC-1, AC-4 |
| `FleetLifecycleService.start` reserved slots | Docblock `:262`–`:283` | Injects the seat's plane endpoint beside `NEO_MCP_REMOTE_TOKEN` | No credential → no base, exactly as today | Docblock lists the new slot | AC-2 |
| `hooks/seatConfig.mjs` (new; replaces four readers) | #562 / #752 / #758 | `readSeatConfig` and `readPlaneConfig` read the seat-side leaves | `seatPlaneGap` names the missing seat leaf. The turn-presence writers keep their generic named skip, because they resolve no config | Module JSDoc | AC-1, AC-3 |

## Decision Record impact

Aligned with ADR 0019: new declarative leaves are read at the use site, with no alias of `fleet.*` (C4 does not apply, since the credential classes differ). Aligned with ADR 0041 §2.4: the seat presents its own bound credential.

## Acceptance Criteria

- [ ] **AC-1:** A hook process whose env holds only a Fleet seat's injections (`NEO_AGENT_IDENTITY`, `NEO_MCP_REMOTE_TOKEN`, the new base slot, no `NEO_FLEET_PLANE_*`) resolves the seat's base and credential through the shared reader, and the Claude, Kimi and Codex hooks all read through it. It is red on dev with `fleet.planeBase is not configured`.
- [ ] **AC-2:** `start` injects the base slot exactly when it injects `NEO_MCP_REMOTE_TOKEN`, carrying the seat's placed endpoint. A `launch.env` naming the new slot is refused by the reserved-slot check.
- [ ] **AC-3:** A process holding only `NEO_FLEET_PLANE_BASE` and `NEO_FLEET_PLANE_BEARER` (a Fleet transport) does not make the hooks present the transport bearer as a seat.
- [ ] **AC-4:** `check-aiconfig-antipatterns` and `lint-config-template-ssot` are green on the new leaves.

## Post-Merge Validation

- [ ] **PMV-1:** On a Fleet-launched Claude seat started after the merge, the SessionStart `wakeArmingHook` reports armed, either in the seat's transcript or as the plane's arming receipt. This is `@neo-opus-vega`'s trigger for retiring the osascript route.

## Out of Scope

- **Operator-launched seats** get the slots only by being launched by the Fleet (#571 moves them). Nothing here writes a seat `.env`.
- **Dropping `claude-desktop` from the osascript dispatch:** `@neo-opus-vega`'s leaf, blocked on this ticket.
- **How Kimi's generated hooks load the seat `.env`.** Identity still rides that file. The Fleet-injected env wins over it.

## Avoided Traps

- **Loading the checkout `.env` in the Claude hook commands.** This is the `learn/agentos/Hooks.md` flavor, and `node --env-file-if-exists` does deliver the file. Two problems:
  - The file holds no seat-plane key today.
  - Filling it would alias the transport class into a seat, on the stopgap path the 2026-09-27 ruling retires.
- **The Fleet injecting `NEO_FLEET_PLANE_*` into a seat:** the same alias. Inside a seat, `fleet.planeBearer` would claim the transport's single-viewer subject.
- **Hooks reading `process.env.NEO_MCP_REMOTE_TOKEN` directly:** ADR 0019 A1; the leaf owns env resolution.

## Related

#562 · #752 · #758 · #705 · #79 · #728 · #571 · #30

## Sweeps

- Live latest-open sweep: checked the latest 20 open issues at 2026-10-02T16:45Z; no equivalent found.
- A2A in-flight claim sweep: the last 30 messages (all read-states, back to 14:53Z); no overlapping claim.
- MC sweep: two queries on the symptom (`wakeArmingHook UNARMED`, `fleet.planeBase not configured`, hooks and seat `.env`, which credential a seat presents). Prior art:
  - `@neo-fable` and `@neo-fable-clio` (2026-09-04) saw the same skip and read it as missing configuration;
  - my #562 intake measured the 41 transcripts.
  - No prior decision on the leaves or the credential class was found.
- Own-assignment sweep: 18 open. Read the bodies of #30, #67 and #571; none owns the seat hooks' plane leaves.

Origin Session ID: 6f7d14a3-e126-4b47-888f-fc28c748ae83

Retrieval Hint: "seat hooks fleet.planeBase transport leaves NEO_MCP_REMOTE_TOKEN seat-side plane slot"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-02T16:45:33Z @neo-opus-ada added the `bug` label
- 2026-10-02T16:45:34Z @neo-opus-ada added the `ai` label
- 2026-10-02T16:45:34Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-02T16:45:34Z @neo-opus-ada added the `agent-os` label
- 2026-10-02T16:49:20Z @neo-fable-clio cross-referenced by #767
- 2026-10-02T16:52:34Z @neo-opus-vega cross-referenced by #768
- 2026-10-02T16:52:38Z @neo-opus-vega marked this issue as blocking #768
- 2026-10-02T16:52:47Z @neo-opus-ada changed title from **Claude seat hooks resolve no plane: they read Fleet-transport leaves** to **Seat hooks resolve no plane: they read Fleet-transport leaves**
- 2026-10-02T17:09:59Z @neo-opus-ada cross-referenced by PR #771
- 2026-10-02T17:48:34Z @tobiu referenced in commit `1b1d4d1` - "fix(fleet): seat hooks reach the plane as the seat, from seat-side leaves the Fleet injects (#766) (#771)

Every plane-reaching seat hook read fleet.planeBase / fleet.planeBearer, the Fleet transport's
leaves, which no seat may carry, so each was a named skip on every seat. The hooks now read
AiConfig.seat through one runtime-resident reader, hooks/seatConfig.mjs, which replaces three
readPlaneConfig copies and wakeArmingHook.readSeatConfig. start injects the seat's placed
endpoint as NEO_SEAT_PLANE_BASE beside its NEO_MCP_REMOTE_TOKEN."
- 2026-10-02T17:48:34Z @tobiu closed this issue
- 2026-10-02T17:49:46Z @neo-opus-ada cross-referenced by #30
- 2026-10-02T18:03:37Z @neo-opus-ada cross-referenced by #460
- 2026-10-02T18:12:11Z @neo-opus-ada cross-referenced by PR #462

