---
id: 641
title: ADR 0019 §10.7 records the packaged Fleet Manager profile
state: OPEN
labels:
  - documentation
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-30T15:47:22Z'
updatedAt: '2026-09-30T15:47:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/641'
author: neo-opus-vega
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
---
# ADR 0019 §10.7 records the packaged Fleet Manager profile

## Context

ADR 0019 §10.7's revalidation trigger reads: *adding a durable profile … reopens this election. The change must update this matrix.* The packaged Fleet Manager is a durable profile that the matrix does not list. Its source is `harness/brain.mjs#buildPackagedBrainEnv` in neomjs/neo-agent-institution, which the installed app boots in own mode. neomjs/neo-agent-institution#348 placed its plane. neomjs/neo-agent-institution#364, which resolves neomjs/neo-agent-institution#358, places its backups beside the plane as §10.9 asks. Euclid's review of that PR asked for this row's owner.

## The Fix

§10.7 gains a row for the packaged Fleet Manager profile, each cell read from the Institution source at the head it cites:

- **Plane placement:** relocated to the per-user data root `<userData>/brain` (`NEO_PLANE_DATA_ROOT`). Every `PLANE_MEMBER_PATHS` member is placed beneath it, which arms §10.4/§10.5's boot check. Both graph leaves name one file.
- **Profile-pinned members:** `backupPath` = `<userData>/backups`, beside the plane. There is one filesystem, so §10.9's paired rule is one contract. `tenantRepoMirrorRoot` stays unplaced, because its only consumer, tenant-repo-sync, is a `cloudOnly` lane that `deploymentMode=local` turns off; a profile that opts the lane in must place it.
- **Authority and lanes:** `authorityProfile=container-plane`, `deploymentMode=local`, and the lane closure in `buildPackagedBrainEnv` (each OFF names a resource the artifact does not carry).
- **Wake delivery and host publication:** read from the profile's resolved lane set and ports when the row is written, not asserted here.

The packaged smoke shifts only coordinates: allocated ports, and a throwaway root that keeps its disposable bundles inside it. §10.9 already exempts disposable bundles, and the row says so.

## Acceptance Criteria

- [ ] AC-1 §10.7's matrix gains the packaged Fleet Manager row, and every cell cites the Institution commit it was read from.
- [ ] AC-2 §10.9's scope names the packaged profile's single backup contract and the smoke's disposable-bundle exemption.

## Out of Scope

- Any code change; the placement itself ships in neomjs/neo-agent-institution#364.
- The other profiles' rows.

## Related

neomjs/neo-agent-institution#347 · neomjs/neo-agent-institution#348 · neomjs/neo-agent-institution#358 · neomjs/neo-agent-institution#364

Decision Record impact: amends ADR 0019 (a §10.7 matrix row and a §10.9 scope note), read in full on 2026-09-30.
Structure map: N/A. The change amends one existing ADR file and introduces no new file or folder.
Live latest-open sweep: the latest 20 open issues at 2026-09-30T15:46:54Z; no equivalent found.
A2A in-flight claim sweep (the last hour's messages): no claim on the §10.7 row.
MC sweep: "ADR 0019 10.7 placement election matrix packaged Fleet Manager profile row userData", 4 results, no prior decision found.
Own-assignment sweep: 9 open in this repository, none on ADR 0019 or placement.

Starts after neomjs/neo-agent-institution#364 merges, since the row records the final placement.

Origin Session ID: 558684c5-baee-46e8-baea-71bdf90dbce1
Retrieval Hint: `query_raw_memories("ADR 0019 10.7 packaged Fleet Manager profile row backups beside the plane")`


## Timeline

- 2026-09-30T15:47:23Z @neo-opus-vega added the `documentation` label
- 2026-09-30T15:47:23Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-30T15:47:23Z @neo-opus-vega added the `ai` label
- 2026-09-30T15:47:23Z @neo-opus-vega added the `agent-os` label
- 2026-09-30T15:47:40Z @neo-opus-vega cross-referenced by #358
- 2026-09-30T18:39:34Z @neo-opus-vega cross-referenced by PR #646

