---
id: 847
title: A plane still starting reads as 'not a Neo plane' and the FM gives up
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-30T23:49:37Z'
updatedAt: '2026-10-31T00:34:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/847'
author: neo-opus-ada
commentsCount: 0
parentIssue: 424
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-31T00:34:39Z'
---
# A plane still starting reads as 'not a Neo plane' and the FM gives up

## Context

Operator, 2026-10-30, prio 0: after a macOS restart the Fleet Manager and the Agent OS started by themselves. The FM said the address "is not a Neo plane", Connect failed too, and the operator generated a new PAT to get back. Earlier logins showed the same pattern: an FM that boots before the Agent OS never recovers.

Observed (FM `main.log`, `docker inspect`, `ps`; all UTC):

| time | event |
|---|---|
| 23:27:51 | macOS boot |
| 23:28:43 | FM boot, `plane-attach` to the stored plane `http://127.0.0.1:3102`. Fleet child: `plane unreachable (TypeError)` |
| 23:28:45 | `HARNESS_BRAIN_PLANE_STARTING`: #735's wait, bound 180 s |
| 23:29:04.197 | mc-server's first log line: the plane is coming up |
| 23:29:05.073 | `HARNESS_BRAIN_BOOT_FAILED`, 20 s into the wait, with no second Fleet spawn |
| 23:29:43 | all six plane containers start again (`StartedAt`) |
| 23:31:27 | FM process relaunched (Connect with the new PAT). 23:31:42 `plane init … verified: @tobiu` |

The wait ended on a probe verdict, not on its bound: 160 s were left and no second Fleet child spawned. The log cannot say which verdict, because `HARNESS_BRAIN_BOOT_FAILED` prints the Fleet child's first message (`plane unreachable (TypeError)`) even after the probe retyped the error. The operator's report names `not-a-plane`.

## The Problem

Reproduced on base (Institution `origin/dev` b645c38). The plane's own ingress config (Brain `deploy/cloud/Caddyfile.local-agent-os`, image `caddy:2-alpine`, as the local Agent OS runs it) runs in a throwaway container with mc-server not listening (`--add-host mc-server:127.0.0.1`) or not resolvable. Its answer to the probe's first `initialize` is **502** with no challenge, and `probePlaneCredential` returns `not-a-plane` in both cases:

```
{"planeBase":"http://127.0.0.1:39102","raw":"502 www-authenticate=null","verdict":"not-a-plane"}   # mc-server refuses
{"planeBase":"http://127.0.0.1:39103","raw":"502 www-authenticate=null","verdict":"not-a-plane"}   # mc-server unresolvable
```

The live plane, once up, answers `401` + `Bearer` to the same request.

Every cold start therefore has a window, ingress up and mc-server not yet listening, in which the plane reads as "not a Neo plane". The wait re-probes every 5 s, so a login almost always lands a probe inside that window. What follows:

1. **Boot:** #735's wait ends at once (`not-a-plane` is terminal by design). The spine says "That address is not a Neo plane. Connect to the plane's own address."
2. **Connect:** the card refuses a live plane the same way inside that window.
3. **Recovery costs a PAT:** Connect always prompts for a PAT (`createPlaneBroker().attach`). The stored one was never refused, but nothing else leads back, so the operator rotated a valid PAT.

Two more paths end the wait for good:

- **The plane restarts between an accepted probe and the Fleet child's own init.** Tonight's containers started again 39 s after mc-server first came up. The wait's second `runAttempt()` is not caught, so a child refusal there is terminal.
- **Silence past `PLANE_STARTING_BOUND_MS` (180 s).** Nothing probes after the bound, so a slow login (Docker plus a local model loading) never attaches without a manual relaunch.

## The Architectural Reality

- `harness/planeConfig.mjs` `probePlaneCredential`: step 1 returns `not-a-plane` for any answer other than 401 + `Bearer`. Step 2 types any status other than 200/401/403 as `initialized = null`, which is `not-a-plane`.
- Brain `deploy/cloud/Caddyfile.local-agent-os`: `handle_path /mc/* { reverse_proxy mc-server:3001 }`. The 401 challenge comes from mc-server itself (MCP SDK `requireBearerAuth`), so the ingress can only answer 502 until mc-server listens. The ingress logs nothing for those 502s (its `handle_errors 502` block handles them), which is why tonight's ingress log is empty. `harness/fixturePlane.mjs:179` answers 502 for an upstream error in the same way.
- `harness/brain.mjs` `bootFleetWithPlaneRecovery` enters the wait only on `plane-unreachable`. Inside it, any cause other than `plane-unreachable`/`plane-refused` throws, `return await runAttempt()` is unguarded, and `now() - startedAt < deadlineMs` ends the wait at 180 s.
- `harness/main.mjs` ≈:1875 logs `'HARNESS_BRAIN_BOOT_FAILED ' + error.message`. The typed `code` never reaches the log.
- `apps/agentos/util/SpineBanner.mjs` `coldFallbackFor` renders the `plane-starting` pill and lead (`Agent OS starting · N s`, plus "Longer than usual." after 60 s).

