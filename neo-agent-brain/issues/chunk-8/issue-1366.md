---
id: 1366
title: lint-guides lints a caller-supplied repository root
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-29T23:42:47Z'
updatedAt: '2026-10-29T23:42:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1366'
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
blocking:
  - '[ ] 20177 neo''s CI runs the guide-quality lint over its guides again'
---
# lint-guides lints a caller-supplied repository root

## Context

`ai/scripts/lint/lint-guides.mjs` is the guide-quality gate (neomjs/neo#14354). It covers Mermaid reserved words and self-loops, dead local links, dead `ai:*` script refs, tool-table refs to OpenAPI operationIds, and, under the operator rule recorded in neomjs/neo#17574, ticket ids in `learn/guides`. The split's cut manifest gives the script the disposition `edge`, so the Brain holds it. Its subjects stayed in neo: `learn/benefits`, nested `learn/guides` and the Engine half of `learn/agentos`.

Since neomjs/neo#17806 (08-27) no repository runs it: neo dropped `ai:lint-guides`, and the Brain has no script or workflow for it (defect-note `a03b68c6`). Measured 2026-10-29 against neo `dev`, the lint finds:
- in the 10 top-level guides: 0 hard, 5 warnings;
- in the 63 nested `learn/guides`: 3 ticket-id findings in 2 files.

## The Problem

The lint resolves everything from its own root (`ROOT_DIR`, `lint-guides.mjs:49`). A neo CI job that checks out the Brain therefore lints the Brain's tree, not neo's. Explicit file arguments help only partly: they skip the guide-series ticket-id scope, and they read script keys from the Brain alone.

## The Fix

A `--root <dir>` option. When it's given:
- guide discovery (`discoverGuides`, `discoverGuideSeries`) and file resolution use that root;
- script keys are the union of the Brain's and the root's `package.json` scripts, since post-split `ai:*` scripts are the Brain's;
- operationIds still come from the Brain's own `ai/mcp/server/*/openapi.yaml`.

Without it, behaviour is unchanged. The neo-side workflow that calls it is a separate neo ticket.

## Acceptance Criteria

- [ ] AC-1: `--root <dir>` lints that root's `learn/agentos`, `learn/benefits` and nested `learn/guides` (unit arm against a fixture tree).
- [ ] AC-2: a guide's `ai:*` ref resolves when the script is in either the root's or the Brain's `package.json`.
- [ ] AC-3: without `--root`, discovery and results are unchanged (the existing arms pass).
- [ ] AC-4: Brain Unit on the PR reports 0 introduced reds.

## Out of Scope

- The neo workflow and the 3 findings in neo's guides (neo ticket).

## Related

Custody: the neomjs/neo#17806 cut manifest (`edge`). Gap: defect-note `a03b68c6`. Prior: #1360 (the `discoverGuideSeries` root parameter), neomjs/neo#14354, neomjs/neo#17574.

Live latest-open + closed sweep: Brain and neo searches for lint-guides wiring at 2026-10-29T23:42:30Z found the prior art above and no equivalent. A2A: no claim. Structure map: N/A, since this changes one script and its spec.

Retrieval Hint: "lint-guides --root consumer root neo guides CI wiring edge custody script keys union"

## Timeline

- 2026-10-29T23:42:47Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-29T23:42:49Z @neo-opus-ada added the `enhancement` label
- 2026-10-29T23:42:49Z @neo-opus-ada added the `ai` label
- 2026-10-29T23:45:40Z @neo-opus-ada cross-referenced by PR #1367
- 2026-10-29T23:46:46Z @neo-opus-ada cross-referenced by #20177
- 2026-10-29T23:47:16Z @neo-opus-ada marked this issue as blocking #20177

