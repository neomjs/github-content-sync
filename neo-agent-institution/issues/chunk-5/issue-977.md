---
id: 977
title: 'macOS refuses every FM notification: the bundle has no sealed identity'
state: OPEN
labels:
  - bug
  - ai
  - build
assignees:
  - neo-opus-ada
createdAt: '2026-11-15T00:54:37Z'
updatedAt: '2026-11-15T01:40:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/977'
author: neo-opus-ada
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
---
# macOS refuses every FM notification: the bundle has no sealed identity

## Context

The operator, 2026-11-15 ~00:40Z: *"i have not seen a single FM OS notification yet. maybe a permissions problem, or something is buggy."* #934 (Resolves #932, merged 2026-11-12) shipped the notifier, and its post-merge check (a merge-ready PR raises one macOS notification) has never passed. `@neo-opus-vega` captured it as a defect-note at 00:45Z.

Measured on the operator's host at about 00:55Z, read-only (`log show`, `codesign -dv`, `~/Library/Logs/Neo Harness/main.log`). Every FM launch in the window logs this, with times in UTC: 11-13 01:02Z and 16:14Z, 11-14 02:00Z, 19:08Z and 22:12Z.

```
[mjs.neo.harness] Requesting authorization with options 7
[mjs.neo.harness] Requested authorization [ didGrant: 0 hasError: 1 hasCompletionHandler: 1 ]
```

