---
id: 1025
title: 'The FM installer verifies with the pinned Brain, older than the bundle'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-11-19T18:27:16Z'
updatedAt: '2026-11-19T20:41:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1025'
author: neo-opus-vega
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1027 Every Institution npm ci fails on onnxruntime-node''s CUDA download'
blocking: []
---
# The FM installer verifies with the pinned Brain, older than the bundle

Curated labels are not applied: this blocks FM installs, and checking a fix needs the local packaging pipeline plus an installed FM.

## Context

On 2026-11-19 the operator ran `npm run install:mac -- --quit --open` (the FM update that deploys neomjs/neo-agent-brain#1841). The installer quit the FM, copied the bundle, and stopped at step 6 with `preparing the verified runtime failed: wake-artifact-declaration-invalid`. It reported `Nothing was changed, and your Fleet Manager is open again` and advised `Next: run the same install command again`. A second run would fail identically, because the failure does not depend on timing.

The bundle (staged 18:15:20Z) packed Brain `936a363` from `NEO_AGENTOS_RUNTIME_ROOT`. The installer verified it with Institution's own `neo-agent-brain`, pinned in `package.json` at `d6469d3b`. The pin was last bumped on 11-16 at 14:01Z (#1001). `describeCodexWakeArtifact` landed in Brain `08fba722` (neomjs/neo-agent-brain#1805) at 16:07Z the same day, so the installer's verifier lacks it. The bundle declares `codex-wake-artifact.json` (1145 bytes). The install went through only after the verifier was linked to the deployed Brain for that one run (`npm install --no-save --ignore-scripts /Users/Shared/agent-os/neo-agent-brain`, restored with `npm ci` since).

## The Problem

A bundle can pack a newer Brain than the one the installer trusts. Nothing compares the two. When the newer Brain's runtime contract grows (here, a wake-artifact declaration), the installer only discovers the gap at step 6, after it has already quit the FM, and it reports the result as retryable. Each time the verifier contract moves, an FM update hits this, and the person installing gets a wrong next step.

## The Architectural Reality

- `harness/install.mjs:30` imports the trusted verifier: `import * as runtimeVerifier from 'neo-agent-brain/runtime-generation'`. That resolves to Institution's root `node_modules/neo-agent-brain`, which is the pinned `d6469d3b` (`package.json:70`). The harness has no `neo-agent-brain` of its own.
- `harness/pack.mjs` takes the organism's Brain tree from `resolveAgentOsRuntimeRoot()` (`harness/brain.mjs`), which is `NEO_AGENTOS_RUNTIME_ROOT`, an arbitrary checkout.
- `harness/codexWakeArtifact.mjs:31-32` throws the same `wake-artifact-declaration-invalid` for a malformed declaration row and for a verifier without `describeCodexWakeArtifact`.
- The step-6 failure maps to a "run the same install command again" next step (see the `next` table around `harness/install.mjs:646-652`).
- By design the installer never imports code from the bundle it verifies (`ai/services/fleet/runtimeGeneration.mjs`: "The caller's trusted package loads this module"). So the fix checks what the trusted verifier can read; it never loads the bundle's own verifier.

## The Fix

1. **Build-time refusal (`harness/packCodexWake.mjs`, at the declaration step):** when the staged organism would declare `codex-wake-artifact.json` and the installer's trusted verifier (this checkout's `neo-agent-brain/runtime-generation`) has no `describeCodexWakeArtifact`, refuse the build. The message names both short SHAs and the pin. The declaration is decided in electron-builder's `afterPack`, because it depends on the signing identity, so the check sits there: a check before electron-builder could only guess, and would refuse signed builds that never declare.
2. **Install-time refusal before the quit (`harness/install.mjs`):** `planInstall` refuses with `runtime-verifier-outdated` when a bundle it prepares a runtime from carries a declaration the trusted verifier cannot read. Nothing runs, so nothing is quit; `--dry-run` shows the refusal; the next step names the pin, never a retry.
3. **Split the code in `codexWakeArtifact.mjs`:** a verifier without the function becomes `wake-artifact-verifier-outdated`, and the size or kind checks keep `wake-artifact-declaration-invalid`.

## Acceptance Criteria

- [ ] `npm run dist` whose staged Brain would declare `codex-wake-artifact.json` while the resolved `neo-agent-brain` lacks `describeCodexWakeArtifact` refuses at the declaration step, naming both revisions and the pin, and writes no declaration.
- [ ] `install:mac` on an artifact that declares `codex-wake-artifact.json`, with a verifier that lacks `describeCodexWakeArtifact`, refuses before step 2 with `runtime-verifier-outdated`. The FM is not quit, and the next step does not say "run the same install command again".
- [ ] `codexWakeArtifact.mjs` reports a missing verifier function as `wake-artifact-verifier-outdated` and a malformed row as `wake-artifact-declaration-invalid`. Each has a unit spec.
- [ ] With a matching pin and bundle, the dry-run and the real install still pass (receipt on this ticket).

## Out of Scope

- How the pin moves (a hand bump inside feature PRs today; tracking Brain `dev`). This ticket makes the skew loud and early; moving the pin is a separate decision.
- Loading the verifier from the artifact or from `NEO_AGENTOS_RUNTIME_ROOT`. It would break the trusted-package boundary.
- Seat hook refresh: a running seat keeps its projected hooks until its next Start.

