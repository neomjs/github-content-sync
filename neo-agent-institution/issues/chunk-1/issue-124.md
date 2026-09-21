---
id: 124
title: Remove vulnerable sharp from the inherited embedding dependency
state: CLOSED
labels:
  - bug
  - ai
  - dependencies
  - security
assignees:
  - neo-fable-clio
createdAt: '2026-09-12T11:39:47Z'
updatedAt: '2026-09-18T11:22:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/124'
author: neo-gpt
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
closedAt: '2026-09-18T11:22:46Z'
---
# Remove vulnerable sharp from the inherited embedding dependency

## Context

The 2026-09-12 operator-requested security sweep found two open runtime alerts: [25](https://github.com/neomjs/neo-agent-institution/security/dependabot/25) ([libheif advisory](https://github.com/advisories/GHSA-rgj7-g3m4-5g8c), patched sharp 0.35.4) and [23](https://github.com/neomjs/neo-agent-institution/security/dependabot/23) ([libvips advisory](https://github.com/advisories/GHSA-f88m-g3jw-g9cj), patched 0.35.0).

## The Problem

The root lock resolves sharp 0.34.5 through neo-agent-brain -> @chroma-core/default-embed -> @huggingface/transformers. A patched sharp 0.35.4 is now published, but the Transformers dependency admits only 0.34.x; a normal patch refresh cannot select it.

## The Architectural Reality

Institution pins Brain at bd5417155be455b59983de2a6e5bd8249b113d2b and has its own root lock. Root-only npm overrides in an installed dependency are not inherited by consumers, so a Brain repair must be verified again from Institution's root. Package presence proves affected bytes, not that the text-only consumer invokes vulnerable image codecs.

## The Fix

Coordinate with neomjs/neo-agent-brain#300: establish the actual import/call path, qualify sharp 0.35.4 against the retained Transformers API and platform binaries, then update the root resolution using a compatible parent release or a narrowly scoped, documented override. Retain Engine/Brain pins unless an independently accepted upgrade is required.

## Acceptance Criteria

- [ ] Complete parent chain and image-codec reachability are recorded against the current root install.
- [ ] All runtime sharp resolutions meet both advisory floors; no silent npm-only or platform-specific residual remains.
- [ ] The retained Transformers import and used sharp operations work on the qualified platform; Institution's isolated and Brain-contract CI pass.
- [ ] Recheck both alerts after the human merge; disclose scanner delay without dismissing alerts to simulate repair.

## Out of Scope

Harness js-yaml is already covered by Dependabot PR #122. No unrelated Engine migration, model replacement, or dependency-major sweep.

## Avoided Traps

An override must be qualified because 0.35 is outside the current parent range. Do not infer that Brain's npm override automatically governs this root. Do not claim exploitability from a version match.

## Related

neomjs/neo-agent-brain#300; PR #122 for the separate harness YAML repair.

unowned-rationale: the shared sharp compatibility decision belongs first to the existing Brain #300 lane; this ticket preserves the required consumer verification and root-lock follow-through.

Decision Record impact: none.
Origin Session ID: 92f5d790-2865-4f69-b25e-150175745a6d
Retrieval Hint: sharp 0.34.5 0.35.4 Transformers inherited Brain runtime libheif libvips.

Creation freshness: latest open issues, all-state sharp/YAML searches, own assignments, 30 all-status A2A messages and Memory Core problem queries checked immediately before filing. Brain #300 is the upstream owner; no Institution remediation tracker exists.


## Timeline

- 2026-09-12T11:39:49Z @neo-gpt added the `bug` label
- 2026-09-12T11:39:49Z @neo-gpt added the `ai` label
- 2026-09-12T11:39:49Z @neo-gpt added the `dependencies` label
- 2026-09-12T11:39:49Z @neo-gpt added the `security` label
- 2026-09-12T11:40:33Z @neo-gpt cross-referenced by #300
- 2026-09-18T10:25:56Z @neo-fable-clio cross-referenced by #149
- 2026-09-18T10:26:04Z @neo-fable-clio assigned to @neo-fable-clio
### @neo-fable-clio - 2026-09-18T10:38:16Z

**Intake (2026-09-18), verdict `valid-as-written` — taken, shipped together with #149.**

Created 2026-09-12, pre-stale, no blocker edge; alerts 23 + 25 re-read open today. Currency: `@huggingface/transformers@4.3.0` now asks `sharp ^0.35.4` itself (4.2.0 still asked `^0.34.5` when this was filed), while `@chroma-core/default-embed@0.1.9` (latest) stays on Transformers `^3.5.1` and 3.8.1 is the last 3.x — no compatible parent release exists, so the narrow override is the only root-level move.

`Prescription checked: package.json overrides — owns the concern`, scoped to the one parent whose range excludes the fix: `"@huggingface/transformers": {"sharp": "^0.35.4"}`.

**AC-1, chain + reachability** (root install): `neo-agent-brain` → `@chroma-core/default-embed` 0.1.9 → `@huggingface/transformers` 3.8.1 → `sharp`. chromadb resolves the provider with a dynamic `import("@chroma-core/default-embed")`; Transformers' `src/utils/image.js:17` imports sharp **statically**, so the native library loads whenever the provider does. The codec paths (libvips/libheif decode) run only on image input — `RawImage.read/fromBlob/resize/crop/save`; a text-embedding pipeline never calls them. Bytes load, codecs are not invoked. No exploitability claimed.

**One finding the ticket could not see: the pack stage.** `harness/pack.mjs#stageOrganism` writes a *generated* manifest (`buildOrganismManifest` → `dependencies` only) and runs `npm install` **without a lock**. A root override never reaches it. Measured with the staged manifest's exact shape: `{"dependencies": {"@chroma-core/default-embed": "0.1.9"}}` resolves `sharp 0.34.5`; the same manifest plus the overrides resolves `0.35.4`. So the root lock would read clean, both alerts would close, and the packaged artifact (`SUPPLEMENTAL_DEPENDENCIES.brain` ships the provider) would keep the vulnerable libraries — the "silent residual" of AC-2. The PR makes the staged manifest repeat both owners' `overrides` (one key, one value, or the pack fails — the rule `dependencies` already follow), with a red-first arm.

**AC-3, qualification — differential, through Transformers' own call sites** (`RawImage.fromBlob` → `metadata()` / `rotate().raw().toBuffer()`, `resize`, `crop` → `extract`, `center_crop` → `extract` and `extend`, `save` → `toFile`, plus the `default-embed` ESM import; darwin-arm64, Node 25.9):

| | sharp | libvips | libheif | digest |
|---|---|---|---|---|
| control (Brain checkout) | 0.34.5 | 8.17.3 | 1.20.2 | `{"grayChannels":3,"grayPixel":[77,77,77],"resizedPixel":[200,100,50],"centerLargePixel":[0,0,0],"centerLargeCorner":[0,0,0]}` |
| this branch | 0.35.4 | 8.18.6 | 1.23.2 | identical |

Both packaging models carry `src/binding.gyp` in the same place, so `@electron/rebuild` meets the same module shape. Not run here: a full `npm run dist` (no Electron tooling in this checkout) — named as a post-merge check in the PR.

The Brain's own lock (neomjs/neo-agent-brain#300) stays that lane's; this evidence transfers to it unchanged.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59


- 2026-09-18T10:39:43Z @neo-fable-clio cross-referenced by PR #150
- 2026-09-18T10:40:56Z @neo-fable-clio cross-referenced by #151
- 2026-09-18T11:22:47Z @tobiu closed this issue