Each refusal returns within 4–9 ms, and no prompt ever appears. When work arrives, the notifier raises a notice and macOS refuses it. This happened at 00:17:32Z (neomjs/neo-agent-skills#393 went merge-ready at about 00:15Z), then at 00:23:25Z and 00:24:29Z:

```
[mjs.neo.harness] Adding notification request 3B0E-5533 to destinations: Default, contentDate: (null)
[mjs.neo.harness] Added notification request: [ hasError: 1 hasCompletionHandler: 1 ]
[mjs.neo.harness] Removing 1 delivered notifications with identifiers ( "3B0E-5533" )
```

`main.log` shows the notifier's reads answering, 29 lines since 2026-11-12 20:01Z, such as `operator notices: roster, open work, questions answer ok ok ok`. It has nothing about delivery.

The installed bundle's code identity (`CFBundleIdentifier` is `mjs.neo.harness`):

```
Identifier=Electron
CodeDirectory flags=0x20002(adhoc,linker-signed)
Info.plist=not bound
Sealed Resources=none
designated => cdhash H"469fa943d43665b157d475e61d1ed9923e0898a8"
```

## The Problem

electron-builder runs with `mac.identity: null`, so the bundle ships Electron's stock linker signature: identifier `Electron`, no sealed resources, and an unbound Info.plist. The notification service refuses authorization to that signature before any grant exists. So the app never gets an entry the operator could allow, and every request fails silently.

What is measured and what is inferred:
- **Measured:** the refusal and the signature.
- **Inferred:** that the signature causes the refusal. usernoted logs no public reason; AC-3 is the falsifier.
- **Consistent, not verified:** since 2026-10-29, the FM's own Accessibility grant has also failed with `-25211` for the same identity (#842).

The installed FM is the unsigned leg that ADR 0034 §2.5.2 reserves for CI verification. The FM has been installed and used as if it were the release, and the signed release that §2.5 plans for v1 was never built. Notifications are the first visible cost.

## The Architectural Reality

- ADR 0034 §2.5.1: *"macOS `dmg` (signed + notarized)"*. §2.7 scopes v1 to that one artifact, macOS on Apple Silicon. #12 and Epic #7 leave the signing and notarization pipeline to a release-engineering leaf that was never filed.
- `harness/electron-builder.yml`, `mac.identity: null`: the unsigned leg. `hardenedRuntime` is unset.
- electron-builder 26.15.3 (`app-builder-lib/out/mac/MacTargetHelper.js`):
  - `identity: null` skips signing, even when `CSC_NAME` is set, so the release build overrides the identity explicitly.
  - Hardened runtime defaults on for a signed non-MAS build.
  - Notarization reads `APPLE_KEYCHAIN_PROFILE`, a `notarytool store-credentials` profile, so the key stays in the operator's keychain.
- `harness/operatorNotifications.mjs`, `createNoticePresenter`: calls `new Notification(...).show()` and listens only for `click` and `close`. Electron 43 also emits `failed` on darwin, as `(event, error: string)`, and nothing listens for it.
- `harness/main.mjs`, `startOperatorNotifier`: gated on `Notification.isSupported()`, which is true on macOS whether or not the app is authorized.
- The FM keeps its PATs through Electron `safeStorage`, which uses the keychain item `Neo Harness Safe Storage`. That item's access follows the app's designated requirement.
- Electron's macOS autoUpdater (Squirrel.Mac) runs only on a code-signed app, so the E7 update channel (ADR 0034 §2.5.3) needs the same signature.

Design authority: ADR 0034 §2.5.2, *"cert provisioning/notarization credentials are operator-owned (human-only, like the merge gate); CI produces unsigned artifacts for verification, the release pipeline signs. No leaf embeds signing material in repo tooling."* It sits beside #932's accepted outcome, *"An OS notification tells the operator a merge or a question waits"*.

## The Fix

1. The presenter reports a refused delivery: it listens for Electron's `failed` event and writes one `HARNESS_OPERATOR_NOTICES delivery failed: <error>` line per distinct error, so a refusal reaches `main.log`.
2. The release build signs and notarizes with the publisher's **Developer ID Application** certificate. It turns on the hardened runtime with Electron's entitlements, and every Mach-O in the bundle is signed, the organism's native binaries included. Notarization runs through the keychain profile. Every operator's install then carries one stable identity (team ID + `mjs.neo.harness`), so:
   - notification authorization works;
   - keychain access and privacy grants survive updates;
   - Gatekeeper opens a download;
   - E7's autoUpdater can run.

   CI and dev builds stay unsigned. The release build gets only two names, both from the operator's config: the certificate's and the notarization profile's. No secret enters the repo (§2.5.2).

### Implementation notes (read-only survey, 2026-11-15 ~01:30Z, before the certificate exists)

- **Mach-O in the installed 968 MB bundle:**
  - the main executable, four helper apps, Electron Framework (6), Squirrel (2), ReactiveObjC and Mantle;
  - nine native binaries under `Resources/organism/node_modules`: `chromadb-js-bindings` (darwin-arm64), `better-sqlite3` prebuilds (darwin-arm64 and darwin-x64), `@img/sharp-darwin-arm64` and its `libvips-cpp` dylib, and `onnxruntime-node` (arm64 and x64, each a binding plus a dylib).
  
  All carry ad-hoc linker signatures. Notarization rejects any Mach-O that is unsigned or lacks the hardened runtime, so the deep signing must reach `Resources`. The two x64 prebuilds are dead weight on an arm64-only v1; pruning them at pack time is optional.
- **Children run under the hardened runtime.** The FM runs its own executable as Node (`ELECTRON_RUN_AS_NODE=1`) for:
  - the Fleet child (`main.mjs`);
  - the runtime probes (`runtimeGeneration.mjs`);
  - the `node` shim that seats' MCP servers start through (`pack.mjs:755`).
  
  All of them inherit the app's entitlements. Consequences:
  - V8 needs `com.apple.security.cs.allow-jit`.
  - Library validation passes while every native module carries the same team signature. So start without `disable-library-validation`, and add it only if a load measurably fails.
- **Entitlements:** a tailored `harness/entitlements.mac.plist`. The defaults are too broad: `@electron/osx-sign`'s `default.darwin.plist` adds camera, microphone, Bluetooth, USB, print and location, and electron-builder's template adds `allow-unsigned-executable-memory` and `disable-library-validation`. If any harness path sends Apple Events (`osascript`), it also needs `com.apple.security.automation.apple-events` plus an `NSAppleEventsUsageDescription`. Check that before the first signed build.
- **The release invocation:** `electron-builder --mac --publish never -c.mac.identity="<certificate name>"`, with the hardened runtime and the entitlements set, and `APPLE_KEYCHAIN_PROFILE=<profile>`. The default `identity: null` keeps CI unsigned. Notarize and staple the `.app` before it is zipped.
- **Verification:**
  - `codesign --verify --deep --strict --verbose=2`;
  - `codesign -dv` (AC-2);
  - `spctl -a -vv -t exec` (AC-2);
  - `xcrun stapler validate` on the `.app`;
  - then a launch smoke before the install.

Rejected and deferred alternatives:
- **A self-signed certificate per operator:** rejected. The operator, 2026-11-15: *"if FM aims to become a product which many operators use, what makes sense then? asking each one to manually create a cert on their own probably not."*
- **An ad-hoc seal (`mac.identity: "-"`):** not shipped by default. It would seal an identity without any certificate, but its designated requirement moves with every build, so the keychain is expected to re-prompt after every FM update. It stays only as an optional bridge for the operator's own machine until the Developer ID exists.

**Blocked on:** the publisher's Developer ID. The operator confirmed 2026-11-15 that he will enroll and set it up in the afternoon (his time), not at night. The one-time setup, all of it his because these are credentials (§2.5.2):

1. Enroll in the Apple Developer Program ($99/year). An organization account shows the legal entity as publisher and needs a D-U-N-S number; an individual account is quickest.
2. Create the **Developer ID Application** certificate: Xcode → Settings → Accounts → the team → Manage Certificates → + → Developer ID Application. Only the Account Holder can. Its private key stays in the login keychain.
3. Create a notarization key: App Store Connect → Users and Access → Integrations → App Store Connect API → **Team Keys** (personal keys do not work with notarytool) → +, with the Developer role. Download the `.p8` (Apple allows one download) and note the Key ID and the Issuer ID.
4. Store it as a keychain profile: `xcrun notarytool store-credentials neo-harness-notary --key AuthKey_<KEYID>.p8 --key-id <KEYID> --issuer <ISSUER>`, then confirm it with `xcrun notarytool history -p neo-harness-notary`.
5. Hand over the certificate's name as Keychain shows it (`Developer ID Application: <Name> (<TEAMID>)`) and the profile name. Neither is a secret.

Fix 1 is done on branch `ada/977-release-signing` (`ea8f436`). Its new spec fails on `dev` and passes at the branch. The PR opens once a signed build verifies.

## Acceptance Criteria

- [ ] AC-1: A refused notification writes one `delivery failed` line with Electron's error (unit spec: a fake `Notification` that emits `failed`).
- [ ] AC-2: The release artifact verifies: `codesign -dv` reports `Identifier=mjs.neo.harness`, `Authority=Developer ID Application: …` and a `TeamIdentifier`, and `spctl -a -vv` reports `accepted, source=Notarized Developer ID`. CI artifacts stay unsigned.
- [ ] AC-3 (live; the operator must be present): the signed FM's first launch shows macOS's notification prompt. Once allowed, it logs `Requested authorization [ didGrant: 1 hasError: 0 ]`, and the next merge-ready PR raises a notification the operator sees (#934's unticked check).
- [ ] AC-4: The first signed install asks for the keychain item and the privacy grants once, because the identity changed. The next FM update asks for neither. Measured and recorded here.

## Out of Scope

- The E7 update channel itself (autoUpdater wiring, ADR 0034 §2.5.3). This ticket only gives it the signature it needs.
- Windows and Linux signing (ADR 0034 §2.7, each a later, demand-gated leaf).
- The operator's idea, verbatim: *"i guess a 'silent mode' toggle button might make more sense, to temporarily enable or disable OS level notifications."* It needs working delivery first, so it gets its own ticket once AC-3 holds.
- The `-25211` Accessibility path: typing belongs to the wake receiver (#842).

## Avoided Traps

- **Asking the operator to flip a permission:** there is no entry to flip, because the refusal happens before any grant.
- **Trusting `Notification.isSupported()`:** it is true for an app that macOS will refuse.
- **Probing with a stock `Electron.app`:** it has another bundle id and another identity. Only the installed bundle answers the question.
- **A per-operator certificate:** each operator would carry a credential chore, and it still fails Gatekeeper and notarization for anyone who downloads the app.
- **Setting `CSC_NAME` alone:** with `identity: null` in the config, electron-builder still skips signing.
- **The stock entitlement sets:** they grant hardware and privacy access the FM never uses.

## Related

#932 / #934 (the notifier) · #842 (`-25211` on the same identity) · #12 (the signing pipeline is out of its scope) · #732 (updates without closing a harness) · Epic #7 (shell packaging; already at 30 subs, so not linked as one)

## Sweeps

- Live latest-open + closed sweep at 2026-11-15T00:52Z, re-run at 00:54Z: the latest 20 open issues, plus open and closed searches for `notification OR codesign OR signing OR signature OR 25211`. I read Out of Scope on #354 and #12. No equivalent exists; #12 leaves signing to a release-engineering leaf that has not been filed.
- Re-scope sweep, 01:08Z: `notarize OR notarization OR "Developer ID" OR autoUpdater OR "auto-update"`. No signing-pipeline ticket exists; this ticket becomes that leaf.
- Backlog: 57 open (red), with no owning ticket to fold into.
- A2A: the latest 30 messages hold no claim on this surface; Vega routed it to the notifier's owner.
- MC: one problem-noun query found no prior note.
- Own assignments: #516 and #424, neither on this surface.
- Structure map: n/a. This is `harness/` packaging with no `ai/` surface.

Decision Record impact: `aligned-with adr:0034` (§2.5.1 signed + notarized dmg; §2.5.2 operator-owned credentials and unsigned CI).

Origin Session ID: 969c301f-16ab-43e0-b26f-c030f272320b

Retrieval Hint: `query_raw_memories("FM notification authorization refused linker-signed Electron identity Developer ID")`

Authored by ⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-15T00:54:37Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-15T00:54:39Z @neo-opus-ada added the `bug` label
- 2026-11-15T00:54:39Z @neo-opus-ada added the `ai` label
- 2026-11-15T00:54:39Z @neo-opus-ada added the `build` label
- 2026-11-15T00:55:09Z @neo-opus-ada cross-referenced by PR #934
- 2026-11-15T01:02:19Z @neo-opus-ada cross-referenced by #928
### @neo-opus-ada - 2026-11-15T01:40:56Z

**Handover (session sunset, 2026-11-15 ~01:45Z): Ada's next session picks this up in the operator's afternoon.**

- **State:** Fix 1 is committed on `ada/977-release-signing` @ `ea8f436` (pushed). Its new spec fails on `dev` and passes at the branch. Fix 2 is unbuilt; the body's *Implementation notes* list the binaries, the child processes, the entitlements and the commands.
- **Pickup:** the operator runs the five setup steps in the body and hands over the certificate name and the notarization profile name. Then: the tailored entitlements plist, the release invocation, the deep verify (AC-2), a launch smoke, and the install with the operator present (AC-3, AC-4). Open the PR only once AC-2 passes.
- **Owner:** `@neo-opus-ada`.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


