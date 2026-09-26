---
id: 259
title: Document how operators update an installed Fleet Manager
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - build
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-26T20:46:07Z'
updatedAt: '2026-09-26T21:26:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/259'
author: neo-gpt-emmy
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
closedAt: '2026-09-26T21:26:51Z'
---
# Document how operators update an installed Fleet Manager

## Context

The operator asked how the installed Electron shell gets updated and expected automatic updates for new packaged releases. On 2026-09-26 the installed `0.0.1` bundle was built at 08:57 UTC and still contained dummy roster/activity seeds after their removal merged at 13:43. Reopening the app did not deliver the new code.

The current update work and producer defects are separated in [the delivery map on #7](https://github.com/neomjs/neo-agent-institution/issues/7#issuecomment-5849670470).

## The Problem

`harness/README.md` explains packaging for contributors, but gives an operator no installed-update procedure. `harness/electron-builder.yml` explicitly has no publish provider or updater metadata, and the releases API is empty. A merged change, a new checkout, a rebuilt artifact and a replaced running application are four distinct steps.

## The Architectural Reality

- `harness/package.json` owns the Electron package and `npm run dist`; `pack.mjs` stages Product, pinned Engine and an explicit Brain root.
- `organism-build-info.json` records Engine pin, Brain revision and build time. The current product entry records name/version, not its Git revision; a maintainer must retain the source revision alongside the artifact.
- `appLifecycle.mjs` hides the window on ordinary close; the explicit Quit action retires owned children. Closing the window is not quitting the shell.
- `planeConfig.mjs` keeps the plane record and encrypted bearer under Electron userData, outside the app bundle. Updating must preserve that directory and the existing principal.
- ADR 0034 §2.5 requires whole-package updates; signing and the E7 updater channel remain separate release work.

## The Fix

Add an operator-facing **Updating an installed app** section to `harness/README.md`, linked from the root README's shell section:

1. State the present manual-update status without implying a release feed exists.
2. Explain obtain the replacement artifact from the maintainer/release channel, explicitly quit, replace the whole app bundle, reopen, and verify the intended build and plane connection.
3. Preserve userData, credentials and the old bundle for a possible rollback; never copy selected renderer/Brain files into an installed app.
4. Keep source installation, explicit Product/Engine/Brain build provenance and smoke verification in a clearly identified maintainer subsection.
5. Separate installation success from live-source success: honest empty/unavailable states are valid observations; fixture rows do not prove a live connection. Link the existing E7 requirement for automatic updates.

## Acceptance Criteria

- [ ] An operator can find the installed-update instructions from the root README without reading the packaging implementation.
- [ ] Instructions distinguish window close from explicit Quit, whole-bundle replacement from partial updates, and persistent userData from application resources.
- [ ] The manual path names how to verify the artifact and saved plane after relaunch, including the current build-metadata limitation.
- [ ] The maintainer path uses the existing explicit Brain-root packaging command and records source revisions; it does not invent an installer command or claim an unsigned build auto-updates.
- [ ] The PR cites the actual update receipt from today's rebuilt shell, with any remaining live-data defects named separately.

## Decision Record impact

`aligned-with ADR 0034 §2.5`. Documentation of the current update path; no new credential, process, persistence or updater API.

## Out of Scope

Implementing E7's signed release/update feed; altering authentication or the saved viewer; fixing Brain #459/#53; task sample removal (#240); whole harness demotion (#17).

## Avoided Traps

Telling operators to run npm; treating a repository merge as installation; deleting userData to get a clean-looking boot; bypassing macOS security prompts; calling a sample roster a live-data receipt.

## Related and intake

Parent #7. #214 is the fixture-plane smoke, #178 the broader learning tree, and #257 the polling change; none documents this installed-update path. Epic-review cap is already satisfied by [Euclid](https://github.com/neomjs/neo-agent-institution/issues/7#issuecomment-5438105371) and [Phoebe](https://github.com/neomjs/neo-agent-institution/issues/7#issuecomment-5438105479); no third structured review is added. Ada confirms Emmy is the artifact owner while Clio is away.

Freshness: latest 20 open issues and latest 30 A2A messages across read states checked immediately before filing on 2026-09-26; no equivalent claim. Own open Institution assignments: none. MC problem-noun sweep returned unrelated material; live README, lifecycle, plane-config and packaging sources establish the gap.

Origin Session ID: 01a0deee-3f9b-7180-ac35-f90129ccaa40

Retrieval Hint: `installed Fleet Manager update whole bundle userData planeConfig organism-build-info`.


## Timeline

- 2026-09-26T20:46:07Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-26T20:46:08Z @neo-gpt-emmy added the `documentation` label
- 2026-09-26T20:46:08Z @neo-gpt-emmy added the `enhancement` label
- 2026-09-26T20:46:09Z @neo-gpt-emmy added the `ai` label
- 2026-09-26T20:46:09Z @neo-gpt-emmy added the `build` label
- 2026-09-26T20:46:27Z @neo-gpt-emmy added parent issue #7
- 2026-09-26T20:57:17Z @neo-gpt-emmy cross-referenced by #7
- 2026-09-26T20:57:47Z @neo-gpt-emmy cross-referenced by PR #260
- 2026-09-26T21:04:30Z @neo-opus-grace cross-referenced by #261
- 2026-09-26T21:26:51Z @tobiu referenced in commit `f865693` - "Merge pull request #260 from neomjs/codex/259-installed-fm-updates

docs(harness): explain installed Fleet Manager updates (#259)"
- 2026-09-26T21:26:51Z @tobiu closed this issue
- 2026-09-26T21:31:43Z @neo-gpt-emmy cross-referenced by #10

