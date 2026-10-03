---
id: 473
title: The install leg replaces Neo Harness in place and keeps one rollback
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-10-03T06:33:28Z'
updatedAt: '2026-10-03T08:46:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/473'
author: neo-opus-vega
commentsCount: 0
parentIssue: 7
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-03T08:46:10Z'
---
# The install leg replaces Neo Harness in place and keeps one rollback

## Context

The operator's standing complaint (2026-10-03): every refresh of the installed Fleet Manager drops another copy into `/Applications` instead of updating the app in use. Read at 06:26Z today: `/Applications` held the live `Neo Harness.app` (receipt `stagedAt 2026-10-01T14:21Z`, Brain `741f9f3`, Engine `e7d550e`) beside **eleven** `Neo Harness.previous-<date>-<tag>.app` siblings, dated Sep 25 → Sep 30. Emmy archived those eleven outside `/Applications` at ~06:36Z as part of the #12 cut (no deletion); the folder is clean now, and this ticket replaces the mechanism that filled it. The harness README's [Updating an installed app](https://github.com/neomjs/neo-agent-institution/blob/dev/harness/README.md#updating-an-installed-app) section prescribes a hand-copy: "keep the previous bundle … replace the whole app in Applications". There is no tooling behind it, so each maintainer invents a backup name, and the backups land in the one folder they should not.

## The Problem

The packaging pipeline ends at `npm run dist` (`harness/electron-builder.yml`, output `dist-artifacts/mac-arm64/Neo Harness.app`). The last mile — stop, replace, keep a way back, prove what is now installed — is prose. Two consequences:

1. **Clutter is a symptom of an unbounded backup policy.** "Keep the previous bundle" has no retirement rule, so the folder grows by one launchable app per refresh. Launchpad and Spotlight offer all twelve.
2. **Nothing distinguishes the builds.** The shell version is `0.0.1` across development builds and the bundle identifier is constant (`mjs.neo.harness`), so the only truth about what is installed is `Contents/Resources/organism/organism-build-info.json` (`stagedAt`, Brain revision, Engine pin, product version). A hand-copy never prints it.

Quitting the app also stops the peer harnesses it launched (README, same section), so a replacement that kills the running process silently is unsafe.

## The Architectural Reality

- `harness/pack.mjs` and `harness/prepareAssets.mjs` are the maintainer-side packaging scripts; `harness/package.json` carries the `predist` / `dist` legs. The install leg is the next script in that row, nothing more.
- ADR 0034 §2.5.3: the organism "updates by shipping a new package … no self-mutating installed app in v1". Replacing the **whole bundle** is exactly that contract; partial in-place organism updates stay deferred.
- ADR 0034 §2.5.2: signing material never enters repo tooling. The leg installs the unsigned development artifact only; the stranger's path (signed installer + E7 update channel) is untouched.
- `app.getPath('userData')` is `~/Library/Application Support/neo-harness/`; it holds the plane record (`brain/`) and encrypted credentials, and must survive an update untouched (README step 4).
- `/Applications`, `~/Library` and the checkout live on one APFS volume (device `16777232` on the reference machine), so a bundle move is a `rename(2)`: atomic and instant.

## The Fix

`harness/install.mjs`, exposed as `npm run install:mac` (a custom script name; npm's `install` lifecycle hook is not touched). Pure planner + thin executor:

1. **Plan** (`planInstall({mode, artifact, installed, rollback, parked, running, flags, custodyDir})` — pure, returns ordered steps or a refusal): the process census reads every executable inside a bundle named `Neo Harness….app` (canonical, a renamed `previous-*` copy, a dist copy); any path outside the canonical bundle is refused by path, with or without `--quit`; a canonical process is refused unless `--quit` is passed; equal receipts are an empty plan (exit 0).
2. **Execute:** with `--quit`, ask the running app to quit (`osascript … tell application "/Applications/Neo Harness.app" to quit` — by path: every build shares the identifier, and 17 copies were registered on the reference machine) and wait for the processes to exit; hash the custody files (`<userData>/brain/fleet`) once the app is down; copy the artifact (`ditto`) to `Neo Harness.app.installing` beside the destination and **verify its receipt there, before any slot moves**; then move the current bundle to the fixed rollback path `~/Library/Application Support/neo-harness/rollback/Neo Harness.rollback`, replacing whatever was there (**exactly one** rollback, never a sibling in `/Applications`, and never a registered copy: the slot's name carries no `.app` suffix), and rename the staged copy into place.
3. **Receipt:** print old → new `stagedAt`, Brain revision, Engine pin, product version, and the rollback path; verify the installed receipt equals the artifact's; compare the custody hash **before** any `--open`, and stop with the app down when it differs. Exit non-zero on either.
4. `--restore` swaps the two slots through `Neo Harness.app.restoring` (three renames) and prints the same receipt pair. A run that finds a `.restoring` bundle completes the interrupted swap toward the empty slot and stops (both slots full beside it is a refusal); the leg never treats an interrupted state as an absence. `--open` relaunches the canonical app last.
5. README: the manual steps become the command, the warning about quitting stays, and the section names the rollback path.
6. Unit spec `test/playwright/unit/harness/install.spec.mjs` over the planner (sibling precedent: `planeConfig.spec.mjs`): refusal while running, refusal on a non-canonical running path, the single-rollback replacement, receipt mismatch as a failure, `--restore` symmetry.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `npm run install:mac` (`harness/install.mjs`) | `harness/package.json` scripts; sibling `dist` leg | Replace `/Applications/Neo Harness.app` whole with the one bundle under `dist-artifacts/mac*/`; displaced bundle → the single rollback slot | `--artifact <bundle.app>` names the bundle; 0 or >1 candidates refuse | `harness/README.md` "Updating an installed app" | `install.spec.mjs` planner + executor arms; live run deferred (see ACs) |
| Flags `--quit` · `--open` · `--restore` · `--dry-run` | `parseInstallArgs` | Only `--quit` stops a running app (orderly quit addressed to the canonical bundle by path, never a signal, never the shared identifier); `--open` relaunches last and never after a failed custody comparison; `--restore` swaps the slots; `--dry-run` prints the plan and changes nothing | unknown flag → usage, exit 2 | README | planner arms (step order), CLI usage |
| Running-app refusals | `runningHarnessPaths` census (`ps -axo comm=`, bundle pattern `Neo Harness….app`) → `planInstall` | canonical process without `--quit` → `running` (exit 1, peer warning printed); any process outside the canonical bundle → `running-off-canonical` naming the paths, with or without `--quit` | none; the refusal is the contract | README | census arm through the real reader→planner path |
| Receipt identity | `Contents/Resources/organism/organism-build-info.json` (written by `pack.mjs`) | equal installed/artifact receipts → empty plan, exit 0; staged copy verified before any slot moves; installed verified after; mismatch → exit 1 | no receipt in the artifact → `artifact-receipt-missing` | README (version label cannot prove an update) | staged-verify arm (both slots untouched on a corrupt copy) |
| Rollback + recovery | the two slots + `Neo Harness.app.restoring` / `.installing` siblings | exactly one rollback at the fixed path, named `Neo Harness.rollback` so Launch Services never registers it; a `.restoring` bundle found at run time is completed toward the empty slot and the run stops; both slots full beside it → `interrupted-restore-ambiguous` | a leftover `.installing` is re-staged | README | interruption arms at both rename boundaries; repeated-restore arm |
| Custody preservation | `<userData>/brain/fleet` (README step on store survival) | hashed after the app is down and again after the slots moved, before `--open`; a difference stops the run with the app down | custody dir absent → the comparison is `absent`, not a failure | README (scoped claim: never written; read twice to hash) | shutdown-write vs installer-write arms |

Structural pre-flight: sibling-file lift — `harness/*.mjs` maintainer scripts (`pack.mjs`, `prepareAssets.mjs`). Structure map (Brain-hosted `ai:structure-map`): N/A, Institution packaging root; no Brain placement.

## Acceptance Criteria

- [ ] `npm run install:mac` from `harness/` replaces `/Applications/Neo Harness.app` with `dist-artifacts/mac-arm64/Neo Harness.app` and leaves **no** new `.app` in `/Applications`; the previous bundle is at the fixed rollback path, and a second install replaces that rollback rather than adding one. — Unit: the executor on real temp directories (install → restore → restore). **Live `/Applications` run: `[L3-deferred — the installed cut is #12's boundary]`, Residual-Owner: #12, both install AND restore.**
- [ ] While a Neo Harness process runs, the command refuses with the running path named and changes nothing; `--quit` is the only way it stops the app, and the README warning about launched peers is printed before it does.
- [ ] A running Neo Harness whose executable is not under `/Applications/Neo Harness.app` is refused by path, with or without `--quit`.
- [ ] After a successful install the printed receipt shows old and new `stagedAt`, Brain revision, Engine pin and product version, and the installed `organism-build-info.json` equals the artifact's; a mismatch exits non-zero.
- [ ] `--restore` swaps the rollback back into `/Applications` and prints the same receipt pair. — Unit: the restore and repeated-restore arms, plus both interruption boundaries. **Live run: `[L3-deferred]` with AC-1, Residual-Owner: #12.**
- [ ] `<userData>/brain/fleet` is byte-identical between the app's shutdown and the moment before `--open` (hashed twice; a difference stops the run and nothing relaunches).
- [ ] The spec covers the five cases in step 6, the census through the real reader→planner path, both restore interruption boundaries, a corrupt staged copy, and the two custody timings, and runs in the Isolated Institution CI job.
- [ ] `harness/README.md` "Updating an installed app" names the command, the rollback path and `--restore`; the hand-copy steps are gone.

## Out of Scope

- The archived `Neo Harness.previous-*.app` bundles (Emmy moved them out of `/Applications` under #12). They are historical rollback material the operator retires; the command only lists any such sibling it still finds and never deletes anything it did not place.
- The signed/notarized installer and the autoUpdater channel (E7, ADR 0034 §2.5.3).
- Today's live installed cut (#12, Emmy): this ticket ships the tool the **next** cut uses.
- Windows/Linux legs; the Agent OS container refresh.

## Avoided Traps

- **Rollback as another `/Applications` sibling** — that is the clutter. Fixed path outside the folder, one slot.
- **Trusting the version label** — `0.0.1` and `mjs.neo.harness` are constant across builds; the receipt is the only identity (Euclid's census, 06:29Z).
- **Killing the process** — quitting stops launched peer harnesses; refuse by default, `--quit` opt-in, print the warning.
- **Copy-into-the-old-bundle** — README already forbids partial file copies; the leg only ever moves whole bundles.
- **A `cp -R` of 340 MB with the destination live** — stage beside, then rename; the live path is never half-written.

## Related

Parent: #7 (Electron shell epic; this is the local-install leaf beside E6 packaging). #259 (closed) wrote the manual procedure this replaces. #12 (the live cut this tool serves next time). ADR 0034 §2.5 (`neomjs/neo` `learn/agentos/decisions/0034-electron-shell-architecture.md`).

Decision Record impact: aligned-with ADR 0034 §2.5.2 (unsigned leg, no signing material) and §2.5.3 (whole-package replacement, no partial in-place organism update).

Live latest-open sweep: latest 20 open Institution issues read at 2026-10-03T06:30:54Z; no equivalent. Org search "harness install Applications rollback": none. A2A sweep (30 rows, all read-states): Clio proposed the same command after my 06:27:34Z claim (`MESSAGE:08f397e8`); first-claim tiebreak — one owner, packet sent to her. Memory Core sweep on the symptom returned no prior decision; the README section is the prior art. Own-assignment sweep: no open Institution assignments.

Origin Session ID: 075e6b2a-b93a-4972-b143-0fca9e7c06d8
Retrieval Hint: "Neo Harness install:mac rollback outside Applications receipt"



## Timeline

- 2026-10-03T06:33:29Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-03T06:33:30Z @neo-opus-vega added the `enhancement` label
- 2026-10-03T06:33:30Z @neo-opus-vega added the `ai` label
- 2026-10-03T06:33:31Z @neo-opus-vega added the `build` label
- 2026-10-03T06:33:37Z @neo-opus-vega added parent issue #7
- 2026-10-03T06:39:35Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-03T06:43:06Z @neo-opus-vega cross-referenced by PR #474
- 2026-10-03T06:44:59Z @neo-opus-vega referenced in commit `0c2837f` - "docs(harness): the install leg's header describes behavior, not the ADR coordinate (#473)"
- 2026-10-03T07:04:31Z @neo-fable cross-referenced by #475
- 2026-10-03T07:17:53Z @neo-opus-vega referenced in commit `b2945f5` - "docs(roadmap): row 5's install-and-update anchor names the install leg (#473)"
- 2026-10-03T07:30:30Z @neo-opus-vega referenced in commit `e058db4` - "fix(harness): the install leg's census sees every Neo Harness bundle, verifies the staged copy before any slot moves, completes an interrupted swap, and compares custody before relaunch (#473)"
- 2026-10-03T07:50:21Z @neo-opus-vega referenced in commit `03e845f` - "fix(harness): the install leg quits the canonical bundle by path and keeps the rollback slot unregistered (#473)"
- 2026-10-03T08:46:10Z @tobiu referenced in commit `ce90152` - "feat(harness): the install leg replaces Neo Harness in place and keeps one rollback (#473) (#474)

* feat(harness): the install leg replaces Neo Harness in place and keeps one rollback outside Applications (#473)

* docs(harness): the install leg's header describes behavior, not the ADR coordinate (#473)

* docs(roadmap): row 5's install-and-update anchor names the install leg (#473)

* fix(harness): the install leg's census sees every Neo Harness bundle, verifies the staged copy before any slot moves, completes an interrupted swap, and compares custody before relaunch (#473)

* fix(harness): the install leg quits the canonical bundle by path and keeps the rollback slot unregistered (#473)"
- 2026-10-03T08:46:10Z @tobiu closed this issue
- 2026-10-03T09:53:03Z @neo-opus-vega cross-referenced by #495
- 2026-10-03T10:05:52Z @neo-gpt-emmy cross-referenced by PR #496

