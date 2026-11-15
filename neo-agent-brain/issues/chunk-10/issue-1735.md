---
id: 1735
title: Retire AppleScript message delivery after native peer validation
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-15T01:14:15Z'
updatedAt: '2026-11-15T06:14:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1735'
author: neo-gpt-emmy
commentsCount: 7
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1731 Fleet provisions native wakes for managed Codex and Antigravity seats'
blocking: []
---
# Retire AppleScript message delivery after native peer validation

## Context

Design authority: the operator elected a one-time cut after signed native delivery works for our real peers. This is the retirement step for the normal Fleet setup in #1731. Existing operators do not need a permanent legacy-mode switch.

## The Problem

Removing the old route before both real peer journeys are proven would strand running seats. Keeping the old sender after they are proven would leave supported clients able to type a peer message into a desktop composer, including after a future configuration regression. Codex has a confirmed installed wake. The signed Antigravity real-recipient witness is still pending on #571.

## The Architectural Reality and Fix

Fleet chooses ordinary wake transport in `armFleetSeatWake`. The receiver accepts historical signed snapshots in `receiver.mjs` and dispatches through `localWakeAdapters.mjs`. `daemon.mjs` also has a direct AppleScript sender. Migrate durable unmatched records once by subscription/seat and preserve proven receipt and unknown-send semantics. Remove window typing and its selection/defaults for A2A digests in both paths. Inspect subscribers and manifest readers, templates and diagnostics against the emitted supported contract. Preserve the independent authenticated Claude `session/rotate` command and the existing non-typing OpenCode transport. Evaluate OpenCode separately with evidence if its use resumes.

| Target | Source | Behavior after cut | Verification |
| --- | --- | --- | --- |
| Signed `wake/digest` / direct send | Family-native transport and durable mailbox | No path writes peer text, simulates Enter, creates a new chat, or downgrades a native record to GUI typing | Trace both dispatch entry points, tampered/missing route, ambiguous send, startup/handoff and retry |
| Existing operator’s fleet | Installed receipt and authenticated seat bindings | Convert remaining eligible seats once, preserve old queued work without duplicates, leave foreign/private profiles alone | Reconcile selected root, no subagent delivery, current-epoch/model receipt and per-seat migration results |
| Explicit `session/rotate` | Separate signed rotation route | A deliberate Claude sunset still opens its one successor | Existing isolated rotation suite plus installed sunset receipt |

## Acceptance Criteria

- [ ] Before removal, attach installed signed receipts for real Codex and Antigravity main chats on the provisioned paths, with focus/draft observations, old-root/helper rejection and queue disposition. Keep the operational legacy lane until this evidence lands.
- [ ] Default Add/Start for a new managed Codex or Antigravity seat selects native delivery without an enrollment CLI or any global Codex setting; an unready seat keeps its mail pending with a visible reason.
- [ ] Remove A2A typing senders/defaults and tests that bless them, replacing them with negative controls: no typing dispatch under missing/mismatch/retry, and one native delivery after recovery. Claude hook delivery and the separate sunset command retain positive controls. No permanent legacy preference.
- [ ] Record one migration for the current fleet and repeat the fresh-operator installation smoke with independent seats, including coexistence with an unmanaged Codex instance. Installed owner: #571.

Out of scope: OpenCode changes without a real owner/use case; removing osascript uses outside A2A message delivery.
Decision Record impact: aligned with the existing signed wake/rotation split. Related: #1731 · #1706 · #562 · #768 · #1373. Not a newcomer ticket: execution waits for the protected installed recipients.
Live latest-open + closed sweep: newest 20 Brain issues, red backlog; AppleScript titles #30, #562, #606, #768 are earlier Claude migration or legacy reliability work. #1731 owns preparation. No competing retirement claim in the recent 30 A2A rows.
Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: "retire osascript peer wake digest after real Codex and Antigravity receipt"

## Timeline

- 2026-11-15T01:14:17Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-15T01:14:17Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T01:14:17Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T01:15:18Z @neo-gpt-emmy cross-referenced by #1731
- 2026-11-15T01:16:59Z @neo-gpt-emmy marked this issue as being blocked by #1731
- 2026-11-15T01:25:35Z @neo-gpt-emmy cross-referenced by PR #1737
- 2026-11-15T01:29:03Z @neo-gpt-emmy cross-referenced by #1738
- 2026-11-15T02:25:03Z @neo-gpt-emmy cross-referenced by #1741
### @neo-gpt-emmy - 2026-11-15T02:43:34Z

### Read-only pre-cut rehearsal · 15 November

The two existing native trials predate the setup receipts introduced by #1737. I checked their enrollment and exact native manifest routes, each live App process/epoch and registered profile, and the old commands against a full verification of the retained immutable runtime generation. In disposable copies, the current materializers preserve foreign hooks and trust data and refuse a different runtime if they lack proof of the old registration. The Codex trial used the main packaged executable; the Antigravity trial used its Helper and dependency-layout modules. A migration cannot assume one old path for both families.

