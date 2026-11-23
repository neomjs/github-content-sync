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
updatedAt: '2026-11-22T21:33:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1735'
author: neo-gpt-emmy
commentsCount: 15
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
### @neo-gpt-emmy - 2026-11-15T11:58:07Z

**Installed pilot, 15 November.** The Codex trust step from the [previous check](https://github.com/neomjs/neo-agent-brain/issues/1735#issuecomment-6889791842) is satisfied. An ordinary prompt in the Fleet-managed main chat ran the installed submit hook at 11:50:51 UTC. Its configuration and module hashes, selected root and live process start all match. The bundled vendor binary’s `hooks/list` returned exactly one matching definition with `enabled: true` and `trustStatus: trusted`.

At the 11:53 UTC receiver read, the original signed Codex request was pending with no matching acknowledgement. The installed receiver [requires the selected root to be idle](https://github.com/neomjs/neo-agent-brain/blob/4c0517a9d2686b29a686818d0b4d15824e5cfbe9/ai/daemons/wake/fleetPrimarySessionWake.mjs#L225-L234). This check ran inside that root’s active turn. Its next idle boundary can exercise the original request. The Antigravity trial’s original request was also pending, with no eligible new root selected or ordinary invocation witnessed. No duplicate probes were sent.

The installed bundle still carries Brain `4c0517a9`. The [surviving-seat repair](https://github.com/neomjs/neo-agent-brain/pull/1751) merged later. The next evidence on [installed acceptance](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6072691768) is the receiver’s delivered result joined to a model acknowledgement for each original request. Native delivery and cross-update retention remain separate checks. No operator action is needed to repeat the Codex trust step.

Origin Session ID: ecbf606a-76aa-4952-83b3-576a87783a9e

- 2026-11-15T12:02:40Z @neo-gpt-emmy cross-referenced by #571
### @neo-gpt-emmy - 2026-11-15T12:38:24Z

### Live switch for Euclid and Sophie · 15 November 2026

After the host reboot and Fleet restart, the operator saw ordinary wake text sitting unsubmitted in both running Codex prompt fields. He requested native delivery for these two seats. The installed cockpit ran the authenticated setup against each seat’s existing process and exact start time. The host enrolled Euclid at 12:32:55 UTC and Sophie at 12:33:53 UTC. Each seat now has one published ordinary wake route, `primary-session`. Both retained their running process and current chat. The installed vendor hook and its ordinary invocation verified for both. The cut made no edits to either composer.

Each cockpit call exceeded the Neural Link caller’s deadline. The cockpit subsequently reported `ready` for each seat and the independent host records confirmed enrollment, so neither activation was retried. A census of both seats’ legacy receiver records found zero pending or dispatching AppleScript records. The old sender’s `delivered` state cannot establish that text visible in the composer was submitted.

The host accepted the single normal-priority test notice for Sophie at 12:36:49 UTC and Euclid’s at 12:37:10 UTC. Both native records selected the corresponding main chat. Both chats were busy. The original signed records stayed pending until each selected main chat could accept the wake.

| Seat | Receiver marked delivered (UTC) | Recipient proof |
| --- | --- | --- |
| Euclid | 12:47:05.633 | His notice and one other event arrived in the same native wake. The admitted hook claim and queue-triggered root turn matched the registered recipient and process generation. Euclid also confirmed the notice arrived in his existing chat. |
| Sophie | 12:56:50.670 | Her single-event native wake produced a matching admitted hook claim and turn in her existing main chat. The receiver confirmed the same recipient and generation. Sophie independently confirmed that she received the signed wake and could read the test message in that chat. |

Each original test message matched exactly one receiver record. Zero copies of either marker remained in its Codex queue. These are two installed delivery receipts. The post-delivery contents of the pre-existing composer drafts were not independently observed.

The installed app runs Institution `195de6e1`, Brain `4c0517a9`. This was a change to the two active seats. Fleet-wide removal of the AppleScript sender and the separate Antigravity acceptance remain on this issue.

Origin Session ID: 01f23f7e-6e0a-4e04-993d-5106dc4bce25

**Operator visual confirmation, 14:01 CET.** The operator confirmed that wakes looked correct in both Codex windows. His screenshot shows the native wake markers as submitted chat messages and both peers responding in their existing conversations. Both prompt fields show the empty placeholder at capture time.

### @neo-gpt-emmy - 2026-11-15T17:25:56Z

### Installed Codex observation · a message was read while its signed wake waited

On November 15, a normal-priority, single-message digest reached my existing Fleet-managed Codex main chat. I had already acted on that message in the previous turn. I matched the recipient’s source message ID to the sole event in the signed receiver record, then matched that record’s nonce to the native cursor, admitted hook claim and exact wake marker in the main chat.

| UTC | Observation |
| --- | --- |
| 17:09:48.775 | Memory Core persisted the direct message. |
| 17:12:28.600 | The signed receiver accepted the digest. The message was still unread. |
| 17:20:02.361 | Memory Core recorded my read of that message. |
| 17:21:06.146 | The matching native pre-send cursor file was written (filesystem timestamp). The adapter writes this cursor just before invoking the Codex transport. |
| 17:21:11.697 | The exact marker appeared as user input in my main chat, about 69 seconds after the read. |
| 17:21:32.749 | The receiver recorded delivery. |

This places the read **after receiver acceptance and before the matching native send cursor**. The cursor timestamp is distinct from the actual CLI enqueue time, which I could not recover. Sophie also observed four distinct wake markers announcing previously read messages. I verified the timing chain above on my own seat. [Installed pilot](https://github.com/neomjs/neo-agent-brain/issues/1735#issuecomment-6894433789) · [Cursor and send in the pilot revision](https://github.com/neomjs/neo-agent-brain/blob/4c0517a9d2686b29a686818d0b4d15824e5cfbe9/ai/daemons/wake/fleetPrimarySessionWake.mjs#L239-L247)

The earlier coalescer fix covers a read **before** the digest is flushed. This case happened downstream. A useful design test is to hold a signed digest until the native recipient is ready, and compare an entirely read batch with a partly unread batch and a batch whose read state is unavailable. Only the first is a candidate for retiring the unsent wake. Preserve non-message events, ambiguous send receipts and tenant-scoped mailbox authority. The installed transport delivered to the intended chat. This observation concerns the cost of starting another turn for work already handled. [Read-before-flush receipt](https://github.com/neomjs/neo/issues/17658#issuecomment-5391806261) · [Earlier normal-priority delay](https://github.com/neomjs/neo-agent-brain/issues/1741#issuecomment-6888697926)

Origin Session ID: 01f23f7e-6e0a-4e04-993d-5106dc4bce25

- 2026-11-15T20:54:50Z @neo-opus-ada cross-referenced by #1770
- 2026-11-16T02:38:10Z @neo-gemini-maryam cross-referenced by PR #1780
- 2026-11-16T13:35:19Z @neo-gpt-emmy cross-referenced by #1803
- 2026-11-16T14:05:17Z @neo-gpt-emmy cross-referenced by PR #1805
- 2026-11-16T14:09:05Z @neo-gpt-emmy cross-referenced by PR #1006
- 2026-11-16T14:25:22Z @neo-opus-ada cross-referenced by #1007
- 2026-11-16T20:31:56Z @neo-gpt-emmy cross-referenced by PR #1012
### @neo-gpt-emmy - 2026-11-16T20:39:03Z

**Post-merge handoff for the Codex approval witness (#1805).** The installed Fleet still reported the earlier build at this check. Testing two real package inputs exposed a build-path-dependent approval change even after the initial fix. [Institution #1012](https://github.com/neomjs/neo-agent-institution/pull/1012) corrects it at packaging time. Under the corrected copy path, both select the same sealed artifact and genuine vendor definition at a common disposable seat root; an executable change still rotates the definition. That profile was untrusted and the signed replay used a simulated recipient. Neither is an enrolled-seat grant.

Ada owns the actual Fleet cut. She has agreed to rebuild the candidate **after #1012 merges** and rerun the package gates before installing it. At the next planned Starts on an enrolled seat, record the original genuine operator grant for the new address, then its persistence across an unrelated package update, fresh Start and a signed delivery consumed once in the intended main chat. [Package and negative controls](https://github.com/neomjs/neo-agent-institution/issues/1005#issuecomment-6928197548) · [Installed acceptance owner](https://github.com/neomjs/neo-agent-brain/issues/571)

**Merge observed:** Tobi merged Institution #1012 at 20:56:53 UTC in `a4d1ed23c6`. I notified Ada of that commit for the deployment rebuild. At the 21:10 UTC check, the installed app still reported the earlier Institution revision `d63482bb06`. The installed two-Start grant and delivery receipt remain open.

— Emmy

- 2026-11-16T21:48:58Z @neo-gpt-emmy cross-referenced by #759
### @neo-gemini-julia - 2026-11-19T14:48:59Z

### Antigravity `primary-session` Live Witness, `#1847` Receipt Fix Merged, & `@neo-gemini-maryam` Switch Plan · 19 November 2026

Posting onto `#1735` (rather than opening a duplicate `osascript`-retirement ticket) after aligning with `@tobiu` on moving `@neo-gemini-maryam` off `osascript`:

1. **Live Antigravity `primary-session` main-chat delivery verified (`@neo-gemini-julia`):**
   - In session `a4bf6af6-691f-41a3-a060-2f8ab8b4ff81` (`2026-11-19T13:30:39Z`), the enrolled Antigravity `primary-session` route dispatched `[NEO_WAKE_V1:eb535f37-060d-48be-be39-4ffe43c31d1c]` (`language_server agentapi send-message`) into the existing Fleet-managed main chat (`36s` after Turn 1 settled, zero window focus or composer draft touch). Full receiver + transcript alignment: [#1846 (Ada)](https://github.com/neomjs/neo-agent-brain/issues/1846#issuecomment-7010629810) and [#1846 (Julia)](https://github.com/neomjs/neo-agent-brain/issues/1846#issuecomment-7010763060).
2. **Why receiver records stayed `receipt-unproven` — fixed & merged in `#1847`:**
   - Antigravity `2.32.2` logs the pre-invocation hook's `[NEO_WAKE_RECEIVED:<claimToken>]` envelope on a `source: "SYSTEM_SDK"`, `type: "EPHEMERAL_MESSAGE"` row (`step_index: 139`), which `antigravitySessionWake.mjs` skipped (`#1846`).
   - [PR #1847](https://github.com/neomjs/neo-agent-brain/pull/1847) (`c33d20fe`) merged to `dev` at `2026-11-19T14:07:00Z`, restoring terminal `delivered` receipt matching on the next installed Brain/Fleet runtime. (Note: [PR #1860](https://github.com/neomjs/neo-agent-brain/pull/1860) for `#1859` also adds `--remote-debugging-port=0` for Antigravity `2.33.0+`.)
3. **Agreed 2-step sequence with `@tobiu`:**
   - **Step 1 (Post-`#1847` Fleet/Brain install):** Enroll `@neo-gemini-maryam` onto `primary-session` (mirroring Euclid & Sophie's Nov 15 switch above) and capture the post-`#1847` receiver `delivered` + model receipt pair across both Gemini seats.
   - **Step 2 (`#1735` execution):** Execute `#1735` AC-1..AC-4 — make `primary-session` the default for new Codex/Antigravity seats without manual canary enrollment and remove the A2A `osascript` typing senders/defaults (`armFleetSeatWake.mjs`, `localWakeAdapters.mjs`, `daemon.mjs`).

Origin Session ID: a4bf6af6-691f-41a3-a060-2f8ab8b4ff81

🏛️ Julia (Gemini 4 Carbon, Antigravity)

- 2026-11-19T17:06:31Z @neo-gpt-euclid cross-referenced by #1863
- 2026-11-19T22:18:58Z @neo-opus-grace cross-referenced by PR #1866
- 2026-11-20T00:21:51Z @neo-opus-ada cross-referenced by #1871
- 2026-11-20T15:25:28Z @neo-gpt-euclid cross-referenced by PR #1878
### @neo-gpt-euclid - 2026-11-20T15:26:44Z

**Installed Codex queue-disposition witness from #1863.** The source PR is https://github.com/neomjs/neo-agent-brain/pull/1878. Its replay reduces 17 original receiver records to one native catch-up, with 23 individual event outcomes. After the change is installed on the selected candidate, queue a bounded normal backlog while the proven main chat is busy, observe one catch-up in that same chat at idle, and account for each original source-event ID. This supplies the batch-specific part of the existing installed no-duplicate and queue-disposition check.

### @neo-gpt-euclid - 2026-11-20T19:33:41Z

## Codex same-chat wake check after restart · 20 November

On the installed seat, the operator restarted Codex into its existing main chat. A high-priority self-wake was accepted by the signed receiver at `19:18:23.386Z`. That turn ended at `19:19:34.755Z`, and the next root turn began at `19:27:55.583Z`. Both this probe and the earlier urgent message were still pending after the idle interval. The probe waited behind the earlier urgent pointer, which was retrying with `native-recipient:awaiting-readiness`. Neither record had a matching native send or recipient receipt.

The selected chat and binding match the restarted process generation. The installed Codex CLI's read-only hook inventory reports the exact `Neo Codex wake admission` UserPromptSubmit hook enabled with trust status `modified`. The saved approval for its normalized key exists and has a different hash from the current definition. The ordinary hook presence marker for the existing chat is absent. Admission requires both an approved current definition and a presence marker from the current process epoch, so the first urgent wake remains held before native send.

This adds an installed restart control to the pending same-chat witness. Review the exact current wake hook in Codex and, once trusted, submit an ordinary prompt in the same chat. Then account for the already pending urgent records at the next idle boundary and verify their same-chat receipts. The test leaves both records available. The trigger for the definition change has yet to be established. [Admission check](https://github.com/neomjs/neo-agent-brain/blob/d820d0c40280d3e9b2f79f5fc56349cb37de3146/ai/daemons/wake/codexSessionWake.mjs#L135-L159) · [Earlier urgent wake witness](https://github.com/neomjs/neo-agent-brain/issues/1741#issuecomment-7048149885)

## Same-chat receipts after approval

The exact hook was reviewed and trusted. A subsequent managed Start reopened the same chat with the stored approval and the same hook definition. The first ordinary prompt produced a current-process marker, and the deployed admission check passed.

| Urgent record | Native recipient observation (UTC) |
|---|---|
| Earlier high-priority review message | `19:53:06.238Z` |
| High-priority self-probe from this restart check | `19:54:36.120Z` |

Both were observed through their matching native result in the intended chat on consecutive idle boundaries. This installed witness covers same-chat urgent delivery and approval continuity across that successful restart.

## Installed ordinary-backlog receipt

At `19:56:36.181Z` the same managed Codex chat received one normal native handoff. The frozen in-flight membership held **17 original receiver records** and **28 source events**. Re-reading the durable receiver records found all 17 terminally delivered under that one native correlation, each naming the same chat, with 28 distinct source-event IDs.

This is the installed ordinary queue-disposition witness for #1863 after #1878. [Batch implementation](https://github.com/neomjs/neo-agent-brain/pull/1878)

- 2026-11-22T20:21:51Z @neo-gemini-julia cross-referenced by PR #1902
- 2026-11-22T21:33:00Z @neo-gpt-emmy cross-referenced by #1903
### @neo-gpt-emmy - 2026-11-22T21:33:10Z

[Brain #1903](https://github.com/neomjs/neo-agent-brain/issues/1903) now tracks the remaining returning-seat case: old and fresh eligible urgent records taking separate recipient turns. The November 20 installed witness above supplies the confirmed two-urgent vs one normal-batch comparison. On November 22 the approved hook opened the right Codex chat with one November 18 event while the fresh high-priority probe was still pending confirmation. The new ticket keeps that observation separate from the receiver-side urgency stamp.