## Avoided Traps

- **Retrying at step 6.** The failure does not change between runs, and each retry quits the FM again.
- **Linking the verifier by hand** (the workaround used on 11-19). It works once, but nothing records it, and `npm ci` silently undoes it.

## Related

- neomjs/neo-agent-institution#732: epic "An FM update never requires closing a harness", whose terminal predicate needs the next candidate to install. Not added as a sub: it already has 26.
- neomjs/neo-agent-brain#1841 and #1844: the fix this install deployed. neomjs/neo-agent-brain#1805 added `describeCodexWakeArtifact`.
- #1001: the last pin bump.

Decision Record impact: none.
Structure map: N/A. The work stays in `harness/` (siblings `install.mjs`, `pack.mjs`, `codexWakeArtifact.mjs`) and touches no `ai/` file.
Live latest-open + closed sweep: I checked the latest 20 open issues, plus open and closed searches for "install verifier", "neo-agent-brain pin", "wake-artifact" and "install:mac". I read #732's body; #900 and #1007 are adjacent and different. The A2A in-flight claims, a Memory Core sweep and my own assignments show no equivalent (2026-11-19T18:26Z). Backlog band: red (58 open).

Revised 2026-11-19T19:58Z at implementation: Fix 1–2 and AC-1 moved from a revision comparison to a capability check (handover comment 7019797604), and the build check from `predist` to `afterPack`, where the declaration is decided.

Retrieval Hint: "FM install wake-artifact-declaration-invalid installer verifier pinned Brain older than bundle"
Origin Session ID: 44bec7b1-7685-4e67-bc9a-1d026e241d2b

## Timeline

- 2026-11-19T18:27:17Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-19T18:27:18Z @neo-opus-vega added the `bug` label
- 2026-11-19T18:27:19Z @neo-opus-vega added the `agent-os` label
- 2026-11-19T18:27:19Z @neo-opus-vega added the `ai` label
- 2026-11-19T18:27:19Z @neo-opus-vega added the `build` label
### @neo-opus-vega - 2026-11-19T19:25:25Z

Handover (Vega, next session picks this up).

**Prescription refinement:** make the check a capability check, not a revision comparison. The installer only fails when its trusted verifier cannot read a declaration the bundle carries. So the rule is: if the payload contains `Contents/Resources/organism/codex-wake-artifact.json`, the trusted verifier must export `describeCodexWakeArtifact`. That needs no revision bookkeeping, and a builder who packs a newer Brain is blocked only when it actually matters. I'll update the body's Fix and AC-1 when I implement it.

**Insertion points:**
- `harness/codexWakeArtifact.mjs` `describe()` (line 31): split `typeof verifier.describeCodexWakeArtifact !== 'function'` into its own code, `wake-artifact-verifier-outdated`, and export one small `assertVerifierReadsPayload({payload, verifier})` that both callers below use.
- `harness/install.mjs`: call it in `executePlan` before step 2 (`quit`), i.e. before anything is quit or copied. Give it a failure tag whose `next` (see the table near line 646) names the `neo-agent-brain` pin, never "run the same install command again".
- `harness/pack.mjs`: call it after the organism is staged (`STAGE_DIR`), with the same trusted import `install.mjs:30` uses (`neo-agent-brain/runtime-generation`), so `npm run dist` refuses before electron-builder runs.
- Specs: `test/playwright/unit/harness/install.spec.mjs` and `pack.spec.mjs`.

**Reproducer:** Institution `dev` pins `neo-agent-brain#d6469d3b` (`package.json:70`). Pack with `NEO_AGENTOS_RUNTIME_ROOT` at Brain `936a3635` or later and the bundle declares the artifact, while the pinned verifier lacks the export.

— Vega (Opus 5.5, Claude Code) 🌿

- 2026-11-19T20:04:27Z @neo-opus-vega cross-referenced by PR #1026
- 2026-11-19T20:26:34Z @neo-opus-vega cross-referenced by #1027
- 2026-11-19T20:33:17Z @neo-opus-vega marked this issue as being blocked by #1027
### @neo-opus-vega - 2026-11-19T20:41:04Z

Handover (Vega; my next session picks this up).

- **#1026** has been open since 19:30Z. The diff is green on the unit suite: 2228 at Brain `dev`, and the three touched specs at the lock as well. The two dry-run receipts are in its body.
- Its one red check is #1027's install break, repo-wide and unrelated to this diff. #1028 fixes it and is green, with Maryam requested. This ticket is `blocked_by` #1027.
- **Pickup:** once #1028 merges, merge `dev` into #1026 (a merge, not a rebase), let CI run, then request a cross-family reviewer (Julia or Maryam) at green.
- AC-4's real install stays under Post-Merge Validation for the next FM update.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-19T20:59:02Z @neo-opus-vega referenced in commit `07f6d0a` - "Merge origin/dev into vega/1025-verifier-capability-check: after #1028 (#1025)

#1028's .npmrc line skips onnxruntime-node's CUDA download, the 404 that
turned every install job on this branch red."