Antigravity’s actual enabled sidecar entry also carries `projectId` and `schemaVersion: 1`. The live config remained unchanged. Google documents the project field as the destination for **creating** conversations. This sidecar uses only the existing-conversation send operation, with the seat lease and its registered project as routing authority. The PR is being corrected to preserve these fields and verify the actual shape. [Vendor sidecar contract](https://antigravity.google/docs/sidecars#user-configuration-configjson)

**At cut:** adopt a prior definition only after the receiving deployment, enrolled seat/family, live process and workspace, complete retained generation and exact old command/definition all agree. Preserve disabled settings and private Codex profiles, and reverify any changed Codex command through native trust. Hold unproved cases; reference checks and this rehearsal establish no signed recipient receipt. The installed acceptance remains on #571.

- 2026-11-15T02:44:51Z @neo-gpt-emmy cross-referenced by #1743
### @neo-gpt-emmy - 2026-11-15T03:04:40Z

### Merged package rehearsal · 15 November

Staged the two merged `dev` revisions together, Fleet Manager `195de6e1` and Brain `4c0517a9`, with Engine `2f38667e`. The packaging freshness check found the backend equal to `origin/dev` at 03:33 UTC, without an override. The stage rebuilt native dependencies for Electron 43.8.1. Runtime checks used a matching packaged Electron in Node mode.

- **Host to Fleet.** All 22 merged host-carrier tests passed, including the Fleet child environment. A disposable LaunchAgent supplied a deliberately nonstandard state directory. Fleet preparation and its host receipt used the directory the carrier actually returned.
- **Settings.** Both families passed initial preparation, upgrade and retry on disposable homes. An Antigravity registration containing `projectId` and `schemaVersion: 1` retained its exact bytes, mode and inode, including unrelated settings. The Codex trust file remained intact.
- **Delivery.** With the carrier-supplied directory, each signed receiver fixture settled one delivery. Repeating the request caused no second send. Antigravity rejected fork and child targets. Vendor processes, sends and model receipts were simulated. The UI was exercised separately by [the Fleet PR](https://github.com/neomjs/neo-agent-institution/pull/981).
- **Live boundary.** Read-only inspection confirmed the merged inspector accepts the enrolled Antigravity seat’s actual enabled registration and verified executable. Its signed request still reports `native-recipient:awaiting-readiness` without a matching model receipt. The installed desktop bundle checked after merge still contains earlier revisions.

The next installed check is the regular seat action followed by a genuine signed receipt in the current main, observing any existing draft. Julia’s next naturally started main is the intended Antigravity trial. The startup-ordering and restored-root controls remain on [#1743](https://github.com/neomjs/neo-agent-brain/issues/1743#issuecomment-6888271295).

### @neo-gpt-emmy - 2026-11-15T03:54:38Z

### Complete macOS bundle and pre-install rehearsal

Built the unsigned arm64 `.app` from merged Institution `195de6e1`, Brain `4c0517a9` and Engine `2f38667e` with pinned Electron 43.8.1. The bundled executable ran against its own resources. The wake receiver carrier in `app.asar` matched the merged host source, and both family fixtures settled one signed wake and suppressed a duplicate. Antigravity’s fork and child controls refused. Vendor/model endpoints in those fixtures are stand-ins.

The packaged fixture-plane GUI smoke passed. Fleet attached to its own disposable plane, the roster answered, both windows mounted, no renderer or asset failures were reported, and all temporary ports were released. Launch-to-semantic-first-paint was 22.183 s, of which 2.754 s was renderer time. The official pre-install rehearsal with its seeded viewer and Claude/Codex stand-ins also passed with zero failures, including restart/recovery. The rehearsal receipt SHA-256 is `eb3dab635852e718ff0f0f9716c03b78ece5e2a09e4ab4f62f2c3641912c8bcf`.

Two earlier checks on the same bundle are distinct results. The cold UI-only smoke loaded both windows but missed the launch-to-first-paint budget at 74.107 s. In own-mode smoke, the disposable graph did not contain the inherited maintainer identity, so Fleet correctly refused to bind and the test exited nonzero. The fixture-plane and seeded pre-install paths above exercised working configurations. The canonical installed app still has earlier revisions. Installed receipt in the current real Antigravity main remains outstanding.

- 2026-11-15T04:11:07Z @neo-gpt-emmy cross-referenced by PR #983
### @neo-gpt-emmy - 2026-11-15T04:17:16Z

The canonical installation dry run exposed a blocker outside the bundle. Two retained Electron Helpers were running the Antigravity wake sidecar, and the checkout installer classified their nested `.app` names as separate Fleet applications. The original installer exited before any change. [Institution #983](https://github.com/neomjs/neo-agent-institution/pull/983) passed 122/122 focused tests and seven hosted checks, was independently approved, and merged as `929e04fa` at 04:23 UTC. I reran the supported dry run from that clean `dev` revision. It printed the normal 13-step update plan with both sidecars still alive. The read-only pre-Quit continuity check also passed. The existing rehearsed bundle needs no rebuild for this checkout-side fix.

The two running helpers also gave a live check of the shared-sidecar boundary. Reading process ancestry, the enrollment, lease and project found exactly one matching the enrolled Antigravity instance. The other helper belonged to a different app profile. No wake was sent for this check. The enrolled peer’s existing signed record remained pending throughout a 50-second live-process observation.

Before retiring the sender, re-read the manifest as well as the live roster. At the time of this check it had two `primary-session` wake identities and five `osascript` wake identities, including the other two GPT seats and the other team Gemini peer. It also had five separately typed Claude `session/rotate` routes; preserve that command. A manifest row alone does not establish whether its seat is active.

- 2026-11-15T04:40:48Z @neo-gpt-emmy cross-referenced by #732
### @neo-gpt-emmy - 2026-11-15T04:41:12Z

The rehearsed desktop package is now installed under operator authorization. The canonical app and retained runtime carry Institution `195de6e1` / Brain `4c0517a9`. The replaced app is in the rollback slot, custody matched, and all nine running peers kept their original process births. The reopened cockpit attached to its saved plane, loaded 13 roster entries, and exposed `nativeWakeSetup.expectedStartedAt` on all three Codex and both Antigravity team seats. [Installed continuity receipt](https://github.com/neomjs/neo-agent-institution/issues/732#issuecomment-6889287276).

The existing enrolled Codex and Antigravity processes also matched their original lease and profile after the update. The original Antigravity signed request remained pending, with no selected new root or ordinary invocation witness. That is still the outstanding real-recipient evidence. The app installation did not perform the fleet-wide transport switch or a container deployment.

### @neo-gpt-emmy - 2026-11-15T05:01:52Z

**Installed setup on the two original trial seats.** The installed cockpit’s authenticated, epoch-bound setup initially refused each unrecorded manual definition and preserved the existing registration. For both trials I verified the old and installed immutable runtime generations and the exact, previously approved commands against their seat bindings. The installed Fleet action created its own private preparation records after the one-time reconciliation.

**Antigravity.** The original signed request had no attempt, claim or selected root. Only the enrolled trial’s workspace hooks and the shared sidecar definition moved. The enabled user configuration remained byte-identical. Fleet returned `prepared` for the hooks and sidecar and `ready` with recipient unknown. Kernel ancestry showed one new-runtime sidecar in the same enrolled App process, zero old-runtime sidecars there, and the other App profile excluded. The original signed request remains pending for the intended main session.

**Codex.** The old hook’s opaque marker reached the existing main chat, its private acknowledgement matched the original signed record, and the receiver terminalized it as delivered. The queue then contained zero native markers for that chat and the old generation had no unresolved attempt. With that barrier satisfied, only the enrolled seat’s owned submit-hook command moved to the installed runtime. The trust file and all foreign hooks were preserved. The installed Fleet setup returned `prepared` for the hook and `ready` with recipient unknown, and its published hook path matches the new host receipt. The next durable signed record remains pending without an attempted send. The vendor’s actual inventory reports the new hook `enabled` and `modified`; a new native trust decision and an ordinary invocation in the same main chat are needed. No trust setting was written.

In a disposable vendor-inventory control, formatting alone preserved both key and hash. Changing the generation paths alone preserved the key but changed the current hash. The post-migration signed model receipts remain outstanding for both families; source preparation or route publication does not complete this cut.

**Remaining upgrade case for outside operators.** The stable-command requirement from #1731 remains relevant to #1735. Fleet currently derives both the node executable and module path from its running package. Even an otherwise identical command changes hash when the immutable generation path changes. Keeping a prior command would require an independently verified compatible hook and a real retention reference for its generation; the preparation receipt is not such a reference today. Any design must retain the vendor’s per-definition trust and current-process witness. An unverified executable behind a stable path would weaken that contract.

- 2026-11-15T05:30:25Z @neo-gpt-emmy cross-referenced by #1750
### @neo-gpt-emmy - 2026-11-15T05:31:53Z

The installed manifest has three active Fleet seats still using window typing for ordinary messages: two Codex seats and one Antigravity seat. Two more entries need different dispositions during migration. One is a former Claude login without a current Fleet definition. A historical alias alone cannot authorize forwarding its mailbox. The other is a benched OpenCode seat, covered by this issue’s existing OpenCode exclusion. The five Claude sunset routes serve the separate signed rotation operation.

The upgrade control and repair now have a source owner in #1750. With a simulated Codex resident and vendor admission, the older hook and newer receiver delivered one signed digest on the explicitly matched command and zero on a mismatched command. A separate retention test demonstrated that a preparation receipt alone does not protect its generation once other references disappear. Neither experiment changes an installed route. The two pilot records continue under their existing signed identities.

After the 06:07 UTC Agent OS container update, all four services read back Brain `6cfdf63e`. The two host receiver records kept their original acceptance times and still pointed to their enrolled, surviving peer processes. Both were pending without a matched recipient acknowledgement. In the Antigravity pilot, a read-only comparison restricted to its registered project found no new eligible root outside its baseline, no selected root and no ordinary invocation witness. In the Codex pilot, the enrolled hook still reported `modified`. This records continuity across the container update. The live delivery checks remain open on #571.

- 2026-11-15T06:30:57Z @neo-opus-ada cross-referenced by PR #1751

