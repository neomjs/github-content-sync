---
id: 1119
title: scriptPlaneClosure.spec's ratchet arms pin a stale edge population and a root that no longer needs a shell
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T06:55:06Z'
updatedAt: '2026-10-18T14:43:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1119'
author: neo-opus-ada
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
closedAt: '2026-10-18T14:43:04Z'
---
# scriptPlaneClosure.spec's ratchet arms pin a stale edge population and a root that no longer needs a shell

## Context

This is a #201 cluster: three reds in `test/playwright/unit/ai/scripts/lint/scriptPlaneClosure.spec.mjs`, all in `the unresolved ratchet holds IDENTITIES, not a count`.

## The Problem

- **`an edge already in the ledger passes`** and **`a SUBSTITUTED edge fails even though the count is unchanged`.** The live closure of `backup.mjs` carries four unresolved edges, but the spec's `KNOWN_BACKUP_EDGES` pins three. The missing one is `ai/services/knowledge-base/source/tenantExtractorLoader.mjs::dynamic-import::importModule`. `IngestionService` imports the extractor loader statically, beside the parser loader, so the extractor loader reaches `backup.mjs` by the route the pin's docblock records for the parser loader. The production `UNRESOLVED_EDGE_LEDGER` already lists it. Only the spec's pin is stale.
- **`an unlisted authority conflict fails; a ticketed one is held and still printed`.** The arm declares the real `syncGithubWorkflow.mjs` as `container-plane`, on the premise that the script requires a shell. On dev its closure reaches `execGit` and `getAuthToken` only deferred (`required: []`), so the declaration conflicts with nothing and the lint exits 0. #1101 (PR #1106) removes the remaining full-sync path, so the premise will not come back.

## The Architectural Reality

- The production ledger is deduplicated across the whole population, so it cannot see `backup.mjs` widen onto an edge that another root already reaches. The spec's per-root pin is the only detector of that widening, so the fix re-pins it rather than deriving it.
- `runLint` takes a `projectRoot`, and the closure walks the disk. Probed on dev `623a0109`, an on-disk fixture root whose module scope calls `execFileSync('git', …)` behaves as the arm needs:
  - with no authority declared, it derives `host-edge`;
  - declared `container-plane`, it fails with one `authority-conflict-in-plane`;
  - with the conflict ticketed, it is held at exit 0 and still printed.

## The Fix

1. Add the extractor-loader identity to `KNOWN_BACKUP_EDGES`, and update the docblock: it counts four edges and names the extractor loader as the parser loader's sibling.
2. Run the authority-conflict arm on a temporary fixture root that requires a shell at module scope, with a no-authority control asserting that the root derives `host-edge`.

## Acceptance Criteria

- [ ] **AC-1:** `scriptPlaneClosure.spec.mjs` passes at head. On dev, its three ratchet arms fail.
- [ ] **AC-2:** the authority-conflict arm reads no real script, and its control fails if the fixture root derives anything other than `host-edge` without authority.

## Out of Scope

- #1106's docblock edits in the same spec. Its hunks do not overlap these.
- The other #201 clusters.

## Related

- #201: the burn-down.
- #1101, #1106: retire the full sync.
- #1108, #1110, #1112, #1117: sibling slices.

Live latest-open sweep: the latest 20 open Brain issues, checked at 2026-10-18T06:54:23Z; no equivalent found. Open-PR sweep: only #1106 touches the spec, in docblock lines.
A2A claim sweep: the latest 20 inbox messages; no claim. Grace's defect note (04:35) names the authority-conflict premise.
Own-assignment sweep: #201, #1117.
Structure map: N/A; this changes one spec and adds no `.mjs`.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "scriptPlaneClosure ratchet KNOWN_BACKUP_EDGES tenantExtractorLoader stale pin; authority-conflict arm fixture root requires shell"

## Timeline

- 2026-10-18T06:55:06Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T06:55:07Z @neo-opus-ada added the `bug` label
- 2026-10-18T06:55:07Z @neo-opus-ada added the `ai` label
- 2026-10-18T06:55:07Z @neo-opus-ada added the `testing` label
- 2026-10-18T06:58:07Z @neo-opus-ada cross-referenced by PR #1120
- 2026-10-18T07:00:34Z @neo-opus-ada cross-referenced by #1121
- 2026-10-18T07:03:12Z @neo-opus-ada cross-referenced by PR #1122
- 2026-10-18T07:14:13Z @neo-opus-ada cross-referenced by #1123
- 2026-10-18T07:20:59Z @neo-opus-ada cross-referenced by #1125
- 2026-10-18T07:24:28Z @neo-opus-ada cross-referenced by #201
- 2026-10-18T07:27:25Z @neo-opus-ada cross-referenced by #1127
- 2026-10-18T07:33:14Z @neo-opus-ada cross-referenced by #1129
- 2026-10-18T07:58:38Z @neo-opus-ada cross-referenced by #1131
- 2026-10-18T14:43:04Z @tobiu referenced in commit `8a1a97c` - "fix(test): the script-plane ratchet arms pin backup's four edges and a fixture shell root (#1119) (#1120)

KNOWN_BACKUP_EDGES gains the tenant-extractor loader that IngestionService imports beside the parser loader. The authority-conflict arm stops borrowing syncGithubWorkflow.mjs, which no longer requires a shell, and declares a fixture root that does, with a control on its derived plane."
- 2026-10-18T14:43:04Z @tobiu closed this issue

