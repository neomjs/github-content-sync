---
id: 657
title: Remediate Institution's seven dependency-security alerts
state: CLOSED
labels:
  - bug
  - ai
  - build
  - dependencies
  - security
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-10T14:05:18Z'
updatedAt: '2026-11-12T12:56:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/657'
author: neo-gpt-euclid
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
blocking: []
closedAt: '2026-11-12T12:56:50Z'
---
# Remediate Institution's seven dependency-security alerts

## Context

The operator escalated the seven open [Institution Dependabot alerts](https://github.com/neomjs/neo-agent-institution/security/dependabot) on 2026-10-10. At `dev af1b85d62228a160e1c1185d0a36fbf6e6c82d29`, seven manifest-specific records cover five npm packages and six advisories. They postdate the repaired cohort in closed `#124` / `#149`; those tickets stay closed.

## Current intake — 2026-11-12

The original seven IDs below remain the close-target cohort. Live GitHub reads at Institution dev `72dcd27e` now show:

| Original alerts | Current disposition / next action |
| --- | --- |
| 78 (Sharp) | Open; lock still has 0.35.4. Raise the existing Transformers override floor to ^0.35.5 and refresh Sharp/platform packages. |
| 75, 67 (DOMPurify) | Open; Monaco pins 3.4.15 while Mermaid already resolves 3.4.17. Qualify a root ^3.4.17 override, remove the duplicate nested resolution, and verify the rebuilt Monaco sanitizer. This also addresses the newer DOMPurify advisories 87–89; stopping at the old 3.4.16 floor is insufficient. |
| 74 (KaTeX) | GitHub records fixed at 2026-10-16T13:42:06Z; current root lock is 0.18.10. Preserve the existing repair. |
| 69 (braces) | GitHub returns 410 withdrawn. GHSA-vfj7-8cjw-p6xm was withdrawn 2026-10-23T12:53:02Z. Record as withdrawn, not patched or application-safe. |
| 76, 70 (sprintf-js) | GitHub returns 410 withdrawn. GHSA-hp3w-g68c-fv3c was withdrawn 2026-10-22T19:21:04Z. Record as withdrawn, not patched or application-safe. |

New harness alert 105 (minimatch 9/10, patched floor 11.0.1) now has a qualified repair in this same dependency-verification lane. Latest compatible electron-builder 26.18.0 still requires minimatch ^10.2.5 and pins @electron/universal 2.0.3, so a parent update alone does not fix it. The candidate overrides only app-builder-lib@26.15.3's minimatch@^10.2.5 and @electron/universal@2.0.3's minimatch@^9.0.3 to the independently tested exact 11.0.1. Actual FileMatcher, Universal ASAR and architecture allow-list consumers pass 63 checks on both the original and actual installed candidate graph; legacy callable minimatch 3/5 consumers are preserved. Installed and lock-only npm ls agree across 441 edges, and the existing overrides guard classifies both entries as needed. No global override or unrelated parent upgrade is used. Scanner-fixed remains a post-merge observation, not this qualification result.

Prescription checked: `package.json` / `package-lock.json` own the root resolutions; `harness/pack.mjs#buildOrganismManifest` projects both owners' overrides into an independent stage install. Pinned Brain overrides (`gray-matter/js-yaml`) do not conflict with these proposed keys. No runtime profile, live FM installation or peer seat is changed. Clean-install, pack-contract, native/image and emitted-browser checks must precede the PR; the scanner disposition remains post-merge.

The sections below retain the filing-time analysis and artifact receipt for provenance. References there to then-current npm versions and unpatched upstream advisories are historical; the dated table above governs their current disposition.

## The Problem

New-version PRs are useful delivery vehicles, but there is no current PR covering this cohort and no universal available upgrade. GitHub's [security-update contract](https://docs.github.com/en/code-security/concepts/supply-chain-security/dependabot-security-updates) requires an available resolution: a newer parent can sometimes remove or update a vulnerable child, but a PR is not guaranteed.

| Alert(s) | Package / locked copies | Advisory and fix status at filing |
| --- | --- | --- |
| [78](https://github.com/neomjs/neo-agent-institution/security/dependabot/78) | root `sharp 0.35.4`, runtime | [GHSA-wq5f-xc86-pv6w](https://github.com/lovell/sharp/security/advisories/GHSA-wq5f-xc86-pv6w); fixed at `0.35.5` |
| [75](https://github.com/neomjs/neo-agent-institution/security/dependabot/75), [67](https://github.com/neomjs/neo-agent-institution/security/dependabot/67) | root `dompurify 3.4.15`, development | [GHSA-6688-9rhm-gjv2](https://github.com/cure53/DOMPurify/security/advisories/GHSA-6688-9rhm-gjv2) and [GHSA-p98j-92pf-mc4p](https://github.com/cure53/DOMPurify/security/advisories/GHSA-p98j-92pf-mc4p); both fixed at `3.4.16` |
| [74](https://github.com/neomjs/neo-agent-institution/security/dependabot/74) | root `katex 0.16.47`, development | [GHSA-238p-pmpm-9mq7](https://github.com/KaTeX/KaTeX/security/advisories/GHSA-238p-pmpm-9mq7); fixed at `0.18.2` |
| [69](https://github.com/neomjs/neo-agent-institution/security/dependabot/69) | root `braces 3.0.3`, runtime and development ancestry | [GHSA-vfj7-8cjw-p6xm](https://github.com/advisories/GHSA-vfj7-8cjw-p6xm); no patched release listed |
| [76](https://github.com/neomjs/neo-agent-institution/security/dependabot/76), [70](https://github.com/neomjs/neo-agent-institution/security/dependabot/70) | root `sprintf-js 1.0.3` and nested `1.1.3`; harness `1.1.3` | [GHSA-hp3w-g68c-fv3c](https://github.com/advisories/GHSA-hp3w-g68c-fv3c); no patched release listed |

At filing, npm metadata agreed: latest `sharp 0.35.5`, `braces 3.0.3`, `sprintf-js 1.1.3`; latest Mermaid `12.1.0` still requires KaTeX `^0.16.47`, while latest Monaco `0.57.0` pins DOMPurify exactly `3.4.15`. This is dependency-presence evidence, not proof of application exploitability.

## The Architectural Reality

Exact root/harness lock traversal establishes the introducing paths:

- Sharp: Brain → `@chroma-core/default-embed 0.1.9` → `@huggingface/transformers 3.8.1`. Institution's existing `sharp ^0.35.4` override admits the new fix.
- DOMPurify: Mermaid's `^3.4.12` admits the fix; Monaco's exact requirement does not. Check every nested resolution, not only the hoisted package.
- KaTeX: Mermaid's range excludes the patched release. A compatible parent update or a separately qualified narrow override is needed.
- Braces: Brain → `micromatch 4.0.8`, plus development paths through JSDoc/fast-glob and webpack-dev-server/http-proxy-middleware.
- sprintf-js: Brain/default-embed/Transformers → onnxruntime-node → global-agent/roarr, and gray-matter → js-yaml/argparse; harness ancestry is electron-builder/app-builder-lib → @electron/get → global-agent/roarr.

The two locks are distinct from the shipped organism. [`harness/pack.mjs`](https://github.com/neomjs/neo-agent-institution/blob/af1b85d62228a160e1c1185d0a36fbf6e6c82d29/harness/pack.mjs#L333) derives a manifest from staged imports and supplemental dependencies, merges Brain/Product overrides with conflict refusal, then installs without the root lock. [`afterPack.cjs`](https://github.com/neomjs/neo-agent-institution/blob/af1b85d62228a160e1c1185d0a36fbf6e6c82d29/harness/afterPack.cjs#L24) copies the staged node_modules; the rebuild needs its own dependency receipt.

The locked Engine's [Monaco builder](https://github.com/neomjs/neo/blob/d75cc68543d2bc19d9c37760d5de1ed87cc426e8/buildScripts/build/monaco.mjs#L29) already replaces the vendored sanitizer with npm-resolved DOMPurify and rejects vendor inclusion. Preserve that repair and verify new emitted assets; do not re-file the historical vendor-replacement defect or infer non-execution from a development label.

## The Fix

1. Repair patchable resolutions through compatible parent updates or narrowly qualified root overrides; remove stale nested vulnerable entries and prove clean installation. Do not select an override merely because an advisory supplies a version.
2. For braces and sprintf-js, trace actual input reachability and current parent alternatives. Remove the affected edge through a qualified parent change where possible. Otherwise keep the unresolved row explicit with its introducing owner, containment evidence, upstream advisory, and an activation condition such as a verified safe parent/patch release. Do not invent a fixed version or dismiss the alert.
3. Verify root, harness tooling, generated stage and emitted browser libraries separately. Reuse the existing pack/Brain-contract tests and isolated packaged smoke; no live FM install or seat interruption is needed for this work.
4. Check actual Dependabot job/alert coverage. Current version-update configuration declares daily npm updates only at `/`, with a three-day default cooldown. Harness security alerts still exist independently; absence from that version-update entry does not prove security updates disabled. Admin-only settings probes returned 404 for this non-admin seat, so enablement is unknown, not disabled. An authorized read should resolve that uncertainty; no settings mutation is prescribed here.

## Contract Ledger

| Surface | Authority | Behavior / fallback | Evidence |
| --- | --- | --- | --- |
| Root manifest, overrides and lock | Existing dependency declarations; current advisories | Every patchable copy meets its qualified floor; unavailable fixes remain explicit | Clean install, full resolution-path census and relevant import/build controls |
| Harness manifest and lock | Electron tooling declarations and alert 70 | A tooling upgrade must actually remove or repair its sprintf-js edge | Separate clean harness install and packaging checks |
| Generated organism and browser assets | `buildOrganismManifest`, pack install, afterPack, Engine builders | Root fixes survive fresh stage resolution and rebuilt emitted bytes | Pack-contract tests, actual stage/payload inventory and isolated smoke |
| Seven alert dispositions | GitHub advisory + manifest-specific alert records | Fixed means patched/removed and scanner-confirmed; unresolved remains open | Exact-head evidence, then post-merge default-branch alert read |

Existing manifest/pack JSDoc remains the documentation owner; update it only if a qualified resolution changes a documented behavior.

## Acceptance Criteria

- [ ] All seven alert IDs map to a current dependency path, exposure classification and owned disposition; duplicate advisory records in different manifests are both accounted for.
- [ ] Sharp is at least 0.35.5 and all DOMPurify copies are at least 3.4.17; preserve the scanner-fixed KaTeX resolution. No vulnerable nested copy is hidden by a clean hoisted version.
- [ ] Record the authoritative withdrawal of the braces and sprintf-js advisories and their manifest-specific alerts; retain dependency-presence evidence without claiming a patch or general safety. An active required repair still blocks completion.
- [ ] Harness alert 105's affected minimatch 9/10 edges resolve to the qualified 11.0.1 under the two exact parents; legacy callable 3/5 users remain unchanged. Preserve the actual-consumer compatibility receipt and retire the overrides when a qualified parent release makes them unnecessary.
- [ ] A clean root and harness installation, relevant builds, existing pack/Brain-contract checks and isolated packaged smoke validate the resulting cohort. The rebuilt stage/payload and browser output have their own receipt.
- [ ] Actual Dependabot update/error coverage is recorded without guessing from an absent PR or inaccessible admin settings.
- [ ] **Post-merge:** re-read each manifest-specific alert on `dev`; claim fixed only where GitHub records the repaired/removal result. No blanket dismissal or local-audit-as-installed-proof.

## Decision Record impact

`none`: dependency and artifact verification preserves existing runtime, packaging and configuration ownership. No new production module, architectural layer or agent rule is required.

## Out of Scope

Reopening completed `#124` / `#149`, blanket `npm audit fix --force`, unrelated major upgrades, unqualified overrides or patched forks, advisory severity changes, credential rotation, alert dismissal, repository-setting changes, live FM replacement and installed journey acceptance.

## Current candidate inventory — source alerts remain separate

Read-only inspection on 2026-10-10 matched the refreshed [candidate receipt](https://github.com/neomjs/neo-agent-institution/issues/12#issuecomment-6098915973): Institution `af1b85d6`, Brain `93079328`, Engine `aab57f9f`, Electron `43.5.0`. Stage and unpacked app have the same installed npm-lock SHA-256 `d690a9795e24113a7a7b8423e95041f64a8f27d5d4e7f45ffffda62cbfc61de1`; each listed package's own manifest agrees with that lock.

| Package | Stage and unpacked app |
| --- | --- |
| sharp | `0.35.5` — meets this advisory's package floor |
| braces | `3.0.3` — affected package remains |
| sprintf-js | `1.0.3` below gray-matter and hoisted `1.1.3` — both affected |
| DOMPurify / KaTeX | No entry in this staged Node dependency lock. This candidate's `dist` contains 1,026 files and no `.js`, `.mjs`, `.cjs` or `.wasm` library output; its shell archive contains 24 entries and no such vendor library. |

The immutable app's `app.asar` SHA-256 is `3251df47da8075e6917122cd6f628604eda3374a5ebf97f11d9d8ab450d0379a`; its extracted `contentPolicy.mjs` hash is `c627777cc188ad40412dcadae3f622bc237f493512966d592cf7404fe666b2cf`. Running that actual packaged resolver admitted the existing Engine `Main.mjs` and `Global.css` positive controls, and refused the DOMPurify, KaTeX, Monaco and Mermaid library URL controls with `not-allowlisted`. The renderer ships the source graph plus CSS; the four probes establish this candidate's asset boundary, not every possible source execution path.

This is artifact inventory and policy evidence, not exploitability, installed-adoption or whole-candidate safety. The source root/harness locks still need repair, and a new candidate or checkout-built browser output needs a new receipt. Default-branch alerts remain open; none was dismissed.

## Related and ownership

#124 · #149 · #7 (existing packaging outcome) · neomjs/neo-agent-brain#300 (historical introducing-dependency repair). These links are provenance, not new blockers or authority to change Brain internals.

Owner: `neo-gpt`. Capture is operator-requested; existing FM consumer and Skills security ownership remain intact.

Creation freshness: latest 20 open Institution issues, all-state package/advisory searches, open PRs, own assignments `#596` / `#477`, and latest 30 all-read-state A2A messages were checked immediately before filing. No equivalent current repair found. The newly opened `#656` is Start-fleet waves, not dependency work. MC prior-art sweep recovered the qualified `#149` / `#124` repair; its live post-merge receipt confirms that earlier cohort was fixed.

Origin Session ID: 0a0bd542-0f17-4244-a59e-cec534621a5c
Retrieval Hint: "Institution seven Dependabot alerts sharp DOMPurify KaTeX braces sprintf-js root harness stage no patched release"


## Timeline

- 2026-10-10T14:05:19Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-10T14:05:20Z @neo-gpt-euclid added the `bug` label
- 2026-10-10T14:05:20Z @neo-gpt-euclid added the `ai` label
- 2026-10-10T14:05:20Z @neo-gpt-euclid added the `build` label
- 2026-10-10T14:05:20Z @neo-gpt-euclid added the `dependencies` label
- 2026-10-10T14:05:20Z @neo-gpt-euclid added the `security` label
- 2026-10-10T14:07:27Z @neo-fable-clio cross-referenced by #658
- 2026-10-10T18:03:06Z @neo-gpt-euclid cross-referenced by #666
- 2026-10-11T01:56:03Z @neo-gpt-sophie cross-referenced by #676
- 2026-10-11T01:57:20Z @neo-gpt-sophie cross-referenced by PR #675
- 2026-10-14T10:45:10Z @neo-gpt-euclid cross-referenced by #690
- 2026-11-03T05:24:08Z @neo-gpt-euclid cross-referenced by #862
- 2026-11-03T17:24:27Z @neo-gpt-euclid cross-referenced by #869
- 2026-11-11T01:54:58Z @neo-opus-grace cross-referenced by #912
- 2026-11-12T06:20:13Z @neo-gpt-euclid cross-referenced by PR #935
- 2026-11-12T12:56:50Z @tobiu referenced in commit `2ca2793` - "fix(deps): patch image, sanitizer and glob dependencies (#657) (#935)

* fix(deps): patch sanitizer and image dependency floors (#657)

* fix(deps): bound harness glob fixes to qualified parents (#657)"
- 2026-11-12T12:56:51Z @tobiu closed this issue

