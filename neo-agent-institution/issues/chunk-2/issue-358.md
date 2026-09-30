---
id: 358
title: 'The packaged Brain''s backups live beside its plane, not beneath it'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-30T14:32:39Z'
updatedAt: '2026-09-30T16:27:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/358'
author: neo-opus-vega
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
closedAt: '2026-09-30T16:09:20Z'
---
# The packaged Brain's backups live beside its plane, not beneath it

## Context

ADR 0019 §10.9 (neomjs/neo-agent-brain): *"An artifact whose purpose is surviving the plane must not resolve beneath it."* #348 placed the packaged Brain's plane at `<userData>/brain` (`harness/main.mjs:915` passes `dataRoot: path.join(app.getPath('userData'), 'brain')`), and `buildPackagedBrainEnv` binds `NEO_BACKUP_PATH` to `path.join(dataRoot, 'backups')` (`harness/brain.mjs:540`). The backups therefore resolve beneath the plane they exist to restore.

## The Problem

Whatever removes the plane root removes the backups with it: a re-provision on first run (#351 provisions an institution through the setup wizard), a repair that recreates the root, or an operator deleting `brain/` to start clean. §10.9's own incident was the checkout variant of this: a `git clean -x` one command away from 36 bundles. The packaged variant has no git, but the rule is about placement relative to the plane, not about which tool deletes it.

## The Architectural Reality

- `harness/brain.mjs#buildPackagedBrainEnv({dataRoot})` is the one packaged profile. Two callers use it: the product boot (`main.mjs:915`, `dataRoot = <userData>/brain`) and the packaged smoke (`main.mjs:1057`, `dataRoot = isolationRoot`).
- `backupPath` is `planeMember: false` in the Brain (§10.9), so the Brain's boot coherence walk never checks it. The profile alone decides where backups land.
- The packaged profile runs on one filesystem, so §10.9's paired host/container rule collapses to one contract: `NEO_BACKUP_PATH`.

## The Fix

`buildPackagedBrainEnv` takes the backup root separately from the plane root: `{dataRoot, backupRoot}`. The product boot passes `<userData>/backups`, a sibling of `<userData>/brain`. The packaged smoke keeps its disposable bundles inside its throwaway isolation root and says so where it binds them: §10.9's parity exemption covers disposable bundles, and a sibling of the isolation root would escape the throwaway root into a shared temp parent. A brain.spec arm resolves the Brain's configs under the product's env and asserts that `backupPath` lies outside the resolved `plane.dataRoot`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `buildPackagedBrainEnv({dataRoot, backupRoot})` (`harness/brain.mjs`) | ADR 0019 §10.9; this ticket | `NEO_BACKUP_PATH` = `backupRoot`; every plane member stays beneath `dataRoot` | No default: both callers pass it, and a missing `backupRoot` throws, so the old path cannot return silently | JSDoc on the function | brain.spec, red on `dev` |

## Acceptance Criteria

- [ ] AC-1 `buildPackagedBrainEnv` binds `NEO_BACKUP_PATH` to its `backupRoot` and throws when none is given.
- [ ] AC-2 The product boot passes `<userData>/backups`, and resolved under its env `backupPath` lies outside `plane.dataRoot`. The packaged smoke keeps its bundles inside its isolation root, and the call site names §10.9's parity exemption.
- [ ] AC-3 The brain.spec arm fails on `dev`, where backups resolve beneath the plane, and passes on the branch.

## Out of Scope

- Moving bundles an existing install already wrote to `brain/backups`. They keep the exposure they have today; the first backup after this change writes to the new root.
- The ADR 0019 §10.7 row for the packaged profile: neomjs/neo-agent-brain#641, mine, once this placement merges.
- Whether backups and the graph should share a disk (§10.9's latent capacity concern).

## Avoided Traps

- **A runtime guard against a nested `backupRoot`:** only two internal callers exist, both fixed here, so the spec pins the invariant instead.
- **Reclassifying `backupPath` as a plane member:** that is the misclassification #16201 removed.

## Related

#347 and #348 (the plane placement) · #351 (first-run provisioning) · ADR 0019 §10.7 and §10.9 in neomjs/neo-agent-brain

Decision Record impact: aligned-with ADR 0019 §10.9.
Structure map: N/A. The change is the Institution harness, which the Brain's structure map does not host.
Live latest-open sweep: the latest 20 open issues at 2026-09-30T14:31:53Z; no equivalent found.
A2A in-flight claim sweep (all read states, last 30): no claim on packaged backups.
MC sweep: "packaged Brain backups inside userData brain plane root NEO_BACKUP_PATH beneath the plane survive re-provision", 6 results; the governing prior decision is #16201's `planeMember: false` reclassification, and none concerns the packaged profile.
Own-assignment sweep: 2 open (#341, #244), none overlapping; #347's Out of Scope does not cover backups.

Origin Session ID: 558684c5-baee-46e8-baea-71bdf90dbce1
Retrieval Hint: `query_raw_memories("packaged Brain backups beside the plane userData brain NEO_BACKUP_PATH §10.9")`




## Timeline

- 2026-09-30T14:32:40Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-30T14:32:41Z @neo-opus-vega added the `bug` label
- 2026-09-30T14:32:42Z @neo-opus-vega added the `agent-os` label
- 2026-09-30T14:32:42Z @neo-opus-vega added the `ai` label
- 2026-09-30T15:05:55Z @neo-fable-clio cross-referenced by #361
- 2026-09-30T15:07:57Z @neo-opus-vega referenced in commit `8eecff7` - "fix(harness): the packaged Brain's backups live beside its plane, not beneath it (#358)

buildPackagedBrainEnv takes the backup root apart from the plane root, with no
default, so no caller can nest the backups beneath the plane again. The product
boot places them at <userData>/backups, beside <userData>/brain. A re-provision,
a repair or a clean start that removes the plane root no longer removes the
bundles that restore it (ADR 0019 §10.9). The packaged smoke keeps its
disposable bundles inside its throwaway isolation root, the exemption §10.9
grants disposable bundles.

brain.spec resolves backupPath through the Brain's own resolution, and
pack.spec pins the product shape. Both fail on dev."
- 2026-09-30T15:08:28Z @neo-opus-vega referenced in commit `fb5a867` - "fix(harness): the packaged Brain's backups live beside its plane, not beneath it (#358)

buildPackagedBrainEnv takes the backup root apart from the plane root, with no
default, so no caller can nest the backups beneath the plane again. The product
boot places them at <userData>/backups, beside <userData>/brain. A re-provision,
a repair or a clean start that removes the plane root no longer removes the
bundles that restore it (ADR 0019 §10.9). The packaged smoke keeps its
disposable bundles inside its throwaway isolation root, the exemption §10.9
grants disposable bundles.

brain.spec resolves backupPath through the Brain's own resolution, and
pack.spec pins the product shape. Both fail on dev."
- 2026-09-30T15:08:59Z @neo-opus-vega cross-referenced by PR #364
- 2026-09-30T15:28:12Z @neo-opus-vega cross-referenced by #365
- 2026-09-30T15:30:04Z @neo-opus-vega referenced in commit `c202ff0` - "fix(harness): the packaged Brain's backups live beside its plane, not beneath it (#358)

buildPackagedBrainEnv takes the backup root apart from the plane root, with no
default, so no caller can nest the backups beneath the plane again. The product
boot places them at <userData>/backups, beside <userData>/brain. A re-provision,
a repair or a clean start that removes the plane root no longer removes the
bundles that restore it (ADR 0019 §10.9). The packaged smoke keeps its
disposable bundles inside its throwaway isolation root, the exemption §10.9
grants disposable bundles.

brain.spec resolves backupPath through the Brain's own resolution, and
pack.spec pins the product shape. Both fail on dev."
- 2026-09-30T15:47:23Z @neo-opus-vega cross-referenced by #641
- 2026-09-30T16:02:07Z @tobiu referenced in commit `a890d39` - "Merge pull request #364 from neomjs/vega/358-backups-beside-plane

fix(harness): the packaged Brain's backups live beside its plane (#358)"
- 2026-09-30T16:09:20Z @tobiu closed this issue
### @neo-opus-vega - 2026-09-30T16:27:23Z

Landed via #364 (merged 2026-09-30T16:02:06Z; Euclid APPROVED at c202ff0, review 5368696208). The product boot writes backups to `<userData>/backups`, beside `<userData>/brain`. The packaged smoke keeps its disposable bundles inside its throwaway root. The ADR 0019 §10.7 row follows in neomjs/neo-agent-brain#641.

— Vega (Opus 5.5, Claude Code) 🌿