Design authority:

- `probePlaneCredential` JSDoc: "a plane answers with a bearer challenge (401 + `WWW-Authenticate: Bearer`), and any other answer ends the probe before the PAT leaves this process".
- #735 The Fix §4: "Silence past a named constant (proposed 3 min …) ends in today's `plane-unreachable` refusal … An answer that refuses the credential or is not a plane ends the wait at once".

Both assume a plane that answers for itself. Through its ingress, a starting plane answers 502. The operator's requirement (2026-10-30, prio 0) supersedes the bound: an FM that boots first recovers when the Agent OS comes up. `harness/brain.mjs` already rules waiting the honest move: "A configured plane is authority even when its ingress is currently unreachable".

## The Fix

1. **`probePlaneCredential`:** a 5xx at either `initialize` means the plane has not answered yet, so the verdict is `unreachable`. The JSDoc says why (the ingress answers for an upstream that is not up). Every other answer keeps today's verdict.
2. **`bootFleetWithPlaneRecovery`:** a stored plane's wait has no deadline. It ends on an answer or on quit. An accepted answer attaches. A credential refusal, a changed identity or "not a plane" ends it with that typed refusal, as in #735. A Fleet child that the plane refuses as unreachable after an accepted probe keeps the wait going. `PLANE_STARTING_BOUND_MS` is deleted.
3. **`SpineBanner` `plane-starting`:** from 3 min the pill reads `Agent OS not answering · N min`, and the lead names the action #735's refusal named: "Your Agent OS has not answered for N min. Start it, or connect to another; the Fleet joins by itself when it answers."
4. **`HARNESS_BRAIN_BOOT_FAILED`** carries the typed cause: `HARNESS_BRAIN_BOOT_FAILED [<code|untyped>] <message>`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `probePlaneCredential` (`harness/planeConfig.mjs`) | Its JSDoc (bearer challenge first) + this ticket's premise (Brain `Caddyfile.local-agent-os`) | A 5xx at the unauthenticated or the authenticated `initialize` → `{verdict: 'unreachable', identity: null}`. Unchanged: 401 + `Bearer` then a 200 naming `neo-memory-core` → the accepted path; 401/403 at the second `initialize` → `rejected`; any other first answer (200/404/405…), or a 200 naming another server → `not-a-plane` | No new verdict. Connect (`createPlaneBroker().attach`) now refuses a 5xx with "No plane answered at that address."; `verify` → `runtimePlaneCause` stays `null` for both verdicts; `fixturePlane.mjs` only names the other verdict in its own error | JSDoc | `planeConfig.spec.mjs` (5xx at both steps, PAT withheld at the first) + the unchanged not-a-plane/rejected tests; L3 throwaway-ingress probe |
| `bootFleetWithPlaneRecovery` (`harness/brain.mjs`) | Operator requirement 2026-10-30 (supersedes #735 §4's bound); `brain.mjs` "A configured plane is authority even when its ingress is currently unreachable" | Entry unchanged: a stored record whose boot typed `plane-unreachable`. Re-probes every `PLANE_STARTING_RETRY_MS` (5 s), one in flight, no deadline. Accepted with the stored identity → respawn; a respawn the plane refuses as `plane-unreachable` → the wait goes on | A refused credential, a changed identity or not-a-plane → that typed refusal at once (#735); any other respawn failure → thrown as is; `signal` (quit) → the boot's own `plane-unreachable` refusal. The `deadlineMs` option and the `PLANE_STARTING_BOUND_MS` export are removed; `brain.spec.mjs` was their only consumer | JSDoc | `brain.spec.mjs`: the three #847 tests + #735 AC-1, AC-2 (at once) and AC-4, run by CI's Brain-bound unit contract |
| `coldFallbackFor`, `plane-starting` branch (`apps/agentos/util/SpineBanner.mjs`) | #735 AC-3's words; #477 (a reason and a next step) | Under 180 s unchanged: `Agent OS starting · N s`, its lead, `Longer than usual.` from 60 s. From 180 s: text `Agent OS not answering · N min`, lead "Your Agent OS has not answered for N min. Start it, or connect to another; the Fleet joins by itself when it answers." | `action: null` throughout (Connect stays where it is). The 180 s threshold is renderer-local (`PLANE_SILENT_SECONDS`); no harness constant crosses the wire | `learn/CockpitStateCensus.md` rows | `spineBanner.spec.mjs` (#847 test); visual stamp moves SpineBanner's blob only |
| `HARNESS_BRAIN_BOOT_FAILED` line (`harness/main.mjs`, the boot's `.catch`) | This ticket's Context: the log could not say which verdict ended the wait | `HARNESS_BRAIN_BOOT_FAILED [<cause.source>] <message>`, where `cause = bootFailureCause(error)`, the same value `settleBrainBoot` receives | `[untyped]` for an error outside `TYPED_BOOT_REFUSALS`. No reader parses the line (searched Institution and Brain) | none | L4, deferred: read on AC-7's receipt (#516) |

## Acceptance Criteria

- [ ] AC-1: `probePlaneCredential` returns `unreachable` for a 5xx at the first `initialize` and at the authenticated one. The 401+Bearer, 200 and 4xx verdicts are unchanged. Covered by a unit test, and the throwaway-ingress probe above is green at head (same instrument as on base).
- [ ] AC-2: a stored plane that answers with refused connections, then 502s, then an accepted probe whose Fleet child the plane refuses as unreachable, then acceptance, attaches the Fleet with no relaunch. Unit test with probe and child seams.
- [ ] AC-3: a stored plane silent past 180 s keeps waiting and attaches when it answers. Quit ends the wait. The #735 AC-2 bound test is replaced. Unit test.
- [ ] AC-4: a credential refusal, a changed identity or "not a plane" still ends the wait at once with its typed refusal. Unit test.
- [ ] AC-5: `HARNESS_BRAIN_BOOT_FAILED` names the typed cause, read from `bootFailureCause` (the lifecycle's own typing). `harness/main.mjs` has no unit seam; the line is read on AC-7's receipt.
- [ ] AC-6: the `plane-starting` pill and lead match The Fix §3, before and after 3 min. Spine banner spec.
- [ ] AC-7 `[L4-deferred]`: on the installed FM, a macOS restart with the FM and the Agent OS starting by themselves reaches the live roster with no relaunch and no PAT. One receipt on #516.

## Out of Scope

- Reconnecting with the stored PAT after a boot refusal that is not the credential's (Connect always prompts for one). A defect-note is filed. The wait above removes the case that sent the operator there.
- A PAT update relaunching the FM (#843).
- A cold Memory Core that cannot reach GitHub answers a valid PAT with 401: Brain `AuthService` ends its unreachable branch in `InvalidTokenError`, and the probe types that as `rejected`. Known from reading the code only, not observed tonight.
- The roster read timing out on a cold relaunch (#841).

## Avoided Traps

- **Ordering the login items (FM after Docker).** #735 already ruled this out: launch order says nothing about when a plane answers.
- **Waiting through `not-a-plane`.** After Fix 1, a starting plane no longer answers it. A stored address that answers as another server is a real refusal.
- **Reading the ingress's body or headers.** The status class is the contract, and any gateway answers 5xx for a missing upstream.

## Related

Part of #424 · #735 (the wait this un-bounds) · #425 (its "Recovery is the operator's Connect" assumed Connect costs nothing; it costs a PAT) · #843 · #841 · #516 (row 5, the H0 cold start)

Live latest-open + closed sweep: the latest 20 open issues plus closed searches ("plane starting not a plane login", "probePlaneCredential OR 502 OR not a Neo plane"); #735, #734, #425 and #446 read, including Out of Scope and Avoided Traps; at 2026-10-30T23:47Z. No equivalent or prior exclusion. A2A sweep (last 30, all states, through 23:47Z): no claim on this scope. MC sweep: "address is not a neo plane Fleet Manager connect probe" and "Fleet Manager started before Agent OS after laptop restart never reconnects", 16 results, no prior decision. Own-assignment sweep: #843, #516 and #424 read; #424 is the parent. Structure map: N/A (existing `harness/` and `apps/agentos/` files, no new placement). Decision Record impact: none.

Retrieval Hint: "FM plane-starting wait not-a-plane 502 ingress mc-server cold login PAT rotated"




## Timeline

- 2026-10-30T23:49:38Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-30T23:49:39Z @neo-opus-ada added the `bug` label
- 2026-10-30T23:49:39Z @neo-opus-ada added the `agent-os` label
- 2026-10-30T23:49:39Z @neo-opus-ada added the `ai` label
- 2026-10-30T23:49:48Z @neo-opus-ada added parent issue #424
- 2026-10-31T00:03:22Z @neo-opus-ada cross-referenced by PR #848
- 2026-10-31T00:05:13Z @neo-opus-ada cross-referenced by #849
- 2026-10-31T00:34:39Z @tobiu referenced in commit `2ba5784` - "fix(agentos): a plane still starting no longer ends the Fleet Manager's wait (#847) (#848)

The plane's ingress answers 502 for its Memory Core until that listens, and the probe read any
answer but a bearer challenge as "not a Neo plane", which ends the login wait at once. A 5xx at
either initialize now reads as unreachable. A stored plane's wait has no deadline: it ends on an
answer or on quit, and a Fleet child the plane refuses as unreachable after an accepted probe keeps
it going. From 3 min the pill names the action. HARNESS_BRAIN_BOOT_FAILED names its typed cause.

The visual stamp moves SpineBanner's blob only. Its engine line stays at 95e63df6: two
FleetCockpitVisual goldens differ at the current engine, on base as on head."
- 2026-10-31T00:34:41Z @tobiu closed this issue
- 2026-10-31T00:35:26Z @neo-fable-mnemosyne cross-referenced by #1390
- 2026-10-31T00:48:56Z @neo-opus-ada cross-referenced by #516
- 2026-10-31T00:52:40Z @neo-opus-vega cross-referenced by PR #1391

