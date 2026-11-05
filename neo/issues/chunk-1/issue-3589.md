---
id: 3589
title: buildScripts/createTheme
state: OPEN
labels:
  - enhancement
  - developer-experience
  - ai
  - build
assignees:
  - neo-gpt-euclid
createdAt: '2022-12-08T17:39:53Z'
updatedAt: '2026-11-05T18:17:39Z'
githubUrl: 'https://github.com/neomjs/neo/issues/3589'
author: tobiu
commentsCount: 4
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
# buildScripts/createTheme

especially important for the workspace scope, since we can now create entire themes there.

it should be possible to select the base theme which you want to extend.

adding files as needed and adjusting the root for the css vars selector.

---

## Implementation intake — Euclid, 2026-11-05

The original request above remains unchanged. This section records the implementation contract derived from the current source.

**Classification: valid-as-written.** Created 2022-12-08; last premise read 2026-11-05. The current `mark-inactive-issues.yml` marks at 90 days and never closes; this ticket has neither `stale` nor `no auto close`. Recent activity puts it in the pre-stale automation band, independently of the source check. `buildScripts/create/` still has no theme generator at dev `d556702afb`. Closed #3973 explicitly duplicates this ticket; no open blocker, parent epic, or generator successor was found. Three Memory Core framings found no relevant prior implementation; Knowledge Base and the live source supplied the anchors. ADR successor-risk: no-adr-impact; this adds a build-time generator within the existing theme contract.

Triaged per `ticket-triage`: existing `enhancement` retained; added `ai`, `build`, and `developer-experience`. Premise, prescription, placement, developer consumer, service boundary, and decision-record checks passed.

Prescription checked: `buildScripts/create/` owns scaffolding; `buildScripts/build/themes.mjs` owns compilation and source precedence. `learn/guides/uibuildingblocks/StylingAndTheming.md` sections 6–10, `src/worker/App.mjs#insertThemeFiles`, and `src/main/addon/Stylesheet.mjs#addThemeFiles` define the consumer contract. A custom theme requires its own matching value sheets; naming a base theme in application config does not retarget that base's selectors.

### Contract Ledger (intake-owned)

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| New `buildScripts/create/theme.mjs` and `create-theme` command | Original request; existing create-script family | Developer chooses a new theme name and an existing base, interactively or with explicit flags; output stays under the current workspace's `resources/scss` | Reject invalid names, missing bases, source/destination identity and existing destinations before writing; never overwrite an installed or existing theme | CLI help and build-script usage | CLI success and refusal tests; source tree unchanged on refusal |
| Generated theme SCSS | Current build source order: Engine, declared component packages, workspace | Snapshot the selected base's effective files, retaining internal partials and overriding matching paths from later roots; replace the exact base theme class with the new class without changing selector specificity | Keep shared Sass dependencies resolvable from generated files, including `Global.scss`'s `../motion`; do not overwrite workspace shared partials | Usage explains generated editable source and inherited dependencies | Physical installed-workspace fixture; sparse local overlay; real Sass/theme build |
| Existing `buildScripts/build/themes.mjs` output and theme map | Existing builder and Stylesheet admission | Normal build discovers the new folder; `theme-acme` is configured as `neo-theme-acme`; generated Global and component CSS contain the new selector and inherited values | Generator leaves application theme selection to the developer and reports the exact build/config steps | Build-script usage and existing theming guide | Compiled Global/button CSS and generated `theme-map.json` membership; all shipped base themes exercised |

### Acceptance record

| Consumer effect | Expected | Observed | State / next proof |
| --- | --- | --- | --- |
| Create a workspace theme from a selected existing base | Complete editable source with matching selectors and resolvable dependencies | No generator exists on the inspected dev base | Open — implement and exercise the CLI |
| Build and use the generated theme | Normal compiled files and theme-map membership for its Global/component paths | Not yet run | Open — physical workspace build, then inspect runtime loading/appearance within the evidence available |
| Preserve existing work on refusal | Existing source and destination remain byte-identical | Not yet run | Open — invalid/conflicting destination controls |

No new runtime inheritance layer, theme registry, automatic application config mutation, or replacement ticket is part of this work.


## Timeline

- 2022-12-08T17:39:53Z @tobiu added the `enhancement` label
### @github-actions - 2024-08-30T02:27:41Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-30T02:27:41Z @github-actions added the `stale` label
### @github-actions - 2024-09-14T02:26:44Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-14T02:26:44Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:23:49Z

#19489 set C · T4 · Grace · 2026-10-09: **reopened, valid but later.** There is still no theme generator in `buildScripts/create/`.

- 2026-10-09T20:26:18Z @neo-opus-grace cross-referenced by #19489
- 2026-10-09T20:33:42Z @neo-opus-grace cross-referenced by #3973
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-julia - 2026-11-05T10:58:19Z

### Premise read & verdict (`neomjs/neo#15000` · 2026-11-05 · `@neo-gemini-julia`)

- **Verdict:** **holds (operator-authored)**
- **Premise check against current source (`dev@14d6326fc1`):**
  - `buildScripts/create/` contains `addConfig.mjs`, `app.mjs`, `appMinimal.mjs`, `class.mjs`, `component.mjs`, `example.mjs`, and `scss.mjs`, while theme build and lint tooling lives in `buildScripts/build/themes.mjs`, `buildScripts/helpers/watchThemes.mjs`, `buildScripts/util/check-theme-coverage.mjs`, `buildScripts/util/check-theme-surfaces.mjs`, and `buildScripts/util/check-theme-value-files.mjs`.
  - There is still no `buildScripts/create/theme.mjs` scaffolding script for creating a new theme extending one of the 5 shipped themes (`neo-theme-light`, `neo-theme-dark`, `neo-theme-neo-light`, `neo-theme-neo-dark`, `neo-theme-cyberpunk`).
- **Recommendation:** Premise still holds; leaving open for `@tobiu`.

🏛️ Julia (`@neo-gemini-julia`)

- 2026-11-05T18:42:29Z @neo-gpt-euclid cross-referenced by PR #20365

